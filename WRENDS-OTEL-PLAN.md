# Wren:DS OpenTelemetry Instrumentation — Implementation Plan

Status: proposal · Target: 5.2.0 · Java 17 · Revised 2026-08-19

## Summary

**One new Maven module, one external compile dependency, no SDK, no Datadog artifact
anywhere in the tree.** `wrends-telemetry` holds the shared tracing vocabulary and
lifecycle helpers; instrumentation lives at the natural operation boundaries in the modules
that own them.

```
opendj-core  ──────────────┐
    ▲                      │
    │                      ▼
    │              wrends-telemetry ──► io.opentelemetry:opentelemetry-api
    │                      ▲
    │                      │
opendj-server-legacy ──────┘        (instrumentation call sites)
```

`opendj-core` is **not modified and does not depend on telemetry.** That is the change
that collapses an earlier four-artifact design down to one: the LDAP client wrapper lives
in `wrends-telemetry` rather than inside the SDK, so the SDK's `japicmp`-frozen public API
is never touched.

Three settled decisions:

1. **OpenTelemetry API only.** `io.opentelemetry:opentelemetry-api` is the single telemetry
   dependency. No SDK, no exporter, no autoconfigure, no collector config — the operator
   supplies the runtime via the OTel or Datadog Java agent. This is OTel's documented
   guidance for libraries.
2. **Library instrumentation, not an agent.** Both sides are code we own and ship.
3. **No vendor abstraction layer.** OTel *is* the abstraction. Zero `datadog.*` imports;
   Datadog compatibility comes from attribute naming plus the DD agent's OTel API bridge.

## Module: `wrends-telemetry`

### Name and coordinates

`wrends-telemetry`, package `org.wrensecurity.wrends.telemetry`. Not `opendj-telemetry` /
`org.wrensecurity.opendj.telemetry` — new modules take the product name, and the package
is a clean break from the inherited `org.forgerock.opendj.*` / `org.opends.server.*` trees.

Not `wrends-telemetry-otel` + `-datadog`. That abstraction is exactly what OTel already is.

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

The `opendj-core` dependency is load-bearing, not incidental: the vocabulary keys off core
types on **both** sides of the wire. `Operation.getResultCode()` in the legacy server
returns `org.forgerock.opendj.ldap.ResultCode` — the core type
(`types/Operation.java:25,178`). DN obfuscation walks `DN`/`RDN`/`AVA`. The trace-context
control implements `org.forgerock.opendj.ldap.controls.Control`. One vocabulary, one set
of types, both sides.

Pin the version in the root `dependencyManagement` (already present, `pom.xml:146-249`) as
a single `opentelemetry-api` entry rather than importing the full OTel BOM — that is the
only OTel artifact Wren compiles against.

### Contents

```
org.wrensecurity.wrends.telemetry
├── DirectoryTelemetry          entry point; resolves the Tracer, holds config
├── LdapOperationSpan           AutoCloseable span lifecycle wrapper
├── TelemetryConnectionFactory  client-side ConnectionFactory decorator
├── TelemetryConnection         client-side Connection decorator
├── attribute/
│   ├── LdapAttributes          AttributeKey constants
│   ├── LdapSpanNames           span naming
│   ├── LdapResultCodes         result code -> span status + name
│   └── LdapControlNames        control OID -> short name
├── obfuscation/
│   ├── ObfuscationPolicy       values | full | none
│   ├── Dns                     DN/RDN obfuscation
│   └── Filters                 filter shape, client + server adapters
└── propagation/
    ├── TraceContextRequestControl   W3C traceparent/tracestate carrier
    └── TraceContextControls         TextMapSetter / TextMapGetter
```

### Treat it as internal, not public API

Do **not** add it to `opendj-bom` initially, and mark the packages internal in Javadoc and
via OSGi (`Private-Package` where the bundle config allows). The span schema needs freedom
to change across the first few releases; publishing it as supported API forfeits that.

Note that "internal" here is a stance, not enforcement — as a compile dependency of
`opendj-server-legacy` the artifact still reaches Maven Central. The BOM omission and the
Javadoc marker are what signal intent.

## Where instrumentation lives

### Server — `opendj-server-legacy`

The legacy module owns the distribution and knows *when* an LDAP operation begins and ends.
Start with an `AccessLogPublisher` implementation as the call site for the root span.

That SPI is a better fit than it first appears: it exposes **both** `logXxxRequest(op)` and
`logXxxResponse(op)` — plus `logSearchRequest`/`logSearchResultDone` and
`logConnect`/`logDisconnect` — so spans get a genuine start and end rather than a synthetic
duration. It also brings runtime enable/disable and configuration for free through the
admin framework, and needs no edits to operation dispatch.

- Start the span in `logXxxRequest`, end it in `logXxxResponse` / `logSearchResultDone`.
- **Stash the span on the operation**, not in a thread-local:
  `Operation.setAttachment(String, Object)` / `getAttachment(String)`
  (`types/Operation.java:491,518`). Request and response hooks are not guaranteed to run on
  the same thread.
- Attributes come off `Operation`: `getOperationType()`, `getResultCode()`,
  `getMessageID()`, `getConnectionID()`, `getMatchedDN()`, `getProcessingTime()`.
  `SearchOperation` adds `getBaseDN()`, `getScope()`, `getFilter()`, `getEntriesSent()`.
- Controls: `getRequestControls()` / `getResponseControls()` (`types/Operation.java:116,149`).
  The server already has a `shouldLogControlOids()` notion in its publishers — align naming.
- Config plumbing follows `opendj-server-example-plugin`: a `*Configuration.xml`
  admin-framework definition, `Package.xml`, and a schema LDIF. Budget real time; it is the
  least familiar part of the work.

**Depth comes later, through explicit call sites.** A publisher-based span is flat — no
children for backend access, index lookup, or plugin execution. When that breakdown is
needed, add `DirectoryTelemetry.startBackendOperation(...)` call sites in the backend code
to produce `ldap.search → backend.search → backend.index`. Deliberately deferred: start
with meaningful logical operations, not a span per internal method.

### Client — no in-tree call sites

The client wrapper *is* the instrumentation. `TelemetryConnectionFactory` decorates any
`ConnectionFactory`; the application wraps its **outermost** factory:

```
TelemetryConnectionFactory          <- one CLIENT span per logical operation
  └── CachedConnectionPool          <- wrap separately for "LDAP pool acquire"
        └── RequestLoadBalancer
              └── LDAPConnectionFactory   <- wrap separately for "LDAP connect"
                    └── GrizzlyLDAPConnection
```

Because placement is explicit, the decorator chain never double-counts and no span
suppression logic is needed.

### `opendj-server` — nothing to instrument

Do not add the telemetry dependency here. `opendj-server` ("Wren:DS Server NG") is a
**21-file skeleton** of interfaces — `DataProvider`, `Operation`, `AttachmentHolder`,
`DataProviderConnection` — with no operation dispatch to hook. It is also *upstream* of the
legacy module: `opendj-server-legacy` depends on `opendj-server` (`opendj-server-legacy/pom.xml:161`),
not the other way round. Revisit only if Server NG grows real request handling.

## The shared API

Centralize span mechanics so implementation code carries none of it, and so one file
defines Wren's tracing contract — naming, safe attributes, error handling, PII rules.

```java
try (LdapOperationSpan span = DirectoryTelemetry.startLdapOperation(operation)) {
    processSearch();
    span.setResultCode(resultCode);
} catch (Throwable t) {
    span.recordException(t);   // see below — close() cannot see this
    throw t;
}
```

### Two corrections to the obvious implementation

**1. `close()` cannot see the throwable.** A bare try-with-resources silently loses error
status: if `processSearch()` throws, `setResultCode` never runs and `close()` has no way to
know the block failed, so the span ends with `StatusCode.UNSET` and no exception recorded.
Either require the `catch` above at every call site, or — better — give the helper an
explicit failure path and make the contract impossible to get wrong:

```java
final class LdapOperationSpan implements AutoCloseable {
    private final Span span;
    private final Scope scope;
    private boolean completed;

    void setResultCode(ResultCode rc) {
        completed = true;
        span.setAttribute(LdapAttributes.RESULT_CODE, rc.intValue());
        span.setAttribute(LdapAttributes.RESULT_CODE_NAME, rc.getName());
        if (LdapResultCodes.isError(rc)) {
            span.setStatus(StatusCode.ERROR);
        }
    }

    void recordException(Throwable t) {
        completed = true;
        span.recordException(t);
        span.setStatus(StatusCode.ERROR);
    }

    @Override
    public void close() {          // idempotent
        if (!completed) {
            span.setStatus(StatusCode.ERROR, "operation ended without a result");
        }
        scope.close();             // scope before span, always
        span.end();
    }
}
```

**2. Do not capture the Tracer in a `static final` field.** A tracer resolved at class-init
time can be obtained from `GlobalOpenTelemetry` before the agent or SDK has installed
itself, and the instrumentation then emits nothing for the process lifetime — a failure
mode that produces no error, just silence. Resolve lazily on first use, and accept an
injected `OpenTelemetry` instance for tests and embedded use:

```java
public final class DirectoryTelemetry {
    private static volatile DirectoryTelemetry instance;

    public static DirectoryTelemetry get() { ... }              // lazy global
    public static DirectoryTelemetry using(OpenTelemetry otel) { ... }  // injected
}
```

The injected form is not optional extra credit — it is how the tests assert against an
in-memory exporter without touching global state.

Advanced code may still reach for `Span.current()` where it genuinely needs to, but that
should not become the normal pattern.

## Semantic conventions

**Settle this before any code.** There is no OpenTelemetry semantic convention for LDAP.
Model on the database conventions where they fit; add an `ldap.*` namespace for what they
do not. This is the contract `LdapAttributes` encodes — if client and server disagree on
it, correlation is dead on arrival.

| Attribute | Example | Side | Notes |
|---|---|---|---|
| `db.system.name` | `ldap` | both | drives vendor span-type mapping |
| `ldap.operation` | `search` | both | lowercase, normalized across both codebases |
| `ldap.dn` | `uid=?,ou=?,dc=?,dc=?` | both | values always obfuscated |
| `ldap.result_code` | `32` | both | numeric |
| `ldap.result_code.name` | `noSuchObject` | both | |
| `ldap.matched_dn` | `ou=?,dc=?` | both | on failure only |
| `ldap.message_id` | `7` | both | correlation key |
| `ldap.connection_id` | `114` | both | correlation key |
| `ldap.request.controls` | `paged-results,proxied-auth-v2` | both | sorted, comma-joined short names |
| `ldap.response.controls` | `password-policy` | both | |
| `ldap.search.scope` | `wholeSubtree` | both | |
| `ldap.search.filter` | `(&(objectClass=?)(uid=?))` | both | **off by default**, shape only |
| `ldap.search.entries_returned` | `1` | both | |
| `ldap.search.page_size` | `500` | both | `SimplePagedResultsControl.getSize()` |
| `ldap.response.password_policy.error` | `passwordExpired` | both | high-signal, non-PII |
| `server.address` / `server.port` | `ds01.example.com` / `1636` | client | |
| `ldap.tls.enabled` / `ldap.tls.starttls` | `true` | client | |
| `error.type` | `LdapException` | both | on failure |

### Span shapes

| Span | Kind | Where |
|---|---|---|
| `{operation} {ldap.dn}` | `CLIENT` | one per logical client operation |
| `LDAP connect` | `CLIENT` | real TCP connect + TLS + initial bind |
| `LDAP pool acquire` | `INTERNAL` | pool wait time — never conflated with connect |
| `{operation} {ldap.dn}` | `SERVER` | one per server-side operation |
| `backend.*` | `INTERNAL` | deferred; explicit call sites in backend code |

Span names use the **obfuscated** DN, which is what keeps them low-cardinality — the same
mechanism that makes `GET /users/?` a usable resource name. Connections are long-lived and
are modelled as attributes, never as spans.

### Error classification

Do not port "non-zero result code is an error" from the HTTP conventions; it floods
dashboards with false errors. Default the error set to every result code **except**
`{0 success, 5 compareFalse, 6 compareTrue, 10 referral, 14 saslBindInProgress}`, and make
it configurable. `compareFalse` is a successful compare; `noSuchObject` and
`invalidCredentials` are routine outcomes on an authentication path. Always set
`ldap.result_code` regardless, so teams can build their own monitors on top.

### Obfuscation

DN obfuscation is an allowlist, not a scrub — walk the structure and emit attribute *types*
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

`uid=bjensen,ou=people,dc=example,dc=com` → `uid=?,ou=?,dc=?,dc=?`. Because it cannot reach
a value, it cannot leak one. Modes: `values` (default), `full` (drop attribute types too),
`none` (raw, lab use only).

**The filter obfuscator needs two adapters.** The client sees
`org.forgerock.opendj.ldap.Filter`; the server sees `org.opends.server.types.SearchFilter`.
Unrelated types. Define the shape algorithm once against a small internal interface.

**Never emit control values.** `ProxiedAuthV2RequestControl`'s value *is* an authzid — a
full DN. `SimplePagedResultsControl.getCookie()` is opaque server state. Decode only the
two special-cased fields in the table above. Hardcode the OID→name lookup rather than
referencing the ~30 SDK control classes.

## Client implementation notes

### The trap: do not extend `AbstractConnectionWrapper`

`AbstractConnectionWrapper.add(request)` delegates straight to `connection.add(request)`
(`AbstractConnectionWrapper.java:82-84`) — it does **not** route synchronous calls through
the async methods. Overriding only `*Async` on a wrapper would silently miss every blocking
call in the application.

Extend **`AbstractAsynchronousConnection`** instead and hold the delegate as a field. Its
synchronous methods are all `blockingGetOrThrow(xxxAsync(request))`
(`AbstractAsynchronousConnection.java:45-83`), so instrumenting the nine `*Async` methods
covers the blocking API for free. Cost: hand-implementing ~10 non-operation methods
(`close`, `isValid`, listener registration, …). The convenience overloads on
`AbstractConnection` come along correctly.

The same applies to `LDAPConnectionFactory.getConnection()`, which is
`getConnectionAsync().getOrThrowUninterruptibly()` (`LDAPConnectionFactory.java:440`).

### Span lifecycle

Start the span in the `*Async` method, end it from
`LdapPromise#thenOnResultOrException(ResultHandler, ExceptionHandler)`
(`LdapPromise.java:54`). No context store needed.

**That callback fires on a Grizzly worker thread.** End the span there; do not activate a
`Scope` you will not close. Capture the caller's `Context` at request time and use it as
parent explicitly rather than relying on thread-locals. Note that `LdapOperationSpan`'s
try-with-resources pattern does **not** fit the async client path — it is for the
synchronous server side. The client uses the promise callback directly.

Search entry counts come from wrapping the `SearchResultHandler` argument.

Configuration is exposed as `Option<...>` constants, consistent with the SDK's existing
style: enable/disable, DN obfuscation mode, filter capture (default off), error result-code
set, propagation on/off.

## Trace context propagation

The client injects `TraceContextRequestControl` via the configured `TextMapPropagator`; the
server extracts it in `logXxxRequest` and uses it as the parent context. Opt-in on both
ends, default off.

- **`isCritical()` must return `false`.** A critical unknown control makes any
  non-instrumented server reject the operation with `unavailableCriticalExtension` (12) —
  every directory that has not been upgraded. Cover it with a test.
- Needs a private OID under a Wren Security arc. **Reserve it before implementation
  starts** — it is painful to change after release.
- **The server side is a trust boundary.** Accepting trace context from arbitrary clients
  lets anyone forge parentage or inject unbounded trace IDs. Gate it on an allowlist by
  authenticated identity or client address, and document it as such. The server must not
  echo the control back in the response.

Until this ships, `ldap.connection_id` + `ldap.message_id` are emitted by both sides from
day one and give a manual join key in any backend, at zero protocol cost.

## Datadog verification

Two supported paths, neither adding a Datadog dependency to this repo:

1. **OTLP** to the Datadog agent's OTLP ingest endpoint. `db.system.name`, `span.kind`,
   `server.address`/`server.port` and `service.name` drive Datadog's mapping to span type,
   service and peer.
2. **The Datadog agent's OpenTelemetry API bridge** (`dd.trace.otel.enabled`) — spans
   created through the OTel API are adopted by `dd-trace-java` natively. Verify the flag
   name against the agent version in use.

The same source also runs unmodified under the OTel Java agent, which explicitly supports
applications making plain OTel API calls.

**Acceptance:** a client operation and its server operation appear as one distributed trace
with correct parentage, correct service names, correct error flagging on a failed bind, and
no PII in any tag or span name.

## Build tasks

- Add `wrends-telemetry` to the reactor `<modules>` in `pom.xml`, after `opendj-core`.
  (Maven derives order from dependencies, but listing a foundational module before its
  consumers keeps the POM readable.)
- Add a single `io.opentelemetry:opentelemetry-api` entry to the root
  `dependencyManagement` (`pom.xml:146-249`) with an `${opentelemetry.version}` property.
  Do not import the full OTel BOM.
- Add the `wrends-telemetry` dependency to `opendj-server-legacy` only.
- Configure `maven-bundle-plugin` for the new module, following `opendj-core`'s
  instructions block.
- No `japicmp` baseline (no prior release); no `opendj-bom` entry while it is internal.
- Wire the publisher into the server distribution via `opendj-packages`.
- Third-party license entry (`src/license/THIRD-PARTY.properties`) for `opentelemetry-api`.

## Effort and sequencing

| Phase | Work | Effort |
|---|---|---|
| 0 | Semantic conventions — settle the table above | 1 day |
| 1 | `wrends-telemetry` skeleton: vocabulary, `DirectoryTelemetry`, `LdapOperationSpan`, obfuscation | 3 days |
| 2 | Client: `TelemetryConnectionFactory` / `TelemetryConnection` + tests | 1 week |
| 3 | Server: `AccessLogPublisher` + admin config + tests | 1 week |
| 4 | Propagation control, both directions | 3 days |
| 5 | Datadog + OTel agent verification | 2 days |

Roughly **3.5–4 weeks** for one engineer; phases 2 and 3 parallelize across two once
phase 1 lands.

**Do the client before the server.** It validates the phase 0 vocabulary against a real
consumer, and it is the side that cannot be fixed later by editing application code.

Deferred beyond this plan: `backend.*` child spans, metrics, and any instrumentation of
`opendj-server`.

## Risks

| Risk | Mitigation |
|---|---|
| Tracer captured before the SDK installs → permanent silence | lazy resolution + injectable `OpenTelemetry`; a test asserting spans arrive after late SDK install |
| `close()` swallowing error status | `completed` flag marks unfinished spans ERROR; call-site pattern documented once |
| Critical-flag mistake breaks ops against non-instrumented servers | `isCritical()` hardcoded `false`, covered by a test |
| PII leaking through DNs or filters | structural obfuscator that cannot reach values; filters off by default; a test asserting no raw DN appears in any exported span |
| Result-code error set floods dashboards | explicit exclusion set, configurable |
| Client and server vocabularies drift | single module; a test asserting both paths emit identical attribute keys |
| Admin-framework config plumbing is unfamiliar | budget it explicitly; `opendj-server-example-plugin` is the working template |
| Span schema churn leaking into third-party code | keep out of `opendj-bom`, mark packages internal |

## Open questions

1. Which OID arc for `TraceContextRequestControl`? Needed before phase 4.
2. Should the publisher ship inside the server distribution by default (disabled), or as a
   separately installed extension?
3. Metrics as well as traces? The `AccessLogPublisher` hooks would support operation
   counters and latency histograms cheaply — but that is scope beyond this plan.
