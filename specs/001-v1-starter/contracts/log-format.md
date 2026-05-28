# Contract: Log Format

**Date**: 2026-05-28
**Branch**: `001-v1-starter`
**Scope**: The visual contract of every log line the library emits in v1.

PRD §8 is the source of truth for example output. This document formalizes the structure of those examples so:

- `LogFormatter` has an unambiguous specification to implement (Phase 1, extended in Phase 5).
- `LogFormatterTest` (PRD §20) has a concrete checklist of cases.
- SC-005 ("filter by trace ID, get a contiguous chain") has a verifiable contract.
- Tests that assert on log output have a stable grammar to match against.

All log lines are emitted at SLF4J `DEBUG` level through the dedicated logger named `io.github.cerovskimatija.debugtrace`. Internal warnings (e.g. "serialization fell back to placeholder") emit at `WARN` on the same logger.

---

## 1. Line grammar (informal BNF)

```text
log-line       := trace-prefix? indent? body
trace-prefix   := "[traceId=" trace-id "] "        ; present iff MDC has the configured key (default "traceId")
indent         := "  " * depth                     ; two spaces per call depth (depth==0 → no indent)
body           := entry-body | exit-body | exception-body | http-entry-body | http-exit-body

entry-body     := "→ " fqmn args?
exit-body      := "← " fqmn output? " took=" millis "ms"
exception-body := "× " fqmn " threw " exception-type "(message=" message ")" " after " millis "ms"
http-entry-body:= "HTTP → " http-method " " request-uri query?
http-exit-body := "HTTP ← " http-method " " request-uri " status=" status " took=" millis "ms"

fqmn           := simple-class-name "." method-name
simple-class-name := <Class.getSimpleName() of the bean (NOT the CGLIB proxy)>
method-name    := <Method.getName()>
args           := "(input=[" serialized-args "])"   ; iff method.log-input=true
output         := "(output=" serialized-value ")"   ; iff method.log-output=true; omitted for void returns
serialized-args:= comma-separated `SafeLogSerializer.summarize(arg)` outputs
serialized-value := `SafeLogSerializer.summarize(returnValue)`
millis         := non-negative integer (rounded from System.nanoTime() delta)
exception-type := exception.getClass().getSimpleName()
message        := exception.getMessage() (may be empty string when null)
http-method    := the HTTP method as-received
request-uri    := request URI path (no host)
query          := "?" raw-query-string (omitted when query is null/empty)
status         := response HTTP status code (integer)
trace-id       := the value read from MDC under the configured key, opaque string
```

**Two-space indent** matches the PRD §8 example output. Depth 0 → zero leading spaces. Depth 1 → two spaces. Depth N → 2N spaces.

**Indent applies to the body, not to the trace prefix.** The `[traceId=...] ` prefix is always at column 0 of the SLF4J message; the indent and arrow follow.

---

## 2. Example renderings

### 2.1 Normal nested chain (Phase 1+)

PRD §8 reference, with `method.log-input=true` and `method.log-output=true`:

```text
[traceId=abc123] HTTP → POST /orders
[traceId=abc123] → OrderController.createOrder(input=[CreateOrderRequest{productId=10, quantity=2}])
[traceId=abc123]   → OrderService.createOrder(input=[CreateOrderRequest{productId=10, quantity=2}])
[traceId=abc123]     → PaymentService.charge(input=[PaymentRequest{amount=49.99}])
[traceId=abc123]     ← PaymentService.charge(output=PaymentResult{status=APPROVED}) took=42ms
[traceId=abc123]   ← OrderService.createOrder(output=Order{id=1001}) took=88ms
[traceId=abc123] ← OrderController.createOrder(output=ResponseEntity{status=201}) took=104ms
[traceId=abc123] HTTP ← POST /orders status=201 took=120ms
```

Indent steps: 0 (HTTP) → 0 (controller) → 2 (service) → 4 (payment) → back up.

### 2.2 Exception in a nested call (Phase 3+)

PRD §8 exception reference:

```text
[traceId=abc123] → PaymentService.charge(input=[PaymentRequest{amount=49.99}])
[traceId=abc123] × PaymentService.charge threw PaymentDeclinedException(message=Card declined) after 21ms
```

The exception line REPLACES what would have been the `←` exit line at the same indent.

### 2.3 Without trace ID (Phase 1, 2, 3, 4 — before Phase 5)

```text
→ OrderController.createOrder()
  → OrderService.createOrder()
    → PaymentService.charge()
    ← PaymentService.charge() took=42ms
  ← OrderService.createOrder() took=88ms
← OrderController.createOrder() took=104ms
```

No `[traceId=...]` prefix because nothing has been written into MDC. This is the Phase 1 acceptance demo (PRD-phases.md).

### 2.4 Inputs/outputs off (Phase 1)

Same shape as 2.3 — no `(input=...)` and no `(output=...)`, just `took=Nms` on exits.

### 2.5 Void return value (Phase 2+)

```text
[traceId=...] → InventoryService.reserve(input=[Sku{id=42}])
[traceId=...] ← InventoryService.reserve took=12ms
```

`void` methods OMIT the `(output=...)` segment entirely — no `(output=void)`, no `(output=null)`. (Distinct from a `null` return on a non-`void` method, which renders as `(output=null)`.)

### 2.6 Masked field in serialized object (Phase 4+)

```text
[traceId=...] → LoginController.login(input=[LoginRequest{username=alice, password=****}])
```

### 2.7 Serializer fallback (Phase 3+)

When `SafeLogSerializer` itself fails on a particular value (e.g., a getter throws), that value is replaced with the fallback string in place:

```text
[traceId=...] → SearchService.find(input=[<serialization-failed: WeirdQuery>])
[traceId=...] ← SearchService.find(output=<serialization-failed: SearchResponse>) took=14ms
```

The surrounding chain remains intact.

### 2.8 Truncation (Phase 2+)

Long strings:

```text
description=The quick brown fox ... (truncated)
```

Large collections:

```text
items=[Item{...}, Item{...}, Item{...}, ... (+997 more)]
```

Whole-object overflow:

```text
(output=Order{... (object truncated)})
```

### 2.9 Skip-type placeholders (Phase 2+)

Common runtime types render as type-tagged short summaries rather than dumps:

| Type | Rendered as |
|---|---|
| `MultipartFile` | `<MultipartFile name=upload size=1234>` |
| `HttpServletRequest` | `<HttpServletRequest method=POST uri=/orders>` |
| `HttpServletResponse` | `<HttpServletResponse status=200>` |
| `InputStream` | `<InputStream>` |
| `OutputStream` | `<OutputStream>` |
| `File` | `<File path=/tmp/...>` |
| `Resource` | `<Resource description=...>` |
| `BindingResult` | `<BindingResult errors=2>` |
| `Principal` | `<Principal name=alice>` |
| `Authentication` | `<Authentication name=alice, authenticated=true>` |
| `byte[]` (> 128 bytes) | `<byte[] length=4096>` |

### 2.10 Slow-only mode (Phase 7)

With `method.log-only-slow=true, slow-threshold-ms=100`:

A method completing in 10 ms produces NO output at all (no entry, no exit).
A method completing in 250 ms produces BOTH an entry line and an exit line at its proper indent — they appear together once the method exits, not when it began.
An exception (when `log-exceptions=true`) still emits regardless of duration.

This means that in slow-only mode, log lines may not arrive in real-time order — a slow inner call's lines are buffered until the call returns, then printed all at once when the threshold is crossed. The indent values used are the values that were correct at the moment the calls happened.

**Nesting rule (deferred-emission contract).** Each frame's entry+exit pair is buffered with the depth captured at *call entry* and emitted iff that frame's own elapsed time ≥ `slow-threshold-ms`. Frames are independent: a slow outer call whose inner calls were all fast emits only the outer pair, with its captured depth (e.g. depth 0). The inner pairs are simply absent from the log — there is no synthetic placeholder for them — and the outer's visual chain therefore has gaps where the fast children would have nested. Conversely, a fast outer with a slow inner emits only the inner pair, at the indent value captured when it began (e.g. depth 1), so the resulting line appears two-space-indented with no parent visible. This trades real-time fidelity for a budget-friendly default; adopters who need full chains should disable slow-only mode while debugging the specific call path.

**Overhead note for `method.log-only-slow=true` + `method.log-input=true`.** Buffering the entry line forces argument serialization at call entry on *every* invocation — even ones that turn out to be fast and never emit. `SafeLogSerializer` work is therefore not gated by the slow threshold. This is intentional (we cannot know elapsed time before the call runs) but adopters running `log-only-slow` for production triage should keep `log-input=false` unless argument visibility is essential. The SC-011 budget applies to the default config (`log-input=false`), so this combination is out of the budget's scope.

---

## 3. Determinism rules

The formatter MUST be deterministic on its inputs so tests can assert exact strings:

1. **Whitespace** — exactly one ASCII space between tokens. No tabs. Indent is exactly `"  "` × depth.
2. **No newlines mid-line** — the SLF4J message is a single line. The Logback / Log4j2 pattern is what adds the final newline.
3. **No locale-sensitive formatting** — `String.valueOf(int)` style. Durations are integer milliseconds with no decimal; `Locale.ROOT` is used wherever a `String.format` is involved (not expected to be common).
4. **Argument separator** — `, ` (comma + single space) between argument summaries in `input=[...]`.
5. **Field separator** — `, ` (comma + single space) between fields in DTO summaries (`Order{a=1, b=2}`).
6. **Field syntax** — `name=value`. Names are the Java field name (or record component name) unchanged.
7. **No coloring / ANSI escapes** — the library emits plain text; coloring is the user's appender concern.
8. **Stable field order** — fields are rendered in the order returned by `Class.getDeclaredFields()`. (Implementations of `Field[]` ordering are JVM-implementation-defined but stable per JVM run. PRD does not require cross-JVM stability.) Map entries follow the iteration order of the map (so a `LinkedHashMap` renders in insertion order; a `HashMap` order is undefined but stable within a run).

---

## 4. Trace-ID prefix rules (Phase 5+)

- The prefix `[traceId=<value>] ` (with one trailing space) is added IFF `MDC.get(trace-id.mdc-key)` returns a non-blank value at the moment of formatting.
- The prefix appears on EVERY log line emitted while MDC has the value, including HTTP entry/exit lines, method entry/exit lines, exception lines, and internal warnings.
- The prefix is at column 0 — it precedes the indent.
- If MDC is unset (Phase 1–4, or non-HTTP code paths, or `trace-id.enabled=false`), the prefix is omitted entirely (no `[traceId=]`, no `[traceId=null]`).
- Two concurrent requests on different threads each have their own MDC state; their lines interleave in the output but the trace IDs disambiguate (SC-005).

---

## 5. HTTP envelope rules (Phase 5+)

- HTTP entry line is emitted before any method-chain entry for the request.
- HTTP exit line is emitted after every method-chain exit for the request.
- Both HTTP lines render at depth 0 (no indent).
- HTTP entry line includes `method` and `uri` and optionally `?query`. Query string is omitted when null/empty.
- HTTP exit line includes `status=<code>` and `took=<ms>ms`.

When `http.log-headers=true` (Phase 6):

```text
HTTP → POST /orders headers={Content-Type=application/json, Authorization=****, X-Request-Id=abc123}
```

When `http.log-request-body=true` (Phase 6):

```text
HTTP → POST /orders body={"productId":10,"quantity":2,"password":"****"}
```

The body segment is truncated to `http.max-body-length`. Binary/multipart bodies render as `body=<binary, N bytes>` / `body=<multipart, N parts>`.

Headers are formatted as a single-line `{name=value, name=value}` map. Order follows `HttpServletRequest.getHeaderNames()` enumeration order (servlet-container-defined but stable within a request).

---

## 6. Exception line vs. exit line

When an intercepted method throws:

- The matching `←` exit line is NOT emitted.
- A `×` exception line IS emitted at the same indent the `←` would have used.
- `LoggingContext.pop()` still runs (in the aspect's `finally`) so subsequent calls on the same thread indent correctly.
- The original exception is rethrown unchanged (FR-018, Constitution Principle I).

If `method.log-exceptions=false`:

- No `×` line is emitted.
- No `←` line is emitted.
- `LoggingContext` still pops correctly.
- The exception is still rethrown unchanged.

---

## 7. Internal warning lines

Library-internal warnings emit on the same logger at `WARN`:

```text
[debug-trace WARN] Failed to serialize argument of type WeirdQuery; using fallback placeholder
[debug-trace WARN] Invalid regex in excluded-class-name-patterns: "(unterminated"; entry skipped
[debug-trace WARN] base-packages is empty while enabled=true; no method logs will be emitted
```

The leading `[debug-trace WARN]` tag is illustrative — the actual decoration depends on the user's SLF4J pattern. The IMPORTANT contract is: every internal message goes through the same logger as the trace lines (Constitution Principle VII / R11).

**Startup `base-packages` warning.** The third example above is documented behavior: when the library binds `spring-debug-trace.enabled=true` but `spring-debug-trace.base-packages` is empty (or all entries are blank after trimming), the auto-configuration emits the line *exactly once* at startup. The application still starts and no method-chain logs are produced (matching User Story 1 acceptance scenario 3). The warning is for the developer who flipped `enabled=true` and is wondering why nothing appears — it never re-fires per-request.

---

## 8. Backward compatibility commitment

The log format is a documented part of v1's user contract (PRD §15). Once v1.0.0 ships, the visual shapes in §2.1 (the canonical PRD §8 example), §2.2, and §5 of this document are stable:

- The arrow characters (`→`, `←`, `×`) MUST NOT change without a major version bump.
- The token order in each line MUST NOT change without a major version bump.
- The indent unit (2 spaces) MAY change to 4 spaces only with a major version bump.
- The trace-prefix shape `[traceId=...]` MUST NOT change without a major version bump.
- The placeholder text `<serialization-failed: ...>` MUST NOT change without a major version bump.

Adding NEW segments to existing lines (e.g., a future `thread=...` segment) is a minor-version compatible change provided existing parsers tolerating ignored tail tokens still work.

A future JSON format (PRD §22) will be opt-in via a `format` property *introduced at that time*. v1 does NOT expose the `format` property — adding it now would be scaffolding for a non-goal (Constitution Principle III). Adopters who need structured output today should write a Logback / Log4j2 layout that JSON-encodes the dedicated logger's events.
