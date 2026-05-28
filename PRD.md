# Product Requirements Document: Spring Debug Trace Starter

## 1. Product Name

**Spring Debug Trace Starter**

Working Maven artifact name:

```xml
<groupId>io.github.cerovskimatija</groupId>
<artifactId>spring-debug-trace-starter</artifactId>
```

Alternative names:

* `spring-method-chain-logger-starter`
* `spring-boot-debug-chain-logging-starter`
* `spring-bean-trace-starter`

---

## 2. Summary

Spring Debug Trace Starter is a Spring Boot 3+ auto-configuration library that provides application-wide debug logging for Spring-managed bean method calls.

The library allows developers to add a Maven dependency, configure their application base packages, and automatically receive nested method-chain logs containing execution duration, method input, method output, exceptions, and optional HTTP request/response context.

The main goal is to help developers debug complex Spring Boot applications without manually adding logging statements or annotations across their codebase.

---

## 3. Problem Statement

Debugging request flows in Spring Boot applications often requires manually placing logs across controllers, services, clients, repositories, and other beans.

This is repetitive, inconsistent, and easy to forget. Developers often need to understand:

* Which methods were called during a request
* In what order methods were called
* How long each method took
* What inputs each method received
* What outputs each method returned
* Where an exception originated
* What HTTP request triggered the flow
* What response was returned

Existing logging approaches usually require manual log statements, custom annotations, distributed tracing infrastructure, or heavier observability setups.

Spring Debug Trace Starter provides a lightweight, developer-friendly way to inspect Spring bean execution chains using Spring AOP and Spring Boot auto-configuration.

---

## 4. Target Users

### Primary Users

* Backend Java developers building Spring Boot applications
* Developers debugging complex service-layer flows
* Developers working on monoliths or modular Spring Boot applications
* Teams that want fast local/staging diagnostics without manually adding logs

### Secondary Users

* QA engineers investigating backend behavior
* Technical leads reviewing performance bottlenecks
* Developers maintaining unfamiliar Spring Boot codebases
* Open-source project maintainers who want easy debug visibility

---

## 5. Goals

### Product Goals

1. Allow users to add one dependency and quickly enable method-chain debug logging.
2. Support Spring Boot 3+ applications only.
3. Log Spring-managed bean method calls through Spring AOP.
4. Show nested method call chains with readable indentation.
5. Measure execution time for every intercepted method.
6. Log method input arguments and return values safely.
7. Log exceptions with execution duration.
8. Provide optional HTTP request/response logging for Spring MVC applications.
9. Use MDC trace IDs to correlate all logs from one request.
10. Provide strong safety defaults to prevent secret leakage, huge logs, and self-recursion.
11. Require no annotations in application code.
12. Be configurable through `application.yml` / `application.properties`.

---

## 6. Non-Goals

The v1 product will **not** support:

1. Spring Boot 2.x.
2. Java versions below Java 17.
3. WebFlux request/response body logging.
4. Full JVM bytecode instrumentation.
5. Private method interception.
6. Self-invocation interception, such as `this.someMethod()` inside the same bean.
7. Non-Spring-managed object tracing.
8. OpenTelemetry integration.
9. Distributed tracing across services.
10. Async context propagation for `@Async`, `CompletableFuture`, custom executors, or message listeners.
11. Production-grade APM replacement functionality.
12. Automatic database query tracing.

These may be considered for later versions.

---

## 7. Scope

### In Scope for v1

* Spring Boot 3+ auto-configuration
* Spring AOP method interception
* Package-based include/exclude matching
* Nested method-chain logs
* Method execution time logging
* Input argument logging
* Return value logging
* Exception logging
* Safe serialization
* Sensitive field masking
* HTTP request metadata logging
* Optional HTTP request body logging
* Optional HTTP response body logging
* MDC trace ID support
* Human-readable pretty log format
* Configuration properties
* Global enable/disable flag
* Clear README documentation
* Sample Spring Boot app

### Out of Scope for v1

* WebFlux support
* OpenTelemetry bridge
* AspectJ load-time weaving
* Java agent instrumentation
* Runtime UI/dashboard
* Remote log storage
* Database query analysis
* Security audit guarantees

---

## 8. User Experience

### Desired Setup Experience

A user should be able to install the package with Maven:

```xml
<dependency>
    <groupId>io.github.cerovskimatija</groupId>
    <artifactId>spring-debug-trace-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

Then enable it with configuration:

```yaml
spring-debug-trace:
  enabled: true
  base-packages:
    - com.example
```

No application code annotations should be required.

### Example Output

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

### Exception Output

```text
[traceId=abc123] → PaymentService.charge(input=[PaymentRequest{amount=49.99}])
[traceId=abc123] × PaymentService.charge threw PaymentDeclinedException(message=Card declined) after 21ms
```

---

## 9. Functional Requirements

### FR-1: Spring Boot 3+ Auto-Configuration

The library must provide Spring Boot 3-compatible auto-configuration.

Requirements:

* Use `@AutoConfiguration` or compatible Spring Boot 3 auto-config pattern.
* Register auto-configuration through:

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

* Automatically create beans when enabled.
* Avoid requiring `@EnableAspectJAutoProxy` in user applications if possible.
* Avoid requiring custom application annotations.

Acceptance Criteria:

* Adding the dependency and setting `spring-debug-trace.enabled=true` activates the library.
* The starter works in a standard Spring Boot 3 MVC application.
* The starter does not require code changes in user controllers/services.

---

### FR-2: Global Enable/Disable

The library must be disabled by default or require explicit configuration to avoid accidental noisy logging.

Configuration:

```yaml
spring-debug-trace:
  enabled: false
```

Acceptance Criteria:

* When disabled, no aspect or HTTP filter logging is performed.
* When enabled, configured method and HTTP logging features are active.

---

### FR-3: Package-Based Method Matching

The library must intercept Spring bean methods based on configured base packages.

Configuration:

```yaml
spring-debug-trace:
  enabled: true
  base-packages:
    - com.example
```

Requirements:

* Only methods whose declaring class matches configured base packages should be logged.
* Users must be able to define excluded packages.
* The library package itself must always be excluded.

Configuration:

```yaml
spring-debug-trace:
  excluded-packages:
    - com.example.config
    - com.example.generated
```

Default exclusions should include:

* The library’s own package
* `org.springframework`
* `org.hibernate`
* `jakarta.servlet`
* `javax.servlet`
* `com.fasterxml.jackson`
* `org.slf4j`
* `ch.qos.logback`

Acceptance Criteria:

* Beans inside included packages are logged.
* Beans outside included packages are not logged.
* Beans inside excluded packages are not logged.
* The logger does not log itself recursively.

---

### FR-4: Spring Bean Method Interception

The library must intercept Spring-managed bean methods using Spring AOP.

Requirements:

* Intercept public Spring bean methods invoked through Spring proxies.
* Log method entry.
* Log method exit.
* Log method exceptions.
* Measure execution duration using `System.nanoTime()`.

Acceptance Criteria:

* Controller-to-service calls are logged.
* Service-to-service calls are logged when they go through a Spring proxy.
* Exceptions are logged and rethrown unchanged.
* Method results are returned unchanged.

Known Limitation:

* Self-invocation and private methods are not supported with standard Spring AOP.

---

### FR-5: Nested Method Chain Logging

The library must display nested method calls using indentation.

Requirements:

* Use request/thread-local context to track call depth.
* Increment depth before proceeding into a method.
* Decrement depth after method completion or exception.
* Clean up thread-local state when the root call finishes.

Example:

```text
→ OrderController.createOrder(...)
  → OrderService.createOrder(...)
    → PaymentService.charge(...)
    ← PaymentService.charge(...) took=42ms
  ← OrderService.createOrder(...) took=88ms
← OrderController.createOrder(...) took=104ms
```

Acceptance Criteria:

* Nested method logs are visually indented.
* Indentation remains correct on successful execution.
* Indentation remains correct when exceptions occur.
* Thread-local context is cleaned after request completion.

---

### FR-6: Method Input Logging

The library must optionally log method input arguments.

Configuration:

```yaml
spring-debug-trace:
  method:
    log-input: true
```

Requirements:

* Log argument values safely.
* Skip dangerous or unsupported argument types.
* Apply max length and max depth limits.
* Apply sensitive data masking.

Acceptance Criteria:

* Primitive, enum, string, DTO, collection, and map arguments are logged.
* Sensitive fields are masked.
* Large objects are truncated.
* Unsupported objects are replaced with safe placeholders.

---

### FR-7: Method Output Logging

The library must optionally log method return values.

Configuration:

```yaml
spring-debug-trace:
  method:
    log-output: true
```

Requirements:

* Log return values safely.
* Support `void` methods.
* Support `null` return values.
* Apply serialization limits and masking.

Acceptance Criteria:

* Return values are logged when enabled.
* Return values are omitted when disabled.
* Large return values are truncated.
* Sensitive fields are masked.

---

### FR-8: Exception Logging

The library must log exceptions thrown by intercepted methods.

Configuration:

```yaml
spring-debug-trace:
  method:
    log-exceptions: true
```

Requirements:

* Log exception class.
* Log exception message.
* Log method duration before exception.
* Rethrow the original exception unchanged.
* Do not swallow or wrap exceptions.

Acceptance Criteria:

* Exceptions appear in logs with duration.
* Application exception behavior remains unchanged.

---

### FR-9: HTTP Request/Response Logging

The library must optionally log HTTP request and response metadata for Spring MVC applications.

Configuration:

```yaml
spring-debug-trace:
  http:
    enabled: true
```

Default logged metadata:

* HTTP method
* Request URI
* Query string
* Response status
* Duration
* Trace ID

Acceptance Criteria:

* HTTP request start is logged.
* HTTP response completion is logged.
* HTTP logs share the same trace ID as method logs.

---

### FR-10: Optional HTTP Header Logging

The library must optionally log HTTP request and response headers.

Configuration:

```yaml
spring-debug-trace:
  http:
    log-headers: false
```

Requirements:

* Disabled by default.
* Mask sensitive headers such as `Authorization`, `Cookie`, and `Set-Cookie`.

Acceptance Criteria:

* Headers are not logged by default.
* Headers are logged when enabled.
* Sensitive headers are masked.

---

### FR-11: Optional HTTP Body Logging

The library must optionally log HTTP request and response bodies.

Configuration:

```yaml
spring-debug-trace:
  http:
    log-request-body: false
    log-response-body: false
    max-body-length: 5000
```

Requirements:

* Disabled by default.
* Use content-caching request/response wrappers.
* Do not break response body delivery.
* Skip binary and multipart payloads by default.
* Truncate large bodies.
* Mask sensitive fields when possible.

Acceptance Criteria:

* Request body logging works when enabled.
* Response body logging works when enabled.
* Response body is still returned to the client correctly.
* Binary/multipart bodies are skipped or summarized.

---

### FR-12: Trace ID / Correlation ID

The library must support MDC trace IDs.

Configuration:

```yaml
spring-debug-trace:
  trace-id:
    enabled: true
    mdc-key: traceId
    generate-if-missing: true
    request-headers:
      - X-Request-Id
      - X-Correlation-Id
      - traceparent
```

Requirements:

* Reuse existing MDC trace ID if present.
* Read trace ID from configured request headers if present.
* Generate a trace ID if missing and enabled.
* Store trace ID in MDC under configured key.
* Clean up MDC after request completion if the library generated the value.

Acceptance Criteria:

* All logs during one request contain the same trace ID.
* Existing trace IDs are not overwritten unexpectedly.
* MDC is cleaned correctly after request completion.

---

### FR-13: Safe Serialization

The library must safely convert method arguments, return values, and optional HTTP bodies into loggable text.

Configuration:

```yaml
spring-debug-trace:
  serialization:
    max-depth: 3
    max-string-length: 1000
    max-collection-size: 20
    max-object-length: 5000
    include-null-fields: false
```

Requirements:

* Avoid infinite recursion.
* Avoid excessive log size.
* Handle circular references gracefully.
* Avoid triggering lazy-loading problems where possible.
* Skip unsupported types.
* Never throw serialization exceptions back into user code.

Types to skip or summarize:

* `HttpServletRequest`
* `HttpServletResponse`
* `InputStream`
* `OutputStream`
* `MultipartFile`
* `File`
* `Resource`
* `BindingResult`
* `Principal`
* `Authentication`
* Large binary arrays

Acceptance Criteria:

* Serialization failure does not break application execution.
* Circular object graphs do not cause infinite loops.
* Logs are truncated according to configured limits.
* Unsupported types produce safe placeholders.

---

### FR-14: Sensitive Data Masking

The library must mask sensitive fields and headers.

Configuration:

```yaml
spring-debug-trace:
  masking:
    enabled: true
    replacement: "****"
    fields:
      - password
      - pass
      - token
      - accessToken
      - refreshToken
      - authorization
      - cookie
      - secret
      - apiKey
      - cardNumber
      - iban
```

Requirements:

* Field matching should be case-insensitive.
* Header matching should be case-insensitive.
* Users can add custom sensitive field names.
* Masking should apply to method inputs, method outputs, headers, and JSON-like bodies where possible.

Acceptance Criteria:

* Sensitive fields are masked in logs.
* Sensitive headers are masked in logs.
* Users can customize masked field names.

---

### FR-15: Log Format

The library must support a human-readable format in v1.

Configuration:

```yaml
spring-debug-trace:
  format: pretty
```

Requirements:

* Pretty format must be readable in local console logs.
* Every log line should include method direction, method name, duration where applicable, and trace ID when available.

Future format:

```yaml
spring-debug-trace:
  format: json
```

JSON logging may be added after v1.

Acceptance Criteria:

* Pretty format is stable and documented.
* Logs are readable and consistently structured.

---

### FR-16: Slow Method Logging

The library must support logging only slow methods.

Configuration:

```yaml
spring-debug-trace:
  method:
    log-only-slow: false
    slow-threshold-ms: 500
```

Requirements:

* If `log-only-slow=true`, only methods exceeding the threshold should be logged.
* Exception logs should still be emitted if exception logging is enabled.

Acceptance Criteria:

* Fast methods are omitted when slow-only mode is enabled.
* Slow methods are logged with duration.
* Exceptions are logged according to exception settings.

---

## 10. Configuration Specification

Proposed full configuration:

```yaml
spring-debug-trace:
  enabled: false

  base-packages: []

  excluded-packages:
    - org.springframework
    - org.hibernate
    - jakarta.servlet
    - javax.servlet
    - com.fasterxml.jackson
    - org.slf4j
    - ch.qos.logback

  excluded-class-name-patterns:
    - ".*Configuration"
    - ".*Properties"

  method:
    enabled: true
    log-input: true
    log-output: true
    log-exceptions: true
    log-only-slow: false
    slow-threshold-ms: 500
    include-repositories: true

  http:
    enabled: true
    log-headers: false
    log-request-body: false
    log-response-body: false
    max-body-length: 5000
    skip-binary-content: true
    skip-multipart-content: true

  trace-id:
    enabled: true
    mdc-key: traceId
    generate-if-missing: true
    request-headers:
      - X-Request-Id
      - X-Correlation-Id
      - traceparent

  serialization:
    max-depth: 3
    max-string-length: 1000
    max-collection-size: 20
    max-object-length: 5000
    include-null-fields: false

  masking:
    enabled: true
    replacement: "****"
    fields:
      - password
      - pass
      - token
      - accessToken
      - refreshToken
      - authorization
      - cookie
      - secret
      - apiKey
      - cardNumber
      - iban

  format: pretty
```

---

## 11. Technical Architecture

### Main Components

```text
spring-debug-trace-starter
├── DebugTraceAutoConfiguration
├── DebugTraceProperties
├── MethodLoggingAspect
├── HttpLoggingFilter
├── MethodLogMatcher
├── LoggingContext
├── TraceIdManager
├── SafeLogSerializer
├── SensitiveDataMasker
├── LogFormatter
└── SkipTypeDetector
```

### Component Responsibilities

#### DebugTraceAutoConfiguration

* Registers library beans.
* Applies conditional creation based on configuration and classpath.
* Enables Spring AOP support.

#### DebugTraceProperties

* Maps `spring-debug-trace.*` configuration.
* Defines defaults.
* Provides nested config classes for method, HTTP, trace ID, serialization, and masking.

#### MethodLoggingAspect

* Wraps Spring bean method execution.
* Measures duration.
* Logs entry, exit, and exception events.
* Uses `MethodLogMatcher` to decide whether to log a method.

#### HttpLoggingFilter

* Logs HTTP request/response metadata.
* Manages request-level trace ID.
* Optionally wraps request and response for body logging.

#### MethodLogMatcher

* Checks base packages.
* Checks excluded packages.
* Checks excluded class name patterns.
* Excludes library internals.

#### LoggingContext

* Tracks current call depth.
* Stores request/thread-local method chain state.
* Cleans up after root method or request completion.

#### TraceIdManager

* Reads existing MDC trace IDs.
* Extracts trace IDs from request headers.
* Generates new trace IDs.
* Cleans MDC safely.

#### SafeLogSerializer

* Converts Java objects into safe log strings.
* Applies max depth, size, and collection limits.
* Avoids circular reference issues.
* Delegates masking to `SensitiveDataMasker`.

#### SensitiveDataMasker

* Masks sensitive fields and headers.
* Supports case-insensitive matching.
* Supports user-defined field names.

#### LogFormatter

* Formats pretty log lines.
* Future extension point for JSON format.

---

## 12. Dependencies

### Required Dependencies

* Java 17+
* Spring Boot 3+
* Spring AOP
* SLF4J
* Jackson Databind

### Optional Dependencies

* Spring Web MVC for HTTP logging

The library should conditionally register HTTP logging only when Servlet MVC classes are available.

---

## 13. Compatibility Requirements

### Supported

* Java 17+
* Spring Boot 3.x
* Spring MVC applications
* Maven
* Gradle
* Logback via SLF4J
* Log4j2 via SLF4J

### Not Supported in v1

* Spring Boot 2.x
* Java 8 / 11
* WebFlux body logging
* Native-image compatibility guarantees
* Kotlin-specific features

---

## 14. Security and Privacy Requirements

The library must be designed with conservative defaults.

Requirements:

1. Disabled by default.
2. HTTP headers disabled by default.
3. HTTP bodies disabled by default.
4. Sensitive field masking enabled by default.
5. Sensitive headers masked when header logging is enabled.
6. Large values truncated.
7. Binary payloads skipped.
8. Multipart payloads skipped.
9. Serialization failures must not expose stack traces by default.
10. Documentation must warn against enabling full input/output/body logging in production without review.

README warning:

```text
This library can log sensitive application data if configured broadly.
Use masking, package restrictions, and body logging options carefully.
Avoid enabling full input/output/body logging in production unless you have reviewed your data exposure risks.
```

---

## 15. Performance Requirements

The library introduces overhead because it serializes arguments, return values, and logs method execution.

Requirements:

* Disabled mode should have minimal overhead.
* Method matching should be efficient.
* Serialization should respect max depth and max size limits.
* The library should avoid expensive serialization when input/output logging is disabled.
* The library should not perform reflection-heavy processing more than necessary.
* Exceptions from the logger must not affect application behavior.

Suggested optimization:

* Cache method/class matching decisions.
* Cache class skip-type decisions.
* Short-circuit as early as possible when logging is disabled or method is excluded.

---

## 16. Observability Requirements

The library should use a dedicated logger name:

```text
io.github.cerovskimatija.debugtrace
```

Recommended logger configuration:

```yaml
logging:
  level:
    io.github.cerovskimatija.debugtrace: DEBUG
```

Open question:

* Should method chain logs be emitted at `DEBUG` by default or configurable log level?

Recommended v1 default:

* Emit method and HTTP trace logs at `DEBUG`.
* Emit internal library warnings at `WARN`.

---

## 17. Error Handling Requirements

The logger must never break user application execution.

Requirements:

* Serialization errors must produce placeholders.
* Masking errors must produce safe fallback output.
* Logging errors must not be propagated.
* Exceptions from intercepted user methods must be rethrown unchanged.

Example fallback:

```text
<serialization-failed: OrderResponse>
```

---

## 18. Documentation Requirements

The README must include:

1. What the library does.
2. Spring Boot 3+ requirement.
3. Installation with Maven and Gradle.
4. Minimal setup.
5. Full configuration reference.
6. Example output.
7. Known Spring AOP limitations.
8. Security warning about logging sensitive data.
9. How to mask custom fields.
10. How to enable HTTP body logging.
11. How to exclude packages.
12. How to configure log level.
13. Troubleshooting section.

Troubleshooting topics:

* Why are my methods not logged?
* Why are internal method calls not logged?
* Why are private methods not logged?
* Why is the response body empty? Include `copyBodyToResponse()` implementation note internally.
* How do I reduce log noise?
* How do I disable output logging?

---

## 19. Sample Application Requirements

The repository should include a sample app:

```text
examples/spring-boot-demo
```

Sample app should include:

* Controller
* Service
* Nested service call
* Repository-like bean
* Exception endpoint
* DTO with sensitive fields
* Request/response example
* Example `application.yml`

Sample endpoints:

```text
POST /orders
GET /orders/{id}
GET /orders/fail
```

---

## 20. Testing Requirements

### Unit Tests

* `MethodLogMatcherTest`
* `SensitiveDataMaskerTest`
* `SafeLogSerializerTest`
* `LoggingContextTest`
* `TraceIdManagerTest`
* `LogFormatterTest`

### Integration Tests

* Auto-configuration loads when enabled.
* Auto-configuration does not log when disabled.
* Controller/service chain is logged.
* Excluded packages are not logged.
* Sensitive fields are masked.
* Exceptions are logged and rethrown.
* HTTP metadata logging works.
* HTTP body logging works and does not break responses.

### Compatibility Tests

* Spring Boot latest 3.x version.
* Java 17.
* Java 21.

---

## 21. Acceptance Criteria for v1 Release

v1 is ready when:

1. A Spring Boot 3 user can add the starter dependency and enable it with YAML.
2. No annotations are required in application code.
3. Method chains across Spring beans are logged with nesting.
4. Method duration is logged.
5. Method input/output logging works and is configurable.
6. Exceptions are logged and rethrown unchanged.
7. HTTP request/response metadata logging works.
8. Trace ID correlation works through MDC.
9. Sensitive fields are masked by default.
10. Large objects and bodies are truncated.
11. Unsupported object types are safely skipped.
12. Logger does not recursively log itself.
13. Package include/exclude configuration works.
14. The library is documented with examples and limitations.
15. A sample application demonstrates expected behavior.

---

## 22. Future Enhancements

Potential post-v1 features:

1. JSON structured log format.
2. WebFlux support.
3. Async context propagation.
4. OpenTelemetry span integration.
5. AspectJ load-time weaving option.
6. Java agent mode for deeper tracing.
7. Runtime actuator endpoint to toggle logging.
8. Per-package log configuration.
9. Per-class and per-method filters.
10. Regex-based method name include/exclude.
11. Log sampling.
12. Maximum chain depth configuration.
13. Repository query timing integration.
14. Micrometer metrics for slow methods.
15. Kotlin coroutine support.

---

## 23. Open Questions

1. Should the library be disabled by default, or enabled when `base-packages` is provided?
2. Should the default logger level be `DEBUG` or configurable?
3. Should repository beans be included by default?
4. Should method entry logs be emitted immediately, or should the full chain be emitted only after completion?
5. Should v1 support JSON logs, or reserve that for v1.1?
6. Should trace IDs use UUID, short UUID, or request header values only?
7. Should the project start as one module or separate core/starter modules?
8. Should HTTP logging be enabled by default when method logging is enabled?
9. Should the serializer use Jackson only, reflection only, or a hybrid approach?
10. What should the final artifact and package names be?

---

## 24. Recommended v1 Decisions

Recommended decisions for the initial implementation:

1. Support only Spring Boot 3+ and Java 17+.
2. Disable the library by default.
3. Require users to configure `base-packages`.
4. Use Spring AOP only.
5. Support Spring MVC HTTP metadata logging.
6. Disable HTTP headers and bodies by default.
7. Enable sensitive field masking by default.
8. Use pretty text logs only in v1.
9. Emit trace logs at `DEBUG` level.
10. Include repositories by default, but allow users to exclude them.
11. Implement a single Maven module first for speed.
12. Split into core/starter modules only if the project grows.

---

## 25. Example README Positioning

```text
Spring Debug Trace Starter gives Spring Boot 3 applications automatic nested method-chain logging for Spring-managed beans.

Add the dependency, configure your base package, and see controller-to-service-to-repository execution flow with method inputs, outputs, exceptions, duration, and HTTP request context.

Built for local development, debugging, staging diagnostics, and understanding unfamiliar Spring Boot codebases.
```

---

## 26. Key Differentiator

Unlike manual logging, annotation-based logging, or full observability stacks, Spring Debug Trace Starter is designed to be:

* Dependency-first
* Annotation-free
* Spring Boot native
* Human-readable
* Safe by default
* Useful immediately during local debugging

The core promise:

```text
Add dependency. Configure base package. See your Spring bean method chain.
```
