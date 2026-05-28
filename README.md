# spring-debug-trace-starter

Automatic nested method-chain debug logging for Spring Boot 3 applications. Add the dependency, configure your base package, and see controller → service → repository execution flow with method inputs, outputs, exceptions, duration, and HTTP request context — no annotations required.

> **Status: in development.** This library is being built phase-by-phase per [`PRD-phases.md`](./PRD-phases.md). It is **not yet published** to Maven Central. The install snippets below show the intended coordinates for v1.0.0.

---

## What it does

Built for local development, staging diagnostics, and understanding unfamiliar Spring Boot codebases. The starter is:

- **Dependency-first** — no code changes, no annotations required
- **Spring Boot 3 native** — uses standard auto-configuration
- **Human-readable** — pretty indented logs with arrows and durations
- **Safe by default** — sensitive field masking on, HTTP bodies off, library disabled until you opt in

The core promise:

> Add dependency. Configure base package. See your Spring bean method chain.

## Requirements

- Java 17+
- Spring Boot 3.x
- Spring MVC (for HTTP request/response features; method logging works without it)

## Install (planned coordinates)

**Maven:**

```xml
<dependency>
    <groupId>io.github.cerovskimatija</groupId>
    <artifactId>spring-debug-trace-starter</artifactId>
    <version>1.0.0</version>
</dependency>
```

**Gradle:**

```kotlin
implementation("io.github.cerovskimatija:spring-debug-trace-starter:1.0.0")
```

## Minimal setup

`application.yml`:

```yaml
spring-debug-trace:
  enabled: true
  base-packages:
    - com.example

logging:
  level:
    io.github.cerovskimatija.debugtrace: DEBUG
```

That's it — restart and the library will trace methods inside `com.example.*`.

## Example output

```
[traceId=abc123] HTTP → POST /orders
[traceId=abc123] → OrderController.createOrder(input=[CreateOrderRequest{productId=10, quantity=2}])
[traceId=abc123]   → OrderService.createOrder(input=[CreateOrderRequest{productId=10, quantity=2}])
[traceId=abc123]     → PaymentService.charge(input=[PaymentRequest{amount=49.99}])
[traceId=abc123]     ← PaymentService.charge(output=PaymentResult{status=APPROVED}) took=42ms
[traceId=abc123]   ← OrderService.createOrder(output=Order{id=1001}) took=88ms
[traceId=abc123] ← OrderController.createOrder(output=ResponseEntity{status=201}) took=104ms
[traceId=abc123] HTTP ← POST /orders status=201 took=120ms
```

Exception path:

```
[traceId=abc123] → PaymentService.charge(input=[PaymentRequest{amount=49.99}])
[traceId=abc123] × PaymentService.charge threw PaymentDeclinedException(message=Card declined) after 21ms
```

## Configuration reference

See [`PRD.md` §10](./PRD.md) for the full property tree. The most-used options:

| Property | Default | Purpose |
| --- | --- | --- |
| `spring-debug-trace.enabled` | `false` | Master switch |
| `spring-debug-trace.base-packages` | `[]` | Packages to intercept |
| `spring-debug-trace.excluded-packages` | Spring/Hibernate/Jackson/… | Packages to skip |
| `spring-debug-trace.method.log-input` | `true` | Log argument values |
| `spring-debug-trace.method.log-output` | `true` | Log return values |
| `spring-debug-trace.method.log-exceptions` | `true` | Log thrown exceptions |
| `spring-debug-trace.method.log-only-slow` | `false` | Only log methods over the threshold |
| `spring-debug-trace.method.slow-threshold-ms` | `500` | Threshold for slow-only mode |
| `spring-debug-trace.http.enabled` | `true` | HTTP request/response metadata |
| `spring-debug-trace.http.log-headers` | `false` | Include headers (masked) |
| `spring-debug-trace.http.log-request-body` | `false` | Include request body |
| `spring-debug-trace.http.log-response-body` | `false` | Include response body |
| `spring-debug-trace.masking.enabled` | `true` | Mask sensitive field values |
| `spring-debug-trace.masking.fields` | `password`, `token`, … | Field names to mask (appends to defaults) |

## Security warning

This library can log sensitive application data when configured broadly. Sensitive field masking is on by default, HTTP headers and bodies are off by default, and the library itself is off by default. **Do not enable full input/output/body logging in production without reviewing your data exposure risks.**

## Known Spring AOP limitations

These are inherent to proxy-based AOP, not bugs in this library:

- **Self-invocation** — `this.someMethod()` inside the same bean is not intercepted.
- **Private methods** — not intercepted.
- **Non-Spring beans** — only Spring-managed beans are intercepted.

## Roadmap

The v1 build is split into 9 incremental phases (0–8). Each phase produces a working, demoable chunk. See [`PRD-phases.md`](./PRD-phases.md) for the breakdown and current progress.

For the full product spec, see [`PRD.md`](./PRD.md).

## License

To be added before the v1.0.0 release.
