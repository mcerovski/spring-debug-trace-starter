# Feature Specification: Spring Debug Trace Starter v1

**Feature Branch**: `001-v1-starter`

**Created**: 2026-05-28

**Status**: Draft

**Input**: User description: "v1 of spring-debug-trace-starter per PRD.md and PRD-phases.md"

## Clarifications

### Session 2026-05-28

- Q: Which engine should the serializer use for method arguments, return values, and DTO fields? → A: Pure reflection — walk fields/getters and apply masking, skip-type detection, and depth/size caps inline; Jackson Databind is not a hard runtime dependency.
- Q: Should exception log lines include the full stack trace? → A: Configurable, off by default — exception line shows class + message + duration only (matching PRD §8); a `log-exception-stack-trace` flag enables routing the throwable through SLF4J so the user's appender prints the full trace.
- Q: How should JSON body field masking be implemented when HTTP body logging is enabled? → A: Regex/string substitution on JSON-style key patterns — scan the captured body for `"<sensitive-key>"\s*:\s*"..."` shapes and replace the value, accepting rare false positives in free-text fields in exchange for never failing on malformed bodies and adding no JSON-parser dependency.
- Q: What is the enabled-mode per-call overhead commitment for v1? → A: Relative target keyed to ≥1ms methods — in default configuration (masking on, I/O logging off, HTTP off), library overhead is ≤5% on methods taking ≥1ms; microsecond-method overhead is intentionally unconstrained because slow-only mode filters them out of the realistic debug target.

## User Scenarios & Testing *(mandatory)*

The "user" of this library is a backend developer working on a Spring Boot 3+ application. Each story below is an independently shippable slice of value — a developer who installs the library and stops after any one story still has something useful in their console.

### User Story 1 - See the bean method chain with one dependency (Priority: P1)

A backend developer adds the starter as a Maven (or Gradle) dependency to an existing Spring Boot 3 service, sets two configuration values (an enable flag and the application's base package), and immediately sees indented, time-stamped logs of every Spring-managed bean method that runs during a request. No code changes, no annotations, no manual log statements.

**Why this priority**: This is the core promise of the product — *add dependency, configure base package, see your method chain.* Every other story builds on top of it. Without it, the library has no reason to exist.

**Independent Test**: Install the artifact in a minimal Spring Boot 3 sample app with a controller calling a service calling another service. Set the enable flag and base-package. Hit an endpoint. Verify the console shows entry/exit lines with arrow indicators, indentation that mirrors the call depth, and a duration suffix on each exit line.

**Acceptance Scenarios**:

1. **Given** a Spring Boot 3 app with the starter installed and `enabled=true` plus a base-package matching the app, **When** an HTTP request triggers a controller that calls a service that calls a downstream service, **Then** the developer sees three nested entry lines and three nested exit lines with duration on each exit.
2. **Given** the same setup but `enabled=false`, **When** any request runs, **Then** no method-chain log lines from the library appear and the application behaves identically to a baseline without the library.
3. **Given** the starter is installed but `base-packages` is empty, **When** any request runs, **Then** no method-chain logs are emitted (the developer must opt in by naming their packages). The library MUST also emit exactly one `WARN` line at application startup naming this combination, so the developer who flipped `enabled=true` but forgot `base-packages` is not left wondering why nothing appears.
4. **Given** a class in an excluded package (e.g., a Spring framework class, or the library's own package), **When** it is invoked, **Then** no log line is emitted for it — preventing recursive self-logging and framework noise.

---

### User Story 2 - See what each method received and returned (Priority: P2)

The developer enables input and output logging and now sees argument values on entry and return values on exit, written in a human-readable form, with awkward types (servlet objects, streams, uploads) summarized safely instead of dumped raw, and arbitrarily large or self-referential objects truncated rather than exploding the log.

**Why this priority**: Once the call chain is visible, knowing *what* flowed through it is the next biggest debugging win. This is what turns the library from "stack-trace lite" into a real diagnostic tool.

**Independent Test**: With Story 1 working, enable input/output logging and exercise endpoints with primitive args, DTO args, collection args, a multipart upload, and a method returning a self-referential object graph. Verify each call's logged input and output are readable, that the upload is summarized rather than dumped, and that the self-referential object terminates with a placeholder rather than recursing forever.

**Acceptance Scenarios**:

1. **Given** input and output logging are enabled, **When** a method receives a primitive, a string, an enum, a DTO, a list, and a map, **Then** all of these appear in the entry log line in readable form.
2. **Given** a method is passed a servlet request, a file upload, or an input stream, **When** it is logged, **Then** the argument appears as a short placeholder (e.g., type name + minimal identifying info) rather than the raw object.
3. **Given** a returned object contains a circular reference, **When** it is logged, **Then** logging completes in finite time and does not throw.
4. **Given** a returned string or collection exceeds the configured size limit, **When** it is logged, **Then** the log shows a truncated value with a clear indicator that truncation occurred.
5. **Given** output logging is disabled but input logging is enabled, **When** a method runs, **Then** the entry line shows arguments and the exit line shows duration only — no return value.

---

### User Story 3 - See exceptions inline, never break the application (Priority: P3)

When a user-code method throws, the developer sees the exception class, message, and elapsed time as part of the same indented call chain. The original exception keeps propagating to the application's existing error handlers exactly as it would without the library. If the library's own logging or serialization machinery hits a problem, it absorbs the failure and writes a safe placeholder — it never adds a new failure on top of the user's failure.

**Why this priority**: Exception paths are the most common reason developers turn on debug logging in the first place. They are also where "logging libraries that throw" cause the most damage. Resilience scaffolding has to land before any later feature can be trusted in a real codebase.

**Independent Test**: Throw a user-defined exception from a deeply nested service. Verify the log shows the failure in place with duration and class/message, and that the application's existing error handler still receives the original exception untouched. Then plant a DTO whose `toString` or a getter throws, log a method that returns it, and verify the log shows a serialization-failure placeholder and the application continues running.

**Acceptance Scenarios**:

1. **Given** input/output and exception logging are enabled, **When** a nested service method throws a custom exception, **Then** the log shows the exception type, message, elapsed time, and a distinct visual marker, and the parent call's exit line is replaced by an exception line at the right indentation.
2. **Given** an application has `@ControllerAdvice` translating exceptions to HTTP responses, **When** an intercepted method throws, **Then** the HTTP status code, response body, and any retry logic behave identically to a run without the library.
3. **Given** a logged object's serialization throws, **When** it is being formatted, **Then** the log shows a fallback placeholder (e.g., `<serialization-failed: ClassName>`) and the request completes normally.
4. **Given** an exception is thrown mid-chain, **When** the next request arrives on the same thread, **Then** indentation depth has been reset to zero and the new chain starts fresh.

---

### User Story 4 - Common secrets are hidden in logs by default (Priority: P4)

The developer can leave input/output logging on without leaking passwords, tokens, API keys, or card numbers into the console. Field-name matching is case-insensitive and covers a curated default list; teams can add their own field names, and the additions extend the defaults rather than replacing them.

**Why this priority**: Without masking, broader I/O logging is dangerous in any shared environment. Masking-on-by-default is what makes the library safe to recommend for staging and cautious-production use.

**Independent Test**: Send a login request whose DTO has `username` and `password` fields. Verify the password value is replaced by the configured placeholder while the username is visible. Add a custom field name (e.g., a domain-specific identifier) to the masking config; verify it is also masked in subsequent logs while the default fields stay masked.

**Acceptance Scenarios**:

1. **Given** masking is enabled (the default) and a DTO has a field named `password`, **When** it is logged, **Then** the value is replaced by the configured placeholder.
2. **Given** masking is configured with extra field names, **When** an object with both default-list and custom-list fields is logged, **Then** both are masked — the user-defined list extends the defaults, not replaces them.
3. **Given** field matching, **When** a field is named in any case (e.g., `Password`, `PASSWORD`, `password`), **Then** masking still applies.
4. **Given** masking is enabled, **When** a serialization error occurs while trying to mask a value, **Then** the library still produces a safe output and never accidentally emits the original unmasked value.

---

### User Story 5 - Correlate logs with HTTP context and a trace ID (Priority: P5)

For Spring MVC applications, the developer sees an opening HTTP line at the start of a request and a closing HTTP line at its end, both showing method, URI, status, and duration. Every log line — HTTP and method-chain — for that request shares a single trace ID, so a developer scanning a busy log can isolate one request's flow at a glance. If the upstream caller already supplied a correlation header, that value is used; otherwise the library generates one.

**Why this priority**: This is what turns a noisy per-thread log into a per-request story. It's also the foundation that future distributed-tracing integrations would plug into. HTTP brackets are cheap; trace IDs are the join key.

**Independent Test**: Hit an endpoint twice in quick succession from two clients. Verify each request's logs share a single trace ID, the two requests' trace IDs differ, and supplying an `X-Request-Id` header on a third request causes that exact value to appear in its logs instead of a generated one.

**Acceptance Scenarios**:

1. **Given** HTTP logging is enabled, **When** a request arrives, **Then** an HTTP entry line appears before the controller's entry line and an HTTP exit line appears after the controller's exit line, both labeled with the request method, URI, status (on exit), and duration (on exit).
2. **Given** trace-ID generation is enabled, **When** a request without a correlation header arrives, **Then** all log lines for that request — HTTP and method — carry the same generated trace ID.
3. **Given** a request arrives with a configured correlation header set, **When** trace IDs are read, **Then** the header's value is used as-is rather than a new one being generated.
4. **Given** the library generated the trace ID for a request, **When** the request completes, **Then** the trace-ID context is cleared so it does not leak into the next request handled by the same thread.
5. **Given** the application is a non-Servlet Spring Boot app (no Spring MVC on the classpath), **When** the starter is installed, **Then** method-chain logging still works and the HTTP filter is simply absent — no startup error.

---

### User Story 6 - Optionally see HTTP headers and bodies, safely (Priority: P6)

When debugging a wire-level issue, the developer can opt in to logging headers, request body, and response body. Sensitive headers (Authorization, Cookie, Set-Cookie, plus user-defined names) are masked. Bodies are truncated at a configured length, binary payloads are summarized rather than dumped, multipart uploads are noted by part count rather than streamed, and — critically — the real response body is still delivered to the client byte-for-byte.

**Why this priority**: Body logging is the most powerful and the most dangerous feature: handled wrong, it either leaks secrets or corrupts responses. It deserves its own slice so it can be hardened separately and is opt-in by design.

**Independent Test**: Enable request-body and response-body logging. POST a JSON body containing a `password` field. Verify the logged request body shows the JSON with `password` masked and truncated at the configured limit. Verify the HTTP client receives the full, unmodified response payload. Then send a binary upload and verify the log shows a binary placeholder rather than the bytes.

**Acceptance Scenarios**:

1. **Given** header logging is enabled, **When** a request is logged, **Then** sensitive headers (e.g., `Authorization`, `Cookie`, `Set-Cookie`) appear with their values masked.
2. **Given** request body logging is enabled, **When** a JSON body contains a sensitive field name, **Then** the logged body shows that field's value masked using a best-effort key-based replacement.
3. **Given** response body logging is enabled, **When** the response body would exceed the configured length, **Then** the logged body is truncated with a clear indicator and the actual response sent to the client is unmodified.
4. **Given** a request body is binary or multipart and binary/multipart skipping is enabled (the default), **When** logged, **Then** the body is replaced by a short summary (e.g., byte count or part count) and the upload still reaches the handler intact.
5. **Given** body logging is disabled (the default), **When** a request flows through, **Then** no body bytes appear in any log line.

---

### User Story 7 - Tune verbosity for staging and cautious production (Priority: P7)

The developer can switch the library into "slow-only" mode so only methods that exceeded a configured threshold are emitted, and can exclude classes by regex (e.g., configuration and properties classes) or category (e.g., repositories). When the library is disabled, its overhead is near zero — disabling at runtime is a credible incident-response action, not a meaningful performance hit on its own.

**Why this priority**: This is what makes the library safe to leave wired into staging and selectively useful in production. It lands last because each tunable needs the full set of decision points (matching, depth tracking, masking, HTTP brackets) to already be stable.

**Independent Test**: Enable slow-only mode with a 100ms threshold. Run a workload where one endpoint completes in ~10ms and another in ~250ms. Verify only the slow endpoint produces method-chain logs. Then run a tight 10,000-call benchmark on a method with the library `enabled=true` versus `enabled=false` and verify disabled-mode wall-clock is within a small constant of the baseline.

**Acceptance Scenarios**:

1. **Given** slow-only mode is enabled with a 100ms threshold and exception logging is enabled, **When** a method completes in 10ms, **Then** no log line appears for it; **When** another method completes in 250ms, **Then** an exit line appears with its duration; **When** any method throws, **Then** the exception line still appears regardless of duration.
2. **Given** the class-name pattern list excludes classes ending in `Configuration` and `Properties` — which is the out-of-the-box default; user-supplied patterns APPEND to it and do not replace it — **When** an excluded class's method runs, **Then** no log line is emitted for it. (An adopter who *wants* to see their `@ConfigurationProperties` beans in the chain must override the list explicitly.)
3. **Given** the repository toggle is set to exclude repositories, **When** a repository-pattern bean is invoked, **Then** it is not logged while service-tier beans still are.
4. **Given** the library is `enabled=false`, **When** a benchmark runs the same method many thousands of times, **Then** the measured overhead is within a small constant factor of the baseline-without-library run.

---

### Edge Cases

- **Self-invocation**: A bean that calls one of its own methods via `this.someMethod()` bypasses the Spring proxy and therefore the library cannot intercept the inner call. Documented as a known limitation; not a bug.
- **Private and non-public methods**: Standard Spring AOP cannot intercept these. Documented limitation; not a bug.
- **Async / `@Async` / `CompletableFuture` / message listeners**: Trace-ID and call-depth context is per-thread and is not propagated across executor or messaging boundaries. v1 accepts this; depth resets cleanly when execution returns to a clean thread.
- **WebFlux applications**: HTTP body logging is not supported on WebFlux in v1. Method-chain logging still works through Spring AOP for `@Component`/`@Service` beans, but body logging is out of scope.
- **Lazy-loaded entities (e.g., JPA proxies)**: Serialization is depth- and size-limited and must avoid triggering lazy loads where reasonable; if a lazy access throws, the library catches it and emits a placeholder.
- **Very deep call chains**: Indentation must remain correct, and stack-related failures (if any) in the library must not corrupt the indentation state for subsequent requests on the same thread.
- **High-cardinality argument types** (e.g., a list of 10,000 items): Collection logging is capped by configuration; only the first N items are shown with a truncation marker.
- **Non-Servlet runtimes**: When `spring-web` / servlet API is absent, the HTTP filter must not register and the library must still start. There must be no hard runtime dependency on servlets.
- **Concurrent requests**: Each request's call depth and trace ID must be isolated to its handling thread; one request's exception must not corrupt another's indentation.
- **Re-entrancy through the library**: If the library's own classes are ever called via Spring proxies, the matcher must exclude them so no recursive self-logging occurs.
- **Serialization throws during exception logging**: The exception line itself must still be emitted, with a safe placeholder substituting for the unprintable value, and the original user exception must still be rethrown unchanged.

## Requirements *(mandatory)*

### Functional Requirements

**Activation & scoping**

- **FR-001**: The library MUST be disabled by default, requiring the developer to explicitly turn it on through configuration.
- **FR-002**: When enabled, the library MUST only emit method-chain logs for classes whose package matches the developer-configured base-package list.
- **FR-003**: The library MUST always exclude its own package from interception to prevent recursive self-logging.
- **FR-004**: The library MUST exclude common framework packages (Spring core, Hibernate, Jackson, Servlet APIs, SLF4J, Logback) from method logging by default.
- **FR-005**: The library MUST allow developers to add further excluded packages and excluded class-name regex patterns.
- **FR-006**: The library MUST require no source-code annotations in user controllers, services, or other beans for method logging to take effect.

**Method-chain logging**

- **FR-007**: For each intercepted bean method, the library MUST emit an entry line on invocation and an exit line on normal return, both correlated to the same call.
- **FR-008**: The library MUST visually indent nested calls so the depth of nesting is unambiguous to a developer reading the console.
- **FR-009**: The library MUST measure and report each method's elapsed time on its exit (or exception) line, using a clock source not subject to wall-clock skew.
- **FR-010**: The library MUST keep per-thread call-depth state and MUST reset it cleanly at the end of a root call so subsequent calls on the same thread start at depth zero.
- **FR-011**: The library MUST handle nested call exceptions such that depth is decremented correctly on the exception path and indentation remains consistent for subsequent unrelated calls.

**Inputs and outputs**

- **FR-012**: The library MUST optionally log method input argument values when the corresponding configuration flag is enabled, and MUST omit them otherwise.
- **FR-013**: The library MUST optionally log method return values when the corresponding configuration flag is enabled, and MUST omit them otherwise (including the `void` and `null` cases).
- **FR-014**: The library MUST safely summarize unsupported or sensitive runtime types (e.g., servlet objects, streams, uploads, security principals, framework binding-result holders, large binary arrays) rather than serialize them in full.
- **FR-015**: The library MUST terminate cleanly when serializing self-referential or cyclic object graphs.
- **FR-016**: The library MUST honor configurable serialization limits on recursion depth, string length, collection size, and total per-object length, and MUST indicate when truncation has occurred.

**Exceptions and resilience**

- **FR-017**: The library MUST log exceptions thrown by intercepted methods, including exception type, message, and elapsed time, using a distinct visual marker. The full stack trace MUST NOT be emitted by default; a configuration flag MUST allow developers to opt in to stack-trace emission, which the library MUST route through the standard logging backend so it integrates with the user's existing appender configuration.
- **FR-018**: The library MUST rethrow the original exception unchanged — never wrapping, swallowing, or substituting it.
- **FR-019**: The library MUST never propagate its own failures (serialization, masking, formatting, or logging errors) into user code; such failures MUST be replaced by safe placeholder text in the log output.
- **FR-020**: When the library is enabled, the application's externally observable behavior — HTTP status codes, response bodies, exception flow to `@ControllerAdvice`, retry logic — MUST be identical to running the same application without the library.

**Sensitive-data masking**

- **FR-021**: The library MUST mask sensitive fields by default using a built-in field-name list covering common credential, token, secret, payment-card, and bank-identifier names.
- **FR-022**: The library MUST allow developers to add their own field names to be masked; the developer's additions MUST extend the default list, not replace it.
- **FR-023**: Field-name and header-name matching for masking MUST be case-insensitive.
- **FR-024**: The library MUST allow developers to configure the replacement placeholder text used for masked values.

**HTTP request/response context (Spring MVC)**

- **FR-025**: When the Servlet API is on the classpath and HTTP logging is enabled, the library MUST emit an HTTP entry line at the start of each request and an HTTP exit line at the end, including HTTP method, request URI, response status (on exit), and elapsed time (on exit).
- **FR-026**: When the Servlet API is absent, the library MUST still start successfully and method-chain logging MUST still work; the HTTP filter MUST simply not register.
- **FR-027**: HTTP header logging MUST be off by default; when turned on, sensitive headers (e.g., authorization, cookie, set-cookie) MUST be masked using the same masking pipeline as field values.
- **FR-028**: HTTP request body logging MUST be off by default; when turned on, bodies MUST be truncated at a configured length and binary / multipart bodies MUST be summarized rather than dumped.
- **FR-029**: HTTP response body logging MUST be off by default; when turned on, the actual response delivered to the client MUST remain byte-for-byte unmodified by the library.
- **FR-030**: When a logged body is JSON-shaped and contains keys matching the masking list, the library MUST make a best-effort attempt to mask those values in the logged copy using string/regex substitution on JSON-style key patterns (not a full JSON parse). The substitution MUST operate on the captured-for-logging copy only and MUST NOT alter the response delivered to the client. The library MUST tolerate malformed bodies without failure; rare false positives in free-text fields are an accepted trade-off.

**Trace-ID correlation**

- **FR-031**: The library MUST attach a per-request trace ID to logs so all method-chain and HTTP lines for the same request share an identifier.
- **FR-032**: The library MUST prefer an existing trace ID found in the logging context, then fall back to a configurable list of request headers, then generate one if generation is enabled.
- **FR-033**: The library MUST clean up only the trace-ID state it itself put in place, so existing externally-managed trace IDs are not erased.

**Configurability and verbosity**

- **FR-034**: The library MUST be configurable entirely through standard Spring Boot configuration (no code changes required to toggle features).
- **FR-035**: The library MUST support a "slow-only" mode in which only methods exceeding a configurable duration threshold produce method-chain output, while exception logging continues to fire regardless of duration when exception logging is enabled.
- **FR-036**: The library MUST allow the developer to exclude repository-pattern beans as a class category, separately from regex-based exclusions.
- **FR-037**: When disabled, the library MUST add near-zero runtime overhead per call relative to a baseline without the library.

**Observability of the library itself**

- **FR-038**: All trace logs emitted by the library MUST go through a single, well-known logger name so developers can route or silence them as a group without touching application loggers.
- **FR-039**: Method-chain trace lines MUST be emitted at the conventional debug log level so they are easy to gate via standard logging-framework configuration.

### Key Entities

- **Trace event**: A single log line about a method call — either an entry, an exit, or an exception — carrying the method identity, the depth at which it occurred, the elapsed time (for exit/exception), and the active trace ID when one exists. Trace events are ephemeral; nothing is persisted.
- **HTTP envelope event**: A log line emitted at the start and end of an HTTP request, bracketing all trace events for that request, carrying method, URI, status (on exit), elapsed time (on exit), and the active trace ID.
- **Call-chain context**: Per-thread state tracking the current nesting depth so trace events can be indented correctly. Cleaned up at the end of a root call.
- **Trace ID**: A per-request correlation identifier, either inherited from upstream (logging context or request header) or generated, attached to every log line emitted while the request is in flight.
- **Masking rule set**: The combined default + developer-supplied list of field/header names whose values are replaced with a placeholder when logged.
- **Match decision**: For a given bean class, whether it falls in scope for logging based on base-package inclusion, excluded packages, excluded class-name patterns, and library self-exclusion. Computable once per class.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A developer, starting from an existing Spring Boot 3 application, can install the library and see their first nested method-chain log in under 5 minutes of work, performing no source-code changes other than configuration.
- **SC-002**: In an integration test exercising a request that traverses three nested Spring-managed beans, 100% of the in-scope method invocations appear in the log with correct entry/exit ordering, correct indentation, and a non-zero duration on each exit.
- **SC-003**: Across the full integration test suite for the v1 release, zero failures are caused by the library: every test passes both with the library disabled and with the library enabled, and every assertion on application behavior (HTTP status codes, response bodies, exception types reaching handlers) holds identically in both modes.
- **SC-004**: In a request whose payload contains any field on the default sensitive-field list, the unmasked value appears zero times in the emitted log output; the masking placeholder appears exactly where each such value would have been.
- **SC-005**: For two concurrent requests handled on different threads, the developer can filter all log lines for one request using only the trace ID and obtains a contiguous, correctly nested chain with no lines from the other request.
- **SC-006**: With response-body logging enabled, every byte of the actual response delivered to the HTTP client is identical to the bytes that would have been delivered without the library — verified by byte-comparison across a representative sample of endpoints.
- **SC-007**: With the library disabled (`enabled=false`), the wall-clock time to execute a tight loop of 10,000 invocations of a representative bean method is within 5% of the same loop run without the library on the classpath at all.
- **SC-008**: With slow-only mode enabled at a threshold of 100ms, methods completing in under the threshold contribute zero log lines and methods exceeding it contribute exactly one entry and one exit (or one exception) line each, in a controlled workload.
- **SC-009**: An adopter following only the README — without reading the PRD — can configure the four most commonly customized settings (base packages, sensitive-field additions, HTTP body logging, slow-only mode) on the first attempt, judged by whether the resulting configuration matches the README's documented schema.
- **SC-010**: When the library encounters a deliberately broken serializer target (an object whose accessor throws), the calling request still completes successfully and the corresponding log line shows a serialization-failure placeholder rather than a stack trace from inside the library.
- **SC-011**: With the library enabled in its default configuration (masking on, method input/output logging off, HTTP logging off), the per-invocation overhead added by the library on intercepted methods whose own runtime is at least 1 ms is no greater than 5% of that runtime, measured by averaging a controlled workload. Methods running faster than 1 ms are out of scope for this target.

## Assumptions

- **Runtime baseline**: Target applications run on Java 17 or newer and Spring Boot 3.x. Java 8/11 and Spring Boot 2.x are explicitly out of scope for v1.
- **Web stack**: HTTP features target Spring MVC (Servlet API). WebFlux request/response body logging is explicitly out of scope for v1; method logging may still apply where Spring AOP applies.
- **Interception mechanism**: Standard Spring AOP via proxies is the only interception strategy for v1. No AspectJ load-time weaving, no Java agent, no bytecode instrumentation. Known consequences — self-invocation and non-public methods are not intercepted — are accepted and documented, not worked around.
- **Async propagation**: v1 does not propagate trace-ID or call-depth context across `@Async`, `CompletableFuture`, custom executors, or message listeners. Each thread's state begins clean and ends clean.
- **Format**: v1 emits a single pretty / human-readable log format. A structured (e.g., JSON) format is a post-v1 enhancement.
- **Logging backend**: Output is routed through SLF4J; developers' choice of Logback or Log4j2 (via SLF4J) is supported. The library does not bundle or assume a specific backend implementation.
- **Serializer engine**: Value serialization (method arguments, return values, DTO fields) is performed by reflective traversal of the object graph, applying masking, skip-type detection, depth/length caps, and circular-reference detection at every node. Jackson Databind is NOT a hard runtime dependency of the library — it is declared with Maven `<optional>true</optional>` so it does not propagate transitively to adopters' classpaths; users who already pull it in for their own application code are unaffected, but it is not required for `spring-debug-trace` to function and the v1 codebase does not import it.
- **Module shape**: v1 ships as a single Maven module (artifact `io.github.cerovskimatija:spring-debug-trace-starter`). A core/starter split is deferred until project growth warrants it.
- **Default verbosity**: Method-chain logs are emitted at the debug log level under a single dedicated logger name. Library-internal warnings (e.g., a degraded serializer path) are emitted at the warn level under the same logger.
- **Repository inclusion**: Repository-pattern beans are included by default; developers who find their query layer too noisy can exclude them through the repository toggle.
- **Trace-ID generation**: When no upstream trace ID is provided and generation is enabled, the library produces a UUID-style identifier. The exact format is internal to the library; consumers only care that one request produces one consistent value across all its log lines.
- **Distribution**: The artifact is published in a way that allows a developer to add a standard Maven or Gradle dependency line and have it resolve from a public repository (Central or equivalent) — exact publishing route is an implementation concern, not a behavior the spec constrains.
- **Documentation**: A README and a runnable sample Spring Boot 3 application accompany the v1 release so an adopter can verify the example output by themselves before integrating into their own codebase.
- **Resolved open questions** (per PRD §24): disabled-by-default; opt-in `base-packages`; Spring AOP only; HTTP metadata yes, headers and bodies opt-in; masking on by default; pretty format only in v1; debug level by default; repositories included by default; single Maven module to start.
