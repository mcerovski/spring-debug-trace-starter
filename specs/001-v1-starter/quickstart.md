# Quickstart — Spring Debug Trace Starter v1

**Audience**: a Spring Boot 3 developer adding the library to an existing application for the first time.

**Promise (PRD §26)**: *Add dependency. Configure base package. See your Spring bean method chain.* This page is the end-to-end walkthrough that delivers on that promise — it also doubles as the regression script for SC-001 ("install + first log under 5 minutes") and as the seed for the Phase 8 README.

This document targets v1.0.0. Replace `1.0.0` with the actual release version once published.

---

## 1. Prerequisites

- Java 17 (or Java 21).
- Spring Boot 3.x application — Spring Boot 2.x is not supported.
- Maven 3.8+ or Gradle 7+.
- Optional: `spring-boot-starter-web` (Spring MVC) — only needed if you want HTTP entry/exit lines or HTTP body logging.

---

## 2. Add the dependency

### Maven

```xml
<dependency>
    <groupId>io.github.cerovskimatija</groupId>
    <artifactId>spring-debug-trace-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

### Gradle (Groovy DSL)

```groovy
implementation 'io.github.cerovskimatija:spring-debug-trace-starter:1.0.0'
```

### Gradle (Kotlin DSL)

```kotlin
implementation("io.github.cerovskimatija:spring-debug-trace-starter:1.0.0")
```

No `@EnableXxx` annotation needs to be added to your `@SpringBootApplication`. No other Java code changes.

---

## 3. Minimum configuration (two properties)

Add to `application.yml`:

```yaml
spring-debug-trace:
  enabled: true
  base-packages:
    - com.example                # ← replace with YOUR application's root package
```

Or in `application.properties`:

```properties
spring-debug-trace.enabled=true
spring-debug-trace.base-packages[0]=com.example
```

Make sure your logging level is high enough for the dedicated logger:

```yaml
logging:
  level:
    io.github.cerovskimatija.debugtrace: DEBUG
```

(Or set the root level to `DEBUG` if that's how your team prefers to gate.)

---

## 4. Run the application and hit an endpoint

Start the app as usual (`./mvnw spring-boot:run`, `./gradlew bootRun`, or your IDE run config). Hit any endpoint that goes through your Spring beans, for example:

```bash
curl -sX POST http://localhost:8080/orders \
     -H 'Content-Type: application/json' \
     -d '{"productId":10,"quantity":2}'
```

---

## 5. Expected output

In the application console you should see something like:

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

Trace IDs differ per request. Indentation reflects the call depth. `took=Nms` is the elapsed time of that individual method.

If a method throws, the matching `←` line is replaced by an exception line at the same indent:

```text
[traceId=abc123] → PaymentService.charge(input=[PaymentRequest{amount=49.99}])
[traceId=abc123] × PaymentService.charge threw PaymentDeclinedException(message=Card declined) after 21ms
```

The exception is rethrown unchanged — your `@ControllerAdvice` and retry logic still run normally.

---

## 6. The four most commonly customized properties (SC-009)

Beyond the two-line setup, these are the knobs adopters reach for first:

```yaml
spring-debug-trace:
  enabled: true
  base-packages:
    - com.example

  masking:
    fields:                       # APPENDS to the default list (password, token, …)
      - creditCard
      - ssn

  http:
    log-request-body: true        # show JSON request body (truncated, sensitive keys masked)
    log-response-body: true       # show JSON response body (response delivery to client is unaffected)
    max-body-length: 2000

  method:
    log-only-slow: true           # emit only methods slower than the threshold
    slow-threshold-ms: 100
```

These four blocks cover the most common adopter needs identified in SC-009: extending the masking list, enabling body logging, and trimming verbosity for staging.

---

## 7. Verifying the install (SC-001 check)

This 5-step check confirms the install is working end-to-end:

1. **Aspect loaded?** Set `logging.level.org.springframework.boot.autoconfigure: DEBUG`, restart, and grep the startup log for `DebugTraceAutoConfiguration` — it should appear as a matched auto-configuration.
2. **Method log appears?** Hit any controller endpoint and check the console for at least one `→` line under your base package.
3. **Indentation correct?** A controller-to-service-to-service call produces three nested `→` lines and three nested `←` lines.
4. **Trace ID present?** When `spring-web` is on the classpath and `spring-debug-trace.http.enabled=true` (the default), every log line for one request shares one `[traceId=...]` prefix.
5. **Disabled mode silent?** Set `spring-debug-trace.enabled=false`, restart, run the same request — no library log lines appear and the application behaves identically.

If any step fails, see Troubleshooting below.

---

## 8. Known limitations (Spring AOP)

These are accepted v1 limitations (Constitution Principle V, PRD §6, spec `Edge Cases`):

- **Self-invocation** — `this.someMethod()` inside the same bean bypasses Spring's proxy. The inner call is NOT logged. Workaround: inject the bean into itself or use a separate bean.
- **Private and non-public methods** — Spring AOP does not intercept these. Not a bug.
- **Async / `@Async` / `CompletableFuture` / message listeners** — call depth and trace ID do not propagate across executor handoffs. Each thread starts and ends clean.
- **WebFlux body logging** — out of scope in v1. Method-chain logging still works for `@Component`/`@Service` beans where Spring AOP applies.

---

## 9. Security warning

> This library can log sensitive application data if configured broadly. Use masking, package restrictions, and body logging options carefully. **Avoid enabling full input/output/body logging in production** unless you have reviewed your data exposure risks.

Default safety net (per Constitution Principle II):

- `enabled` is OFF by default — opt-in is explicit.
- Field-name masking is ON by default, covering `password`, `pass`, `token`, `accessToken`, `refreshToken`, `authorization`, `cookie`, `secret`, `apiKey`, `cardNumber`, `iban`.
- HTTP headers and bodies are OFF by default.
- Binary and multipart bodies are skipped by default even when body logging is on.

---

## 10. Troubleshooting

**Q: I see nothing in the console even though `enabled=true`.**

- Verify `base-packages` is non-empty and matches your actual application's root package (not the library's). The matcher does prefix matching, so `com.example` matches `com.example.web.OrderController` but `com.example.api` does NOT match `com.example.web.OrderController`.
- Verify the SLF4J level for `io.github.cerovskimatija.debugtrace` is at least `DEBUG`.
- Verify the bean you expect to be logged is a real Spring bean (`@Component`, `@Service`, `@RestController`, …) and the method is `public`.

**Q: An internal method call is not logged.**

That's the self-invocation limitation from §8. Move the call to a separate bean.

**Q: Why is my response body empty after enabling HTTP body logging?**

It should not be. The library wraps the response in `ContentCachingResponseWrapper` and explicitly invokes `copyBodyToResponse()` so the client receives the full body byte-for-byte. If you see an empty body, file a bug — this is a correctness regression (SC-006).

**Q: How do I reduce log noise without disabling the library?**

- Set `method.log-only-slow=true` and pick a threshold.
- Add `excluded-class-name-patterns` (e.g., `.*Configuration`, `.*Properties` are excluded by default; add `.*Mapper` if you don't want MyBatis mappers).
- Set `method.include-repositories=false` to skip Spring Data repositories.

**Q: I want to mask a domain-specific field name.**

Add it to `masking.fields`. It APPENDS to the default list — defaults stay masked.

---

## 11. Where to go next

- Full property reference: `specs/001-v1-starter/contracts/configuration-properties.md` (the Phase 8 README will mirror this).
- Log format details: `specs/001-v1-starter/contracts/log-format.md`.
- Sample application: `examples/spring-boot-demo` (created in Phase 8).
- Phase plan and acceptance demos: `PRD-phases.md`.
