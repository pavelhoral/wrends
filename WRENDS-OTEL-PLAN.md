# Wren:DS OpenTelemetry Instrumentation — Implementation Plan

Status: proposal · Target: 5.2.0 · Java 17

## Summary

Add one Maven module, `wrends-telemetry`, holding the shared tracing vocabulary and
helpers. Instrumentation lives at the natural operation boundaries in the modules that own
them: a `ConnectionFactory` decorator for the LDAP client, and native call sites on the
operation lifecycle for the directory server.

```
opendj-core  ──────────────┐
    ▲                      │
    │                      ▼
    │              wrends-telemetry ──► io.opentelemetry:opentelemetry-api
    │                      ▲
    │                      │
opendj-server-legacy ──────┘        (instrumentation call sites)
```

`opendj-core` is not modified and does not depend on telemetry. Keeping the client wrapper
in `wrends-telemetry` rather than inside the SDK means the SDK's `japicmp`-frozen public
API is never touched, and the span schema stays free to evolve.

Three decisions the rest of the plan follows from:

1. **OpenTelemetry API only, in production scope.** `io.opentelemetry:opentelemetry-api` is
   the single telemetry compile dependency — no exporter, no autoconfigure, no collector
   config; the operator supplies the runtime via the OTel or Datadog Java agent. Tests use
   `opentelemetry-sdk` / `opentelemetry-sdk-testing` in `test` scope to assert against an
   in-memory exporter.
2. **Library instrumentation, not an agent.** Both sides are code we own and ship, so there
   is no reason to maintain bytecode advice, muzzle ranges, or a vendor fork.
3. **No vendor abstraction layer.** OTel is the abstraction. Zero `datadog.*` imports;
   Datadog compatibility comes from attribute naming plus the Datadog agent's OTel API
   bridge.

## Module: `wrends-telemetry`

Package root `org.wrensecurity.wrends.telemetry`, a clean break from the inherited
`org.forgerock.opendj.*` / `org.opends.server.*` trees.

### Dependencies

```xml
<dependency>
    <groupId>${project.groupId}</groupId>
    <artifactId>opendj-core</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
</dependency>
```

The `opendj-core` dependency is load-bearing: the legacy server's `Operation` exposes core
types — `getResultCode()` returns `org.forgerock.opendj.ldap.ResultCode` and
`getMatchedDN()` returns `org.forgerock.opendj.ldap.DN` (`types/Operation.java:24-25`), as
does `SearchOperation.getBaseDN()`. Result-code classification and DN obfuscation are
therefore genuinely shared code across both sides.

Controls are the exception. The server's `Operation.getRequestControls()` returns
`org.opends.server.types.Control`, an *abstract class* unrelated to the SDK's
`org.forgerock.opendj.ldap.controls.Control` interface, so only a wire codec can be shared
— see [Trace context propagation](#trace-context-propagation).

Pin the version in the root `dependencyManagement` (`pom.xml:146-249`) as a single
`opentelemetry-api` entry rather than importing the full OTel BOM, which is the only OTel
artifact compiled against.

### Package layout

Application code instantiates the client wrapper, which makes it an API contract whether or
not it appears in a BOM. Keep that contract explicit and small; everything else stays free
to change.

```
org.wrensecurity.wrends.telemetry            <- experimental public API
├── TelemetryConnectionFactory
├── TelemetryOptions                          Option<...> constants
└── DirectoryTelemetry                        entry point

org.wrensecurity.wrends.telemetry.internal   <- no compatibility promise
├── LdapOperationTrace
├── LdapAttributes                            AttributeKey constants
├── LdapSpanNames
├── LdapResultCodes                           result code -> status + text
├── obfuscation/{ObfuscationPolicy, Dns, Filters}
└── propagation/{TraceContextCodec, TraceContextRequestControl}
```

Mark the public package experimental in Javadoc, keep `.internal` out of the OSGi
`Export-Package` list, and leave the module out of `opendj-bom` until the schema settles.

## Span model

### What a server span measures

The server span covers the full inbound request: accepted, queued, executed, response
sent. That spans three points in the operation lifecycle, so it takes three call sites.

```
LDAPClientConnection.addOperationInProgress(op)   :1135   <- span starts here
    connectionHandler.getQueueingStrategy().enqueueRequest(op)   :1174
        [ queue wait ]
        operation.run()                                    <- context activated here
            [ processing ]
LDAPClientConnection.sendResponse(op)             :657     <- span ends here
```

Record the queue wait explicitly as `ldap.queue_time`, computed at completion from the span
start and `Operation.getProcessingStartTime()`. Under overload this is the number that
explains an otherwise unattributed gap between the client span and server processing.

### Persistent searches are not traced

A persistent search does not complete when its initial phase does — the final response is
sent only when the search terminates, potentially hours later, and the server warns that
this path "could be multithreaded in the event of a persistent search"
(`SearchOperationBasis:777`). A span held open that long is worse than no span: it is
invisible to the backend until it ends, and it distorts every latency aggregate it lands in.

Detect the persistent-search request control at the start call site and skip tracing the
operation. Streaming-phase observability belongs in metrics or span events, which is a
separate decision.

### Span kinds and names

Span names must be low cardinality. An obfuscated DN is far safer than a raw one but is not
inherently low-cardinality — LDAP trees have arbitrary depth and arbitrary RDN attribute
types (`uid=?,ou=?,dc=?,dc=?`, `service=?,region=?,tenant=?,dc=?,dc=?`, …). The DN shape is
an attribute, not part of the span name.

| Span name | Kind | Where |
|---|---|---|
| `ldap.search`, `ldap.bind`, `ldap.add`, `ldap.modify`, `ldap.modifyDN`, `ldap.delete`, `ldap.compare`, `ldap.extended` | `CLIENT` | one per logical client operation |
| same names | `SERVER` | one per server-side operation |
| `ldap.connect` | `CLIENT` | real TCP connect + TLS + initial bind |
| `ldap.pool.acquire` | `INTERNAL` | pool wait time, never conflated with connect |
| `backend.*` | `INTERNAL` | deferred; explicit call sites in backend code |

Consequence to accept knowingly: OTel span names map to Datadog resource names, so Datadog
resources are coarse (`ldap.search` rather than `search ou=?,dc=?`). Facet on
`ldap.request.dn` instead. Correct cardinality is worth more than a pre-grouped resource.

## Semantic conventions

Settle this table before PR 1. There is no OpenTelemetry semantic convention for LDAP, so
use the stable generic attributes where they apply and an `ldap.*` namespace for the rest.
Client and server must agree on it exactly, or cross-side correlation is worthless.

| Attribute | Example | Side | Notes |
|---|---|---|---|
| `network.protocol.name` | `ldap` | both | stable generic attribute for the app-layer protocol |
| `network.protocol.version` | `3` | both | |
| `network.transport` | `tcp` | client | |
| `server.address` / `server.port` | `ds01.example.com` / `1636` | client | |
| `ldap.operation.name` | `search` | both | follows the `db.operation.name` shape |
| `ldap.request.dn` | `uid=?,ou=?,dc=?,dc=?` | both | values always obfuscated |
| `ldap.request.control_oids` | `["1.2.840.113556.1.4.319"]` | both | string array |
| `ldap.response.status_code` | `32` | both | numeric, follows `http.response.status_code` |
| `ldap.response.status_text` | `noSuchObject` | both | |
| `ldap.response.matched_dn` | `ou=?,dc=?` | both | on failure only |
| `ldap.response.control_oids` | `["2.16.840.1.113730.3.4.4"]` | both | string array |
| `ldap.search.scope` | `wholeSubtree` | both | |
| `ldap.search.filter` | `(&(objectClass=?)(uid=?))` | both | off by default, shape only |
| `ldap.queue_time` | `12` | server | milliseconds between accept and `getProcessingStartTime()` |
| `ldap.search.entries_returned` | `1` | both | |
| `ldap.search.page_size` | `500` | both | from the paged-results control |
| `ldap.response.password_policy.error` | `passwordExpired` | both | high-signal, non-PII |
| `ldap.tls.enabled` / `ldap.tls.starttls` | `true` | client | |
| `error.type` | `invalidCredentials` / `LdapException` | both | see below |
| `ldap.connection_id` / `ldap.message_id` | `114` / `7` | both | opt-in, local diagnostics only |

### Product identity goes in `service.name`, not `db.system.name`

`db.system.name` is not set, for two independent reasons.

**LDAP is not a DBMS product.** The stable OTel database conventions describe a *database
client call* with `SpanKind.CLIENT`, which does not cover the server span at all.

**The attribute names the server product, not the client's vendor.** The definition — "the
DBMS product as identified by the client instrumentation" — qualifies *who* names it
(client-side code, with imperfect knowledge), not *what* is named. The spec's own example
settles it: a PostgreSQL client library connected to CockroachDB sets `postgresql`, not
`cockroachdb`. The value tracks the protocol/product family the client speaks, deliberately
not the actual server vendor.

So a Wren:DS-specific value fails on both sides. On client spans, the SDK connects to
OpenLDAP, Active Directory, 389-DS, or Wren:DS and cannot know which — claiming the product
asserts something the instrumentation has no way to establish. Across sides, if the server
declared one value and the client another, the two spans of a single operation would
disagree on the attribute whose purpose is to group them. That leaves the protocol family
as the only honest value, which the first reason rules out.

Product identity belongs in the OTel *resource* attributes on the server process:

```
service.name    = directory-prod    (the deployment's service name)
service.version = 5.2.0             (the Wren:DS version)
```

These are set once per process by agent or SDK configuration, cost nothing per operation,
and are what every backend including Datadog already keys service identity off. Document
recommended values in the README rather than emitting them from instrumentation code. Note
that `service.name` conventionally names the *deployment*, not the product — a shop running
three directories wants `directory-prod` / `directory-staging`, not three services all
called `wrends`.

If a span-level product attribute is wanted later — to distinguish directory vendors in a
mixed estate from the client side, once the server advertises itself — add
`ldap.server.product` in the `ldap.*` namespace rather than overloading a registry-governed
attribute.

If Datadog classification argues for `db.system.name` after phase 5 verification, add it as
a server-only configurable shim defaulting off, documented as a vendor compatibility
measure, never on client spans.

### `error.type` depends on how the operation failed

Two distinct failure modes, two low-cardinality values:

- **Protocol failure** — the request reached the server and returned a failing result code:
  `error.type` = the result-code name (`invalidCredentials`, `noSuchObject`).
- **Transport or runtime failure** — no result code exists: `error.type` = the exception
  class simple name (`LdapException`, `ConnectException`).

### Error classification

"Non-zero result code is an error" would flood dashboards with false errors. Default the
error set to every result code except `{0 success, 5 compareFalse, 6 compareTrue,
10 referral, 14 saslBindInProgress}`, configurable. `compareFalse` is a successful compare;
`noSuchObject` and `invalidCredentials` are routine on an authentication path. Always set
`ldap.response.status_code` regardless, so teams can build their own monitors.

### Controls

Emit raw OIDs in a string array. That is lossless, needs no lookup table, and survives
controls we have never heard of; dashboards can map OIDs to labels at display time.

Never emit control values. `ProxiedAuthV2RequestControl`'s value *is* an authzid — a full
DN — and the paged-results cookie is opaque server state. Decode only
`ldap.search.page_size` and `ldap.response.password_policy.error`.

### Obfuscation

DN obfuscation is an allowlist, not a scrub — walk the structure and emit attribute types
only, never touching values:

```java
// DN implements Iterable<RDN> (DN.java:695); RDN implements Iterable<AVA> (RDN.java:461)
// AVA.getAttributeName() (AVA.java:542) — AVA.getAttributeValue() is never called
for (RDN rdn : dn) {
    for (AVA ava : rdn) {          // multi-valued RDN -> cn=?+uid=?
        sb.append(ava.getAttributeName()).append("=?");
    }
}
```

`uid=bjensen,ou=people,dc=example,dc=com` → `uid=?,ou=?,dc=?,dc=?`. Because the code cannot
reach a value, it cannot leak one. Modes: `values` (default), `full` (drop attribute types
too), `none` (raw, lab use only).

Enforce hard limits, all configurable with documented defaults: maximum obfuscated DN
length, maximum filter nesting depth, maximum number of control OIDs recorded, and defined
truncation behaviour (truncate with a trailing marker; never drop the attribute silently).
LDAP inputs are attacker-influenced, so unbounded attribute construction is a DoS surface.

Guard every expensive computation so sampled-out spans pay nothing:

```java
if (span.isRecording()) {
    span.setAttribute(LdapAttributes.REQUEST_DN, Dns.obfuscate(dn));
}
```

The filter obfuscator needs two adapters: the client sees
`org.forgerock.opendj.ldap.Filter`, the server sees `org.opends.server.types.SearchFilter`,
and they are unrelated types. Define the shape algorithm once against a small internal
interface.

## Context lifecycle

A `Scope` mutates `Context.current()` on one thread until closed, so it cannot span a
request hook and a response hook that may run on different threads — which the server does
for persistent searches. Attach a span plus its context and hold no long-lived `Scope`:

```java
public final class LdapOperationTrace {
    private final Span span;
    private final Context context;
    private final AtomicBoolean ended = new AtomicBoolean();

    static LdapOperationTrace start(Tracer tracer, String name, Context parent) {
        Span span = tracer.spanBuilder(name)
                          .setParent(parent)
                          .setSpanKind(SpanKind.SERVER)
                          .startSpan();
        return new LdapOperationTrace(span, parent.with(span));
    }

    public Context context() { return context; }   // explicit propagation for children

    public void complete(ResultCode rc) { ... }    // sets status attributes, ends span
    public void fail(Throwable t) { ... }          // records exception, ends span
}
```

- Store the trace on the operation via `Operation.setAttachment(String, Object)`
  (`types/Operation.java:518`).
- Child spans use explicit parenting: `.setParent(trace.context())`. Never rely on
  `Span.current()` crossing a hook boundary.
- `complete` and `fail` are idempotent, guarded by the `ended` flag.

Where automatic log correlation and implicit child parenting are wanted during synchronous
processing, activate the context in a narrow lexical block on the executing thread:

```java
try (Scope scope = trace.context().makeCurrent()) {
    // synchronous processing only
}
```

On the server that block is the `runOperation` helper described below, and it is the only
place a `Scope` is opened. The client path does not use this shape at all: it starts the
span in the `*Async` method and completes it from the promise callback.

## Client instrumentation

`TelemetryConnectionFactory` decorates any `ConnectionFactory`; the application wraps its
outermost factory:

```
TelemetryConnectionFactory          <- one CLIENT span per logical operation
  └── CachedConnectionPool          <- wrap separately for "ldap.pool.acquire"
        └── RequestLoadBalancer
              └── LDAPConnectionFactory   <- wrap separately for "ldap.connect"
                    └── GrizzlyLDAPConnection
```

Because placement is explicit, the decorator chain never double-counts and no span
suppression logic is needed.

### Base class

`TelemetryConnection` extends `AbstractAsynchronousConnection` and holds the delegate as a
field. That base class implements every synchronous method as
`blockingGetOrThrow(xxxAsync(request))` (`AbstractAsynchronousConnection.java:45-83`), so
instrumenting the nine `*Async` methods covers the blocking API too. The cost is
hand-implementing about ten non-operation methods (`close`, `isValid`, listener
registration, …); the convenience overloads on `AbstractConnection` come along correctly.

`AbstractConnectionWrapper` is not usable here: its synchronous methods delegate straight to
the wrapped connection (`AbstractConnectionWrapper.java:82-84`) instead of routing through
the async ones, so instrumenting `*Async` alone would miss every blocking call.

The same applies to `LDAPConnectionFactory.getConnection()`, which is
`getConnectionAsync().getOrThrowUninterruptibly()` (`LDAPConnectionFactory.java:440`).

### Span lifecycle

Start the span in the `*Async` method and complete it from
`LdapPromise#thenOnResultOrException(ResultHandler, ExceptionHandler)`
(`LdapPromise.java:54`). No context store is needed.

That callback fires on a Grizzly worker thread. End the span there and never activate a
`Scope` that will not be closed; capture the caller's `Context` at request time and pass it
as an explicit parent.

Search entry counts come from wrapping the `SearchResultHandler` argument.

Configuration is exposed as `Option<...>` constants in `TelemetryOptions`, consistent with
the SDK's existing style: enable/disable, DN obfuscation mode, filter capture, error
result-code set, propagation, and the obfuscation limits.

## Server instrumentation

Three call sites in `opendj-server-legacy`, one per lifecycle concern, plus a `ServerTelemetry`
helper that keeps OTel types out of the server code.

### 1. Accept — start the span

`LDAPClientConnection.addOperationInProgress(Operation)` (`:1135`) receives the fully
constructed operation, performs the disconnect-in-progress and duplicate-message-ID checks,
then hands it to `enqueueRequest()` (`:1174`). Start the span after those checks and before
the enqueue:

```java
private void addOperationInProgress(Operation operation) throws DirectoryException {
    ...                                    // existing checks
    ServerTelemetry.start(operation);      // attaches Span + Context to the Operation
    try {
        connectionHandler.getQueueingStrategy().enqueueRequest(operation);
    } catch (Throwable t) {
        ServerTelemetry.fail(operation, t);
        throw t;
    }
}
```

This is the common path for all nine traced operation types — abandon, add, bind, compare,
delete, extended, modify, modifyDN, search (`:1638`–`:2161`). Operations rejected by the
preceding checks are not traced, which is the intended behaviour: they never became work.

### 2. Execute — activate the context

`Operation.run()` is invoked from six places: `SynchronousStrategy:39`,
`BoundedWorkQueueStrategy:83,91,247`, `TraditionalWorkerThread:166`, and
`ParallelWorkerThread:168`. Converge them on one helper rather than scattering OTel-specific
code across four files:

```java
static void runOperation(Operation operation) {
    try (Scope ignored = ServerTelemetry.contextOf(operation).makeCurrent()) {
        operation.run();
    }
}
```

Activating the context here — and only here — is what makes spans created inside the
operation become children: Wren's own future `backend.*` spans, agent-instrumented library
calls, and plugin instrumentation alike. Without it, only code that knows about
`Operation.getAttachment()` can parent correctly.

### 3. Respond — end the span

`LDAPClientConnection.sendResponse(Operation)` (`:657`) is the common final-response path.
It already removes the operation from `operationsInProgress` and serializes the outgoing
message, so completing the span there covers response transmission.

### Operations that never reach `sendResponse`

Two cases need explicit handling or the span leaks:

- **Abandon** enters `addOperationInProgress` (`:1638`) but has no response — the code
  comments say so directly. End its span when `run()` returns.
- **Connection teardown** mid-operation, and any path that abandons an operation without
  responding. Guard by ending any still-open span when the operation leaves
  `operationsInProgress`, and by draining the connection's in-flight operations on
  disconnect.

Unbind never enters this path (`processUnbindRequest:2191` constructs its operation
separately) and is not in the span model.

Also out of scope: internal operations via `InternalClientConnection`, which never touch
`LDAPClientConnection` and have no inbound request to represent, and the HTTP/SDK adapter
path (`SdkConnectionAdapter:275`).

### Attributes

Read off `Operation`: `getOperationType()`, `getResultCode()`, `getMessageID()`,
`getConnectionID()`, `getMatchedDN()`, `getProcessingStartTime()`. `SearchOperation` adds
`getBaseDN()`, `getScope()`, `getFilter()`, `getEntriesSent()`. Controls come from
`getRequestControls()` / `getResponseControls()` (`types/Operation.java:116,149`).

### Enablement needs no switch

`GlobalOpenTelemetry` returns a no-op implementation when no agent has installed an SDK, so
the default posture is already "off":

```
no OTel/Datadog agent  ->  no-op API  ->  no spans, negligible cost
agent present          ->  real SDK   ->  tracing works
```

Operators enable tracing with `java -javaagent:opentelemetry-javaagent.jar …`. There is no
enable flag whose state can disagree with whether an SDK is actually installed.

Wren-specific policy still needs configuration — obfuscation mode, filter capture, the
propagation trust allowlist, error result-code set. Give that a dedicated admin-framework
configuration object following `opendj-server-example-plugin` (a `*Configuration.xml`
definition, `Package.xml`, schema LDIF). It configures *how* Wren describes operations, not
*whether* OpenTelemetry exists.

Because these call sites sit on the request hot path, the no-op path must cost
approximately nothing: no allocation, no string building, no obfuscation before
`isRecording()`. Benchmark with no agent installed as part of the work.

## Trace context propagation

### Share the codec, not the control

`org.opends.server.types.Control` is an abstract class with its own `getOID()` /
`getValue()`; the SDK's `org.forgerock.opendj.ldap.controls.Control` is an unrelated
interface. One class cannot satisfy both, and `wrends-telemetry` must not depend back on
`opendj-server-legacy`. So `TraceContextCodec` holds the wire format,
`TraceContextRequestControl` implements the SDK interface for the client, and the server's
start call site decodes directly:

```java
for (org.opends.server.types.Control c : operation.getRequestControls()) {
    if (TraceContextCodec.OID.equals(c.getOID())) {
        Context parent = TraceContextCodec.decode(c.getValue(), propagator);
    }
}
```

### Inject the span context

```java
Span clientSpan = tracer.spanBuilder("ldap.search").setSpanKind(CLIENT).startSpan();
Context toInject = parent.with(clientSpan);
propagator.inject(toInject, controlCarrier, setter);   // not `parent`
```

Injecting the caller context would make the server span a sibling of the client span rather
than its child. Create the span first, then inject.

### Rules

- `isCritical()` must return `false`. A critical unknown control makes any non-instrumented
  server reject the operation with `unavailableCriticalExtension` (12). Cover it with a
  test.
- The control needs a private OID under a Wren Security arc, reserved before PR 10.
- The server side is a trust boundary. Accepting trace context from arbitrary clients lets
  anyone forge parentage or inject unbounded trace IDs, so gate it on an allowlist by
  authenticated identity or client address. The server must not echo the control back.
- Opt-in on both ends, default off.

### `connection_id` and `message_id` are local diagnostics

```
client message ID    == server message ID     ✓  (RFC 4511 echoes the request ID)
client connection ID == server connection ID  ✗  server-assigned, unrelated namespace
```

Message IDs are only unique within an LDAP session, so across a connection pool they
collide freely. These attributes are useful for correlating a span with access-log lines on
one side, but they do not correlate client and server traces — the trace-context control is
the only thing that does. They are also high cardinality, so they are opt-in and off by
default.

## Datadog verification

Two supported paths, neither adding a Datadog dependency:

1. **OTLP** to the Datadog agent's OTLP ingest endpoint.
2. **The Datadog agent's OpenTelemetry API bridge** (`dd.trace.otel.enabled`), where spans
   created through the OTel API are adopted by `dd-trace-java` natively. Verify the flag
   name against the agent version in use.

The same source runs unmodified under the OTel Java agent.

Decide the `db.system.name` shim here, with evidence: enable it server-side, compare
Datadog's classification with and without, and either adopt it as a documented server-only
configurable shim or drop it. Confirm that `service.name` / `service.version` carry product
identity end to end.

**Acceptance:** a client operation and its server operation appear as one distributed trace
with correct parentage (child, not sibling), correct service names, correct error flagging
on a failed bind, and no PII in any attribute or span name.

## Build tasks

- Add `wrends-telemetry` to the reactor `<modules>` in `pom.xml`, after `opendj-core`.
- Add a single `io.opentelemetry:opentelemetry-api` entry to the root
  `dependencyManagement` (`pom.xml:146-249`) with an `${opentelemetry.version}` property,
  plus `opentelemetry-sdk-testing` in `test` scope. Do not import the full OTel BOM.
- Add the `wrends-telemetry` dependency to `opendj-server-legacy` only.
- Configure `maven-bundle-plugin`: export the public package, keep `.internal` private.
- No `japicmp` baseline for the new module; no `opendj-bom` entry while the schema settles.
- Third-party license entry (`src/license/THIRD-PARTY.properties`) for `opentelemetry-api`.

No `opendj-packages` work: the call sites live in `opendj-server-legacy`, which already
ships in the distribution.

## Effort and sequencing

| Phase | Work | Effort |
|---|---|---|
| 0 | Semantic conventions and context lifecycle — settle the sections above | 1 day |
| 1 | Module skeleton: vocabulary, `DirectoryTelemetry`, `LdapOperationTrace`, obfuscation | 3 days |
| 2 | Client: `TelemetryConnectionFactory` / `TelemetryConnection` + tests | 1 week |
| 3 | Server: lifecycle call sites, `ServerTelemetry`, admin config + tests | 1.5 weeks |
| 4 | Propagation: codec, client injection, server extraction | 3 days |
| 5 | Datadog and OTel agent verification | 2 days |

Roughly 4–4.5 weeks for one engineer; phases 2 and 3 parallelize across two once phase 1
lands.

Do the client before the server. It validates the phase 0 vocabulary against a real
consumer, and it is the side that cannot be fixed later by editing application code.

Deferred: `backend.*` child spans, persistent-search observability, metrics, the HTTP/SDK
adapter path, and any instrumentation of `opendj-server` — a 21-file interface skeleton with
no operation dispatch, and upstream of the legacy module
(`opendj-server-legacy/pom.xml:161`).

## Pull request breakdown

Two rules make every PR independently mergeable:

1. **Every PR leaves the build green.** No PR depends on a later one to compile or pass.
2. **Nothing emits a span unless an agent supplies an SDK.** `GlobalOpenTelemetry` is a
   no-op without one, and the client wrapper is opt-in by construction. There is no
   big-bang merge and no flag-flip PR.

All fourteen PRs can therefore land on `main` without changing the observable behaviour of a
running directory. The server call sites do execute on the hot path, so each carries a
no-agent benchmark showing the no-op cost.

Review the span schema and context lifecycle against this document before PR 1 opens — they
are the expensive things to change once dashboards and monitors exist.

### Track A — the module (no consumers, zero runtime impact)

| PR | Contents | Size | Review focus |
|---|---|---|---|
| **1** | Module: pom, reactor entry, `${opentelemetry.version}`, bundle plugin, license. Vocabulary: `LdapAttributes`, `LdapSpanNames`, `LdapResultCodes` | ~350 | The span schema, reviewed against this document rather than the code |
| **2** | `ObfuscationPolicy`, `Dns`, `Filters` (client adapter), limits and truncation | ~350 | The PII boundary. Include a test asserting the obfuscator never calls `AVA.getAttributeValue()`, and a test per limit |
| **3** | `DirectoryTelemetry`, `LdapOperationTrace` | ~250 | Lazy tracer resolution, injectable `OpenTelemetry`, idempotent `complete`/`fail`, no stored `Scope` |

These three are squashable into one ~950-line module PR. Splitting buys focused review on
the two risky parts — 2 is where PII leaks, 3 is where lifecycle bugs live — which get lost
inside a large constants-heavy diff.

### Track B — client (depends on PR 3)

| PR | Contents | Size | Review focus |
|---|---|---|---|
| **4** | `TelemetryConnection` extending `AbstractAsynchronousConnection` + factory, delegation only, no spans. Test exercising every `Connection` method through the wrapper | ~500 | Delegation completeness, answered before span logic exists to distract |
| **5** | Span creation, attributes, completion via `thenOnResultOrException`, `SearchResultHandler` wrapping, `TelemetryOptions` | ~400 | Async lifecycle, the Grizzly-thread completion path, `isRecording()` guards |
| **6** | `ldap.connect` and `ldap.pool.acquire` spans | ~200 | That the two are never conflated |

Splitting 4 from 5 is the highest-value split in the sequence. A wrapper that misses a
delegation is a correctness bug in the *application*, not merely missing telemetry, and it
is invisible in a diff that also introduces span logic.

### Track C — server (depends on PR 3; parallel with Track B)

| PR | Contents | Size | Review focus |
|---|---|---|---|
| **7** | Converge the six `operation.run()` call sites — `SynchronousStrategy:39`, `BoundedWorkQueueStrategy:83,91,247`, `TraditionalWorkerThread:166`, `ParallelWorkerThread:168` — onto one `runOperation(Operation)` helper. Pure refactor, no telemetry | ~120 | Behaviour-preserving across all queueing strategies. Route to a reviewer who knows the work queue |
| **8** | `ServerTelemetry` + the three lifecycle call sites: start in `addOperationInProgress`, activate in `runOperation`, end in `sendResponse`. Abandon handling and the in-flight leak guard | ~450 | Span leaks. Every path that removes an operation from `operationsInProgress` must end its span exactly once |
| **9** | Attributes off `Operation`, server-side `Filters` adapter, `ldap.queue_time`, persistent-search exclusion | ~350 | Schema conformance with Track A; `isRecording()` guards on the hot path |
| **10** | Admin-framework configuration object for obfuscation mode, filter capture, error result-code set | ~350 | Config plumbing; that it configures policy, not whether OpenTelemetry exists |

PR 7 is the highest-leverage split in this track: it is a behaviour-preserving refactor that
can be reviewed and merged on work-queue correctness alone, before any telemetry is in the
diff. PR 8 is where the leak risk lives and deserves the most scrutiny.

### Track D — propagation (depends on PRs 5 and 9)

| PR | Contents | Size | Review focus |
|---|---|---|---|
| **11** | `TraceContextCodec` + `TraceContextRequestControl` + tests | ~250 | `isCritical()` returns `false`, with a test. Pure wire format |
| **12** | Client injection — span context, not caller context — opt-in, default off | ~150 | A test asserting the server span is a child, not a sibling |
| **13** | Server extraction from `org.opends.server.types.Control` + trust allowlist | ~300 | Security review, isolated so the trust boundary is looked at on its own |

PR 11 must not merge before the OID arc is decided.

### Track E — closing out

| PR | Contents | Size |
|---|---|---|
| **14** | Module README: enabling under the OTel and Datadog agents, config reference, the attribute table as user-facing docs, and recommended `service.name` / `service.version` | ~250 |

### Merge order and parallelism

```
  1 ──► 2 ──► 3 ──┬──► 4 ──► 5 ──► 6 ───────┐
                  │                          ├──► 11 ──┬──► 12 ──┐
                  └──► 7 ──► 8 ──► 9 ──► 10 ─┘         └──► 13 ──┴──► 14
```

Tracks B and C are independent once PR 3 lands and touch disjoint files — B is confined to
`wrends-telemetry`, C to `opendj-server-legacy`.

Fourteen PRs averaging ~300 lines. Defensible consolidations are 1+2+3 and 9+10, taking it
to eleven. The splits worth keeping under any consolidation are 4 from 5, 7 on its own, and
13 on its own.

## Risks

| Risk | Mitigation |
|---|---|
| A stored `Scope` leaks across the request/response thread boundary | store `Span` + `Context` only; explicit parenting; no long-lived `Scope` |
| A span held open for a persistent search for hours | detect the persistent-search control at the start call site and skip tracing |
| Span leaked on a path that never reaches `sendResponse` (abandon, disconnect mid-operation) | end the span wherever the operation leaves `operationsInProgress`; drain in-flight operations on disconnect; a test per path |
| Hot-path cost on the accept/run/respond call sites when no agent is installed | no allocation or string building before `isRecording()`; a no-agent benchmark merged with PR 8 |
| The `runOperation` refactor changes queueing behaviour | land it as a standalone behaviour-preserving PR reviewed on work-queue correctness alone |
| Tracer captured before the SDK installs, producing permanent silence | lazy resolution + injectable `OpenTelemetry`; a test asserting spans arrive after late SDK install |
| Client and server spans end up siblings | inject the span context, not the caller context; asserted by a test in PR 12 |
| Critical-flag mistake breaks operations against non-instrumented servers | `isCritical()` hardcoded `false`, covered by a test |
| PII leaking through DNs or filters | structural obfuscator that cannot reach values; filters off by default; a test asserting no raw DN appears in any exported span |
| Unbounded attribute construction from attacker-influenced input | hard limits on DN length, filter depth and control count, with defined truncation |
| Obfuscation cost on sampled-out spans | `span.isRecording()` guard before every expensive computation |
| Result-code error set floods dashboards | explicit exclusion set, configurable |
| Span schema churn leaking into third-party code | `.internal` package split; out of `opendj-bom`; public surface marked experimental |

## Open questions

1. Which OID arc for the trace-context control? Needed before PR 10.
2. Does a `db.system.name` shim materially improve Datadog classification? Decide in
   phase 5 with evidence; adopt only server-side, configurable, defaulting off.
3. Should `ldap.queue_time` also be emitted as a span event marking dequeue, so the wait is
   visible in a waterfall rather than only as an attribute?
4. Do internal operations via `InternalClientConnection` warrant their own span kind later,
   or stay untraced?
