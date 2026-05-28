# PRD Phases: Spring Debug Trace Starter

This document splits v1 of `spring-debug-trace-starter` into incremental phases. Each phase produces a working, demoable chunk that builds on the previous one. The order is chosen so the core promise — *"add dependency, see method chain"* — comes online as early as possible, then safety, HTTP, and polish layer on top.

After Phase 1 you always have a "show a friend" state of the library.

---

## Phase 0 — Project Bootstrap

**Goal:** Empty buildable Maven project with correct coordinates and the Spring Boot 3 baseline.

**Deliverables:**
- Maven `pom.xml` with `io.github.cerovskimatija:spring-debug-trace-starter`, Java 17 toolchain, Spring Boot 3 BOM
- Dependencies: `spring-boot-starter-aop`, `spring-boot-autoconfigure`, `slf4j-api`, `jackson-databind`; `spring-web` as `optional`
- Empty `DebugTraceAutoConfiguration` class
- Empty `DebugTraceProperties` bound to prefix `spring-debug-trace`
- `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` registration
- One Spring context test that loads the auto-config when the jar is on the classpath
- Dedicated logger name `io.github.cerovskimatija.debugtrace` reserved

**Acceptance demo:** A sample Spring Boot 3 app that includes the starter starts cleanly, shows `spring-debug-trace.*` in IDE config metadata, and exhibits no behavior change.

**Covers PRD sections:** FR-1, §12 Dependencies.

**Defers:** All aspect/filter logic.

---

## Phase 1 — Method Interception Skeleton

**Goal:** Spring bean methods in configured packages produce nested entry/exit logs with duration. No inputs/outputs yet.

**Deliverables:**
- Properties: `enabled`, `base-packages`, `excluded-packages` (with library self-exclusion + Spring/Hibernate/Jackson/Servlet/SLF4J/Logback defaults)
- `MethodLogMatcher` — package include/exclude logic
- `LoggingContext` — thread-local depth counter; cleans itself when root call returns
- `LogFormatter` — pretty format with `→`/`←` arrows, indentation per depth, `took=Nms` suffix
- `MethodLoggingAspect` registered only when `enabled=true` and `base-packages` is non-empty
- Duration measured with `System.nanoTime()`, logs emitted at `DEBUG`

**Acceptance demo:**
```
→ OrderController.createOrder()
  → OrderService.createOrder()
    → PaymentService.charge()
    ← PaymentService.charge() took=42ms
  ← OrderService.createOrder() took=88ms
← OrderController.createOrder() took=104ms
```

**Covers PRD sections:** FR-2, FR-3, FR-4, FR-5, FR-15 (pretty only).

**Defers:** Input/output values, exceptions, HTTP, masking, trace IDs.

---

## Phase 2 — Safe Serialization + I/O Logging

**Goal:** Method inputs and return values appear in logs without breaking the app on awkward types.

**Deliverables:**
- `SkipTypeDetector` — short-list of types to summarize: `HttpServletRequest`, `HttpServletResponse`, `InputStream`, `OutputStream`, `MultipartFile`, `File`, `Resource`, `BindingResult`, `Principal`, `Authentication`, large byte arrays
- `SafeLogSerializer` with config: `max-depth`, `max-string-length`, `max-collection-size`, `max-object-length`, `include-null-fields`
- Circular-reference detection via identity set during recursion
- Support for: primitives, enums, strings, DTOs (reflective walk), collections, maps, arrays, `null`, `void`
- Properties: `method.log-input`, `method.log-output`
- Aspect integrates serializer for arguments and return values

**Acceptance demo:** Phase 1 logs now include `(input=[...])` and `(output=...)`. A self-referential DTO logs in finite time. A `MultipartFile` argument shows as `<MultipartFile name=upload size=1234>` rather than dumping the stream.

**Covers PRD sections:** FR-6, FR-7, FR-13.

**Defers:** Masking, exceptions, HTTP.

---

## Phase 3 — Exception Logging + Error Resilience

**Goal:** Exceptions in user methods are logged with duration and rethrown unchanged. The library never breaks the application — not even when serialization fails.

**Deliverables:**
- Aspect handles `Throwable`: logs class + message + duration with `×` marker, rethrows the original throwable unchanged
- `method.log-exceptions` property
- All serialization, formatting, masking, and logging calls wrapped in try/catch with fallback string `<serialization-failed: SimpleClassName>`
- `LoggingContext` correctly decrements depth on the exception path
- Negative-test fixture: a DTO whose `toString()`/getter throws does not propagate

**Acceptance demo:**
```
→ PaymentService.charge(input=[PaymentRequest{amount=49.99}])
× PaymentService.charge threw PaymentDeclinedException(message=Card declined) after 21ms
```
Application behavior — status codes, `@ControllerAdvice`, retry logic — is identical to a run without the starter.

**Covers PRD sections:** FR-8, FR-13 (failure path), §17 Error Handling.

**Defers:** Masking, HTTP.

---

## Phase 4 — Sensitive Data Masking

**Goal:** Common credentials and secrets are hidden in logs by default.

**Deliverables:**
- `SensitiveDataMasker` with default field list: `password`, `pass`, `token`, `accessToken`, `refreshToken`, `authorization`, `cookie`, `secret`, `apiKey`, `cardNumber`, `iban`
- Case-insensitive matching
- Properties: `masking.enabled`, `masking.replacement`, `masking.fields` (user fields append to defaults — do not replace them)
- `SafeLogSerializer` consults the masker for each field name during object walk

**Acceptance demo:** `LoginRequest{username=alice, password=hunter2}` becomes `LoginRequest{username=alice, password=****}`. Adding `creditCard` to `masking.fields` masks that field across all subsequent logs.

**Covers PRD sections:** FR-14, §14 Security defaults.

**Defers:** Header masking (Phase 5/6), JSON body masking (Phase 6).

---

## Phase 5 — HTTP Metadata Filter + Trace ID

**Goal:** HTTP requests bracket method-chain logs, and every log line for one request shares a trace ID.

**Deliverables:**
- `TraceIdManager`: reads existing MDC, reads configured request headers (`X-Request-Id`, `X-Correlation-Id`, `traceparent`), generates UUID if missing and `generate-if-missing=true`; cleans up only the values it put in MDC
- `HttpLoggingFilter` registered with `@ConditionalOnClass(jakarta.servlet.Filter)` — present only when Servlet API is on the classpath
- HTTP start/end log lines with: method, URI, query string, response status, duration, trace ID
- `LogFormatter` prepends `[traceId=...]` when MDC contains the configured key
- Properties: `http.enabled`, `trace-id.enabled`, `trace-id.mdc-key`, `trace-id.generate-if-missing`, `trace-id.request-headers`

**Acceptance demo:**
```
[traceId=abc123] HTTP → POST /orders
[traceId=abc123] → OrderController.createOrder(...)
[traceId=abc123]   → OrderService.createOrder(...)
[traceId=abc123]   ← OrderService.createOrder(...) took=88ms
[traceId=abc123] ← OrderController.createOrder(...) took=104ms
[traceId=abc123] HTTP ← POST /orders status=201 took=120ms
```

**Covers PRD sections:** FR-9, FR-12.

**Defers:** Header and body content.

---

## Phase 6 — HTTP Header + Body Logging

**Goal:** Optional full HTTP payload visibility, safely.

**Deliverables:**
- Content-caching wrappers (`ContentCachingRequestWrapper`/`ContentCachingResponseWrapper`) with explicit `copyBodyToResponse()` at the end so the real response is still delivered byte-for-byte
- `http.log-headers` with sensitive-header masking through `SensitiveDataMasker` (covers `Authorization`, `Cookie`, `Set-Cookie`, plus user-defined names)
- `http.log-request-body`, `http.log-response-body`, `http.max-body-length` (truncation)
- `http.skip-binary-content`, `http.skip-multipart-content` — replace body with `<binary, N bytes>` / `<multipart, N parts>`
- Best-effort JSON-body field masking via the existing masker (string-replace on JSON keys, not full parse)

**Acceptance demo:** Body logging on, hit `POST /orders` with a JSON body containing `password` — logs show the JSON truncated to `max-body-length` with `"password":"****"`. The client still receives the full unmodified response.

**Covers PRD sections:** FR-10, FR-11.

**Defers:** WebFlux, multipart parts inspection, JSON-streaming parsers.

---

## Phase 7 — Polish: Slow-Only, Patterns, Performance

**Goal:** Tunable enough for staging / cautious production use; disabled mode is near-zero cost.

**Deliverables:**
- `method.log-only-slow` + `method.slow-threshold-ms` — when slow-only, emit nothing until the method exits and only if it crossed the threshold (exceptions still log regardless if exception logging is on)
- `excluded-class-name-patterns` (regex) with defaults `.*Configuration`, `.*Properties`
- `method.include-repositories` toggle (default true)
- Cache layer: per-class match decision (`Map<Class<?>, MatchVerdict>`); per-class skip-type decision
- Short-circuit guard at the top of the aspect when `enabled=false` so the AOP advice returns in nanoseconds
- Microbenchmark harness (JMH-style or simple loop) checked into `examples/` to validate overhead claims

**Acceptance demo:** With `log-only-slow=true, slow-threshold-ms=100`, a 10ms method produces no output but a 250ms method does. Disabling the library at runtime via `enabled=false` and rerunning a 10k-call loop shows overhead near the baseline.

**Covers PRD sections:** FR-16, §15 Performance, §10 (`excluded-class-name-patterns`, `include-repositories`).

**Defers:** JSON format, per-class/per-method filters, sampling (PRD §22 future work).

---

## Phase 8 — Sample App, Documentation, Tests, Release

**Goal:** v1.0.0 is shippable to Maven Central and adoptable from the README alone.

**Deliverables:**
- `examples/spring-boot-demo`: controller, nested services, repository-like bean, exception endpoint, DTO with sensitive fields, full sample `application.yml`
- Sample endpoints: `POST /orders`, `GET /orders/{id}`, `GET /orders/fail`
- README sections: what it does, Spring Boot 3+ requirement, Maven + Gradle install, minimal setup, full configuration reference, example output (success + exception), known AOP limitations (self-invocation, private methods), security warning block, custom masking, enabling HTTP bodies, package exclusions, log-level setup, troubleshooting
- Unit tests: `MethodLogMatcherTest`, `SensitiveDataMaskerTest`, `SafeLogSerializerTest`, `LoggingContextTest`, `TraceIdManagerTest`, `LogFormatterTest`
- Integration tests: auto-config loads/skips per `enabled`, full controller-service chain logged, excluded packages silent, masking end-to-end, exceptions logged + rethrown, HTTP metadata, HTTP body logging without breaking response delivery
- CI matrix: Java 17 + Java 21, latest Spring Boot 3.x
- Maven Central publishing setup: GPG signing, `central-publishing-maven-plugin` (or OSSRH), source + javadoc jars, `LICENSE`

**Acceptance demo:** Cut a `1.0.0` tag. A fresh Spring Boot 3 app pulls the artifact from Central, sets two properties (`enabled=true`, `base-packages=...`), and produces the documented output without any other change.

**Covers PRD sections:** §18 Documentation, §19 Sample App, §20 Testing, §21 v1 Acceptance.

---

## Phase Ordering Rationale

- **0 → 1:** Get the auto-config wiring right before any behavior is attached — the hardest debugging is "why didn't my starter activate."
- **1 before 2:** Aspect + nesting are the load-bearing pieces. Get them clean and observable before stacking serialization on top.
- **3 before 4:** Resilience scaffolding has to exist before masking, because masking adds another place serialization can fail.
- **5 before 6:** HTTP metadata is cheap and useful immediately; bodies are the riskiest feature (response-delivery hazards) and benefit from the trace-id infrastructure already being in place.
- **7 last before release:** Caches and slow-only mode are easier to add once the full set of decision points is stable.

## What's Out of v1 (Per PRD §6, §22)

WebFlux, OpenTelemetry, async context propagation, JSON log format, AspectJ LTW, Java agent mode, actuator toggle, per-class/per-method filters, sampling, query timing, Micrometer integration, Kotlin coroutines. These are tracked for v1.1+ and should not be conflated with any v1 phase.
