# Phase 1 Data Model — Spring Debug Trace Starter v1

**Date**: 2026-05-28
**Branch**: `001-v1-starter`
**Scope**: Ephemeral runtime entities used during request handling. Nothing in v1 is persisted to disk, a database, or an external store.

This document captures the in-memory model that the library carries through a single request — what each entity holds, who owns it, when it is created and cleaned up, and how it crosses thread boundaries (it does not).

For domain-level reference, see `spec.md` § *Key Entities*.

---

## Conventions

- **Owner**: which component creates, reads, and clears the entity.
- **Lifecycle**: when the entity is born and when it is reclaimed.
- **Thread-affinity**: per-thread (thread-local), per-request (MDC), per-call (stack-allocated), or shared (process-wide / singleton).
- **Mutability**: immutable, append-only, or mutable.
- **Concurrency**: how concurrent access is handled (or why it cannot occur).

All entities are JVM heap objects. None survives a request, except shared singletons (the masking rule set and the match-decision cache), which are recomputed at startup from configuration.

---

## E1. Call-chain context

The per-thread depth counter that drives indentation in the pretty log format.

| Aspect | Value |
|---|---|
| **Owner** | `LoggingContext` (Phase 1) |
| **Storage** | `ThreadLocal<Integer>` (initial value 0) |
| **Lifecycle** | First push at `MethodLoggingAspect.before()`; root pop calls `ThreadLocal.remove()` to avoid leaks on pooled threads. On the exception path, the pop happens in `finally`. |
| **Thread-affinity** | Per-thread. Not propagated across `@Async`, `CompletableFuture`, executors, or message listeners (PRD §6 non-goal 10). |
| **Mutability** | Mutable counter — `push` increments, `pop` decrements. |
| **Concurrency** | None — one counter per thread. |

**Fields**:
- `depth: int` — current nesting level. `0` means "outside any intercepted call."

**Operations**:
- `push(): int` — increment then return the new depth (caller uses the return value to indent its entry line).
- `pop(): int` — decrement and return the new depth.
- `isRoot(): boolean` — `depth == 0`.
- `clearIfRoot()` — when popping back to 0, call `ThreadLocal.remove()` so the counter does not pin to a pool thread.

**Invariants**:
1. After a root call completes — whether by normal return or by exception — `depth` MUST be exactly `0` and the `ThreadLocal` MUST be removed (Constitution Principle I; FR-010, FR-011).
2. `push` and `pop` MUST be balanced even when `proceed()` throws — the aspect uses `try`/`finally`.

---

## E2. Trace event

A single log line about a method call. Trace events are *emitted* — they are not held in a list; we never aggregate.

| Aspect | Value |
|---|---|
| **Owner** | `MethodLoggingAspect` — constructs the values; `LogFormatter` renders them; SLF4J consumes them. |
| **Storage** | Stack-allocated locals inside the around-advice. No heap retention beyond the formatted `String` passed to SLF4J. |
| **Lifecycle** | Born when the aspect's around-advice is entered; dies as soon as `log.debug(...)` returns. |
| **Thread-affinity** | Implicitly per-call — never shared. |
| **Mutability** | Immutable values used once. |
| **Concurrency** | None. |

**Conceptual fields** (these are method parameters of `LogFormatter`, not a class):
- `kind: enum { ENTRY, EXIT, EXCEPTION }`
- `depth: int` — from `LoggingContext.depth()`.
- `targetClass: Class<?>` — the intercepted bean's class.
- `methodName: String`
- `argumentSummary: String?` — present only when `method.log-input=true`. May be `<serialization-failed: ClassName>`.
- `returnValueSummary: String?` — present only on `EXIT` and only when `method.log-output=true`.
- `exceptionType: Class<? extends Throwable>?` — present only on `EXCEPTION`.
- `exceptionMessage: String?` — present only on `EXCEPTION`.
- `elapsedMillis: long?` — present on `EXIT` and `EXCEPTION`. Computed from `System.nanoTime()` deltas, not `System.currentTimeMillis()` (FR-009).
- `traceId: String?` — pulled from MDC just before formatting.

**Rendering**: see `contracts/log-format.md`.

---

## E3. HTTP envelope event

The bookend lines emitted by `HttpLoggingFilter` (Phase 5+) at the start and end of a Spring MVC request.

| Aspect | Value |
|---|---|
| **Owner** | `HttpLoggingFilter` |
| **Storage** | Stack-allocated locals inside `doFilter`. |
| **Lifecycle** | Entry line is logged before `chain.doFilter(...)`; exit line is logged in `finally` after the filter chain returns. |
| **Thread-affinity** | Per-call. Servlet container assigns one request to one thread for the duration of `doFilter`. |
| **Mutability** | Immutable values used once. |
| **Concurrency** | None. |

**Conceptual fields**:
- `kind: enum { HTTP_ENTRY, HTTP_EXIT }`
- `httpMethod: String` — e.g. `POST`, `GET`.
- `requestUri: String` — request URI including path.
- `queryString: String?` — present iff non-empty; never logged when `http.log-headers=false`? NO — query string is metadata and is included by default with the HTTP entry line per FR-9.
- `responseStatus: int?` — present on `HTTP_EXIT` only.
- `elapsedMillis: long?` — present on `HTTP_EXIT`.
- `traceId: String?` — read from MDC; same source as Trace events.

**Invariant**: every Trace event emitted between an `HTTP_ENTRY` and its matching `HTTP_EXIT` on the same thread MUST carry the same `traceId` as the envelope (SC-005).

---

## E4. Trace ID

A per-request correlation string that joins HTTP envelope events with all Trace events for the same request.

| Aspect | Value |
|---|---|
| **Owner** | `TraceIdManager` (Phase 5) |
| **Storage** | `org.slf4j.MDC` under key `trace-id.mdc-key` (default `traceId`). |
| **Lifecycle** | Resolved at the start of `HttpLoggingFilter.doFilter`; cleaned in `finally`. |
| **Thread-affinity** | Per-thread via MDC's thread-local map. |
| **Mutability** | Immutable string. The MDC slot is set once per request. |
| **Concurrency** | None — MDC is per-thread. |

**Fields**:
- `value: String` — UUID-format when generated; opaque when sourced from upstream.
- `source: enum { EXISTING_MDC, REQUEST_HEADER, GENERATED }` — used only internally to drive cleanup behavior.

**Lifecycle decisions**:
- `source == EXISTING_MDC` — DO NOT remove from MDC on cleanup (we did not put it there).
- `source == REQUEST_HEADER` — DO remove (we promoted the header value into MDC).
- `source == GENERATED` — DO remove.

**Resolution order** (`TraceIdManager.resolve()`):
1. Existing MDC value under the configured key.
2. First non-blank value from `trace-id.request-headers` in declaration order.
3. `UUID.randomUUID().toString()` if `trace-id.generate-if-missing=true`.
4. `null` otherwise.

**Invariants**:
1. Existing externally-managed trace IDs are NEVER overwritten (FR-033).
2. The MDC slot the library set MUST be removed on filter exit even if the wrapped servlet handler threw (FR-033, Story 5 acceptance scenario 4).

---

## E5. Masking rule set

The combined default + user-supplied list of field names and header names whose values are replaced in log output.

| Aspect | Value |
|---|---|
| **Owner** | `SensitiveDataMasker` (Phase 4) |
| **Storage** | Two immutable `Set<String>` instances inside the masker singleton: `fieldNames` (case-folded to lowercase at construction) and `headerNames` (Phase 6 — same shape). |
| **Lifecycle** | Constructed once at bean creation (`@PostConstruct` or constructor). Reused for every serialization. |
| **Thread-affinity** | Shared. Read-only after construction. |
| **Mutability** | Immutable. Reconfiguration requires application restart (standard Spring `@ConfigurationProperties` semantics). |
| **Concurrency** | Safe — set is `Set.copyOf(...)`, the underlying array is final. |

**Fields**:
- `enabled: boolean` — from `masking.enabled` (default `true`).
- `replacement: String` — from `masking.replacement` (default `"****"`).
- `fieldNames: Set<String>` — defaults from PRD §10 (`password`, `pass`, `token`, `accessToken`, `refreshToken`, `authorization`, `cookie`, `secret`, `apiKey`, `cardNumber`, `iban`) UNION user-supplied `masking.fields`. All entries lowercased. **User additions append; they do NOT replace** (Constitution Principle II, FR-022).
- `headerNames: Set<String>` (Phase 6) — defaults `authorization`, `cookie`, `set-cookie`, plus the configured field-name list (since header values and DTO field values often overlap).

**Operations**:
- `shouldMask(name: String): boolean` — `enabled && fieldNames.contains(name.toLowerCase(Locale.ROOT))`.
- `maskValue(originalValue: Object): String` — returns `replacement` regardless of input type. Never throws.
- `maskJsonBodyKeys(body: String): String` (Phase 6) — regex replace on `"<key>"\s*:\s*"..."` for each field name.

**Invariants**:
1. Construction MUST NOT throw on null/empty `masking.fields` — treat as the empty list and union with defaults.
2. A serialization or masking failure MUST NOT cause the original unmasked value to escape into the log (FR-019, FR-021, Story 4 acceptance scenario 4). On internal failure return the configured replacement, not the original value.

---

## E6. Match decision

The cached per-class verdict about whether a bean class is in scope for method logging.

| Aspect | Value |
|---|---|
| **Owner** | `MethodLogMatcher` (Phase 1; cache added Phase 7) |
| **Storage** | `ConcurrentHashMap<Class<?>, MatchVerdict>` inside the matcher singleton (Phase 7). Phase 1 may compute eagerly per call until the cache is introduced. |
| **Lifecycle** | Populated lazily on first method invocation per class; lives for the lifetime of the application context. |
| **Thread-affinity** | Shared. Reads are lock-free; insertions via `computeIfAbsent`. |
| **Mutability** | Append-only at runtime. Cleared at context shutdown by GC. |
| **Concurrency** | `ConcurrentHashMap` handles concurrent computes; verdict computation is pure (no side effects) so duplicate computes are harmless. |

**Verdict values** (`MatchVerdict` enum):
- `INCLUDE` — the class is in scope; emit logs for its intercepted methods.
- `SKIP_OUT_OF_SCOPE` — the class is outside `base-packages`, or matches the default-exclude list, or matches `excluded-packages`, or matches an `excluded-class-name-patterns` regex, or is in the library's own package.
- `SKIP_REPOSITORY` (Phase 7) — the class is a Spring `Repository` and `method.include-repositories=false`.

**Decision algorithm**:
1. If class's package starts with `io.github.cerovskimatija.debugtrace` → `SKIP_OUT_OF_SCOPE` (Constitution Principle V; FR-003).
2. If class's package starts with any default-exclude prefix → `SKIP_OUT_OF_SCOPE` (FR-004).
3. If class's package starts with any user-configured `excluded-packages` prefix → `SKIP_OUT_OF_SCOPE` (FR-005).
4. If class's fully-qualified name matches any `excluded-class-name-patterns` regex → `SKIP_OUT_OF_SCOPE` (Phase 7).
5. If `method.include-repositories=false` and class is annotated `@Repository` or implements `org.springframework.data.repository.Repository` → `SKIP_REPOSITORY` (Phase 7).
6. If no `base-packages` is configured (empty list) → `SKIP_OUT_OF_SCOPE` (FR-002, R2).
7. If class's package starts with any `base-packages` prefix → `INCLUDE`.
8. Otherwise → `SKIP_OUT_OF_SCOPE`.

**Invariants**:
1. The library's own package MUST always evaluate to `SKIP_OUT_OF_SCOPE` regardless of `base-packages` — recursive self-logging is forbidden (FR-003, Constitution Principle V).
2. The cache key is `Class<?>`, not the proxy class — when AOP creates a CGLIB subclass, the matcher reads `class.getSuperclass()` if the class is a CGLIB proxy (`$$EnhancerBySpringCGLIB$$`) to look up the original.

---

## E7. Serialization snapshot (per-call)

The transient structures used by `SafeLogSerializer` while walking an argument or return value.

| Aspect | Value |
|---|---|
| **Owner** | `SafeLogSerializer` (Phase 2) |
| **Storage** | A `StringBuilder` (output buffer) and an `IdentityHashMap<Object, Boolean>` (visited set), both stack-local to the serializer call. |
| **Lifecycle** | Born on each `SafeLogSerializer.summarize(...)` call; dies on return. |
| **Thread-affinity** | Per-call. The serializer instance is shared, but it MUST NOT hold any mutable state in fields. |
| **Mutability** | Mutable within the call; not exposed. |
| **Concurrency** | None — each call has its own buffer and visited set. |

**Fields and limits** (from `serialization.*` config):
- `maxDepth: int` (default `3`) — recursion cap; deeper objects are summarized as `<truncated, depth>`.
- `maxStringLength: int` (default `1000`) — strings beyond this are cut with a `... (truncated)` suffix.
- `maxCollectionSize: int` (default `20`) — only the first N elements are walked; the rest become `... (+M more)`.
- `maxObjectLength: int` (default `5000`) — overall buffer length cap; exceeding it terminates the walk with `... (object truncated)`.
- `includeNullFields: boolean` (default `false`) — when `false`, fields whose value is `null` are omitted from the rendered form.

**Skip categories** (`SkipTypeDetector`, Phase 2):
- `HttpServletRequest`, `HttpServletResponse`, `InputStream`, `OutputStream`, `MultipartFile`, `File`, `Resource`, `BindingResult`, `Principal`, `Authentication`, byte arrays larger than 128 bytes — each emitted as a short placeholder identifying the type and minimal info (e.g. `<MultipartFile name=upload size=1234>`).

**Invariants**:
1. The serializer NEVER throws — every internal failure is caught and replaced with `<serialization-failed: SimpleClassName>` (Constitution Principle I; FR-019).
2. Circular references are detected via identity (`==`, not `equals`) and replaced with `<cycle: ClassName>` once.
3. Lazy-loaded JPA proxies that throw on access are caught at the field level and replaced with `<lazy-access-failed: ClassName>`.

---

## E8. Body capture buffer (Phase 6)

The captured-for-logging copy of an HTTP request or response body. Distinct from the actual body delivered to the client.

| Aspect | Value |
|---|---|
| **Owner** | `HttpLoggingFilter` wraps each request in `ContentCachingRequestWrapper` and each response in `ContentCachingResponseWrapper`. The captured bytes live inside those wrappers. |
| **Storage** | `byte[]` inside the wrapper, up to `http.max-body-length` bytes (default `5000`). |
| **Lifecycle** | Filled as the servlet writes the response; read once at filter exit; the response wrapper THEN calls `copyBodyToResponse()` to release the captured bytes back to the client. |
| **Thread-affinity** | Per-request. |
| **Mutability** | Append-only during the request, then read-only for the brief logging window. |
| **Concurrency** | None — single request, single thread. |

**Invariants**:
1. `copyBodyToResponse()` MUST be called for the response wrapper before the filter chain unwinds, regardless of whether logging succeeded or threw (Constitution Principle I, SC-006).
2. Bodies whose `Content-Type` is binary or multipart are NOT decoded for logging — the wrapper still captures bytes, but the log line shows `<binary, N bytes>` / `<multipart, N parts>` instead of the body.
3. Logged bodies are truncated to `http.max-body-length` characters with an explicit `... (truncated)` indicator. The truncation operates on the captured-for-logging copy only and never on the bytes sent to the client.

---

## Entity relationships

The diagram below shows the data flow during one Spring MVC request with HTTP logging and method logging both enabled:

```text
                Servlet container thread
                │
HTTP request ──►│
                │  HttpLoggingFilter (Phase 5+)
                │  ├─ TraceIdManager.resolve()    ─► writes E4 (Trace ID) into MDC
                │  ├─ emit E3 HTTP_ENTRY          ─► LogFormatter ─► SLF4J
                │  ├─ (Phase 6) ContentCaching*Wrapper ─► holds E8
                │  │
                │  │  chain.doFilter(...)
                │  │  │
                │  │  │  MethodLoggingAspect (Phase 1+)
                │  │  │  ├─ MethodLogMatcher.match(targetClass) ─► consults E6 (Match decision cache)
                │  │  │  ├─ LoggingContext.push()                ─► mutates E1 (Call-chain context)
                │  │  │  ├─ SafeLogSerializer.summarize(args)    ─► consumes E5 (Masking rule set) via SkipTypeDetector + SensitiveDataMasker
                │  │  │  ├─ emit E2 ENTRY                        ─► LogFormatter reads E1.depth + MDC for E4
                │  │  │  ├─ Object result = proceed()
                │  │  │  ├─ SafeLogSerializer.summarize(result)
                │  │  │  ├─ emit E2 EXIT (or EXCEPTION)          ─► LogFormatter ...
                │  │  │  └─ LoggingContext.pop() / clearIfRoot() ─► mutates E1
                │  │  │
                │  │  ▼
                │  ├─ (Phase 6) response.copyBodyToResponse()    ─► drains E8 back to client
                │  ├─ emit E3 HTTP_EXIT
                │  └─ TraceIdManager.cleanup()                   ─► removes E4 from MDC if we put it there
                │
HTTP response ◄─┘
```

All E1, E4, E8 lifecycles are scoped to this thread+request. E5 and E6 are shared singletons that survive the request.

---

## What this model does NOT include (intentional v1 omissions)

- **Persistence**: no database, no file, no log aggregation. Out of scope (Constitution Principle III; PRD §6 non-goal 11).
- **Cross-thread propagation**: nothing in E1 or E4 follows `@Async`, `CompletableFuture`, or executor handoffs (PRD §6 non-goal 10).
- **Distributed trace context**: no W3C `traceparent` parsing beyond reading it as an opaque header value into E4 (PRD §6 non-goal 9).
- **Per-method or per-class enable/disable**: the v1 toggles are package- and pattern-level. Per-method filters are PRD §22 future work.
- **Sampling**: every in-scope call is logged. Sampling is PRD §22 future work.
- **Bytecode-level state**: no Java agent state, no instrumentation hooks (PRD §6 non-goal 4; Constitution Principle V).
