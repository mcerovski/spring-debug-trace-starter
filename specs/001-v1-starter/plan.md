# Implementation Plan: Spring Debug Trace Starter v1

**Branch**: `001-v1-starter` | **Date**: 2026-05-28 | **Spec**: [./spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-v1-starter/spec.md`

## Summary

Deliver v1 of `spring-debug-trace-starter`: a Spring Boot 3 auto-configuration library that emits nested, indented `→`/`←`/`×` debug logs of Spring-managed bean method calls via Spring AOP, plus optional HTTP request/response brackets, MDC trace-ID correlation, sensitive-data masking, and slow-only filtering. No annotations are required in user code; users add the dependency, set `spring-debug-trace.enabled=true` and a non-empty `base-packages`, and the library activates.

The technical approach follows `PRD-phases.md` phases 0–8: bootstrap the Maven module and auto-configuration wiring; layer the aspect, nesting context, and formatter; add the reflective safe serializer and skip-type detector; add exception resilience; add field-name masking; add the HTTP metadata filter and `TraceIdManager`; add HTTP header/body logging with content-caching wrappers; add slow-only mode, pattern-based exclusions, and a per-class match cache; finish with the sample app, README, unit + integration suite, and Maven Central publishing.

## Technical Context

**Language/Version**: Java 17 (baseline source/target). CI verifies on Java 21.

**Primary Dependencies**:
- Runtime (compile): `spring-boot-starter-aop`, `spring-boot-autoconfigure`, `slf4j-api`, `jackson-databind`.
- Optional (provided/marked `optional` in pom): `spring-web` — only used when the Servlet API is present on the host classpath.
- Annotation processor: `spring-boot-configuration-processor` for IDE metadata.
- Test: `spring-boot-starter-test` (brings JUnit Jupiter, AssertJ, Mockito, Spring Test).

**Storage**: N/A. The library is stateless across requests. Per-request state lives in thread-locals (`LoggingContext`) and MDC (`TraceIdManager`); both are cleaned at the end of the root call / request.

**Testing**:
- Unit tests: JUnit 5 (Jupiter, params).
- Integration tests for auto-configuration wiring: `ApplicationContextRunner` for conditional-bean assertions; `@SpringBootTest` for end-to-end controller-to-service flows.
- Compatibility matrix: Java 17 + Java 21, latest Spring Boot 3.x.

**Target Platform**: JVMs running Spring Boot 3.x applications on Linux, macOS, and Windows (Java 17 or 21). The starter itself is a library JAR consumed via Maven Central.

**Project Type**: Library (Spring Boot starter). Single Maven module per Constitution Principle IV and PRD §24 decision 11. Sample application lives in `examples/spring-boot-demo` as a separate Maven project, not a child module of the starter, so the published artifact stays self-contained.

**Performance Goals**:
- **SC-007**: With `enabled=false`, wall-clock of a 10 000-call tight loop on a representative bean method MUST be within 5 % of the baseline run without the library on the classpath at all.
- **SC-011**: With `enabled=true` in default configuration (masking on, `method.log-input=false`, `method.log-output=false`, HTTP off), library overhead on intercepted methods whose own runtime is ≥ 1 ms MUST be ≤ 5 % of that method's runtime. Methods running < 1 ms are intentionally out of scope for this target.
- Phase 7 owns the microbenchmark harness that produces these numbers.

**Constraints**:
- **Safety**: library MUST NOT alter user application behavior — same HTTP status, same response bytes, same exception propagation (FR-020, SC-003, SC-006, Constitution Principle I).
- **Interception**: Spring AOP via proxies only — no AspectJ LTW, no Java agent, no compile-time weaving (Principle V).
- **Self-exclusion**: `io.github.cerovskimatija.debugtrace.*` MUST always be excluded from matching to prevent recursive self-logging.
- **Default-exclude packages**: `org.springframework`, `org.hibernate`, `jakarta.servlet`, `javax.servlet`, `com.fasterxml.jackson`, `org.slf4j`, `ch.qos.logback`.
- **Safe-by-default**: `enabled=false`, masking on, HTTP header/body logging off, binary + multipart bodies skipped (Principle II).
- **Servlet API optional**: `HttpLoggingFilter` MUST register only when `jakarta.servlet.Filter` is on the classpath; method logging MUST work without it.
- **Format**: pretty (human-readable) only in v1; JSON format is post-v1 (PRD §22).
- **Logger name**: every library-emitted log line MUST go through SLF4J logger `io.github.cerovskimatija.debugtrace`; method/HTTP trace at `DEBUG`, internal warnings at `WARN`.

**Scale/Scope**:
- Production code: ~12 classes (autoconfigure, properties, aspect, matcher, context, serializer, skip-type detector, masker, formatter, trace-id manager, http filter, optional small support types).
- Tests: 6 named unit-test classes per PRD §20 + integration tests for the 8 acceptance demos.
- Sample app: 1 Maven project, ~6 Java files (application + controller + 3 services + DTOs) + 1 `application.yml`.
- Configuration surface: ~30 properties under the `spring-debug-trace.*` prefix (full tree in `contracts/configuration-properties.md`).

## Constitution Check

*GATE: Verified before Phase 0 and re-verified after Phase 1 design.* All seven principles are satisfied. No deviations requiring a Complexity Tracking entry.

| Principle | How v1 complies |
|---|---|
| **I. Safety First (non-negotiable)** | Aspect wraps `proceed()` in `try`/`finally` so depth decrements and root thread-local cleanup happen on both branches. Every call site of `SafeLogSerializer`, `SensitiveDataMasker`, `LogFormatter`, and SLF4J is wrapped with a defined fallback (`<serialization-failed: ClassName>`). Intercepted exceptions are rethrown unchanged. `HttpLoggingFilter` invokes `copyBodyToResponse()` so the client receives the unmodified response. |
| **II. Safe-By-Default Configuration** | `spring-debug-trace.enabled=false` by default — no aspect bean, no filter bean. `masking.enabled=true` with the PRD §10 default field list. `http.log-headers`, `http.log-request-body`, `http.log-response-body` all default to `false`. `http.skip-binary-content` and `http.skip-multipart-content` default to `true`. User-supplied `masking.fields` append to defaults; they do not replace them. |
| **III. Phase-Gated, PRD-Anchored Scope** | Every deliverable in this plan is mapped to a phase in `PRD-phases.md` and at least one FR in `PRD.md`. Non-goals (PRD §6 — WebFlux body, OpenTelemetry, AspectJ LTW, Java agent, async context propagation, JSON format, distributed tracing, query tracing) receive no scaffolding, configuration keys, or extension points. |
| **IV. Stable Public Surface & Fixed Identifiers** | Maven coordinates `io.github.cerovskimatija:spring-debug-trace-starter`, root package `io.github.cerovskimatija.debugtrace`, logger name `io.github.cerovskimatija.debugtrace`, property prefix `spring-debug-trace`, AutoConfiguration registration at `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Java 17 baseline / Spring Boot 3.x — locked in `pom.xml`. |
| **V. Spring AOP Only** | No AspectJ LTW, no agents, no bytecode instrumentation. `MethodLogMatcher` always excludes the library's own package. `HttpLoggingFilter` guarded by `@ConditionalOnClass(jakarta.servlet.Filter)`. Self-invocation and private methods are documented limitations in the README — no workaround code in the library. |
| **VI. Test Discipline** | Phase 8 lists all 6 named unit tests and the 7-item integration suite from PRD §20. CI matrix runs Java 17 and Java 21 against latest Spring Boot 3.x. Each earlier phase ships its acceptance demo as the integration assertion for that phase. |
| **VII. Idiomatic Spring Boot Conventions** | Constructor injection only; all DI fields `private final`. `DebugTraceProperties` uses `@ConfigurationProperties("spring-debug-trace")` with nested classes per area. Conditional registration with `@ConditionalOnProperty` / `@ConditionalOnClass` / `@ConditionalOnMissingBean`. SLF4J with parameterized messages (no string concatenation). Javadoc on all public and protected types and members. Package layout by feature (`aspect`, `properties`, `serializer`, `masking`, `http`, `trace`, `format`, `context`, `autoconfigure`). |

**Re-check after Phase 1 design**: No new violations introduced by the data model, contracts, or quickstart artifacts in this plan. The contracts merely capture decisions already locked in spec.md and the constitution.

## Project Structure

### Documentation (this feature)

```text
specs/001-v1-starter/
├── plan.md                              # This file (/speckit-plan output)
├── spec.md                              # Feature spec with FRs + acceptance criteria
├── research.md                          # Phase 0 — decision log
├── data-model.md                        # Phase 1 — ephemeral entities (trace event, depth context, masking rule set, …)
├── quickstart.md                        # Phase 1 — adopter walkthrough (dependency → 2 props → output)
├── contracts/
│   ├── configuration-properties.md      # Full `spring-debug-trace.*` property tree
│   ├── log-format.md                    # Visual contract: entry/exit/exception/HTTP brackets
│   └── public-api.md                    # User-facing Java surface (configuration types + logger name)
├── checklists/                          # (created by /speckit-checklist if requested)
└── tasks.md                             # Phase 2 — created later by /speckit-tasks
```

### Source Code (repository root)

```text
pom.xml                                                                # Phase 0
src/main/java/io/github/cerovskimatija/debugtrace/
├── autoconfigure/
│   └── DebugTraceAutoConfiguration.java                               # Phase 0 + grows through Phase 5/6/7
├── properties/
│   └── DebugTraceProperties.java                                      # Phase 0 (stub) → grows each phase
├── aspect/
│   ├── MethodLoggingAspect.java                                       # Phase 1; exceptions Phase 3; slow-only Phase 7
│   └── MethodLogMatcher.java                                          # Phase 1; regex patterns + cache Phase 7
├── context/
│   └── LoggingContext.java                                            # Phase 1
├── format/
│   └── LogFormatter.java                                              # Phase 1; trace-id prefix Phase 5
├── serializer/
│   ├── SafeLogSerializer.java                                         # Phase 2; masking integration Phase 4
│   └── SkipTypeDetector.java                                          # Phase 2
├── masking/
│   └── SensitiveDataMasker.java                                       # Phase 4; header masking Phase 6
├── trace/
│   └── TraceIdManager.java                                            # Phase 5
└── http/
    └── HttpLoggingFilter.java                                         # Phase 5; body wrappers Phase 6

src/main/resources/
└── META-INF/
    └── spring/
        └── org.springframework.boot.autoconfigure.AutoConfiguration.imports   # Phase 0

src/test/java/io/github/cerovskimatija/debugtrace/
├── aspect/                MethodLogMatcherTest                        # Phase 1
├── context/               LoggingContextTest                          # Phase 1
├── format/                LogFormatterTest                            # Phase 1
├── serializer/            SafeLogSerializerTest                       # Phase 2
├── masking/               SensitiveDataMaskerTest                     # Phase 4
├── trace/                 TraceIdManagerTest                          # Phase 5
└── autoconfigure/         DebugTraceAutoConfigurationIT (+ siblings)  # Phase 0 onward — ApplicationContextRunner + @SpringBootTest

examples/spring-boot-demo/                                             # Phase 8
├── pom.xml
├── src/main/java/.../OrdersDemoApplication.java
├── src/main/java/.../web/OrderController.java
├── src/main/java/.../service/OrderService.java
├── src/main/java/.../service/PaymentService.java
├── src/main/java/.../service/InventoryService.java
├── src/main/java/.../repository/OrderRepository.java
├── src/main/java/.../dto/{CreateOrderRequest, Order, LoginRequest}.java
└── src/main/resources/application.yml

README.md                  # Existing — expanded in Phase 8 with full configuration + troubleshooting
LICENSE                    # Phase 8 (Maven Central requirement)
```

**Structure Decision**: Single Maven module (per Constitution Principle IV and PRD §24 decision 11). Java packages follow the **by-feature** layout from Principle VII (`aspect`, `properties`, `serializer`, `masking`, `http`, `trace`, `format`, `context`, `autoconfigure`) so each PRD §11 component lives at a predictable address. The sample app `examples/spring-boot-demo` is a sibling Maven project (its own `pom.xml`), not a child module, so the published starter artifact has no demo coupling.

## Phase 0 — Outline & Research

**Status**: complete. All `NEEDS CLARIFICATION` items from spec.md have been resolved either in the Clarifications session (2026-05-28) or in PRD §24. The research log in `research.md` captures the decisions that affect implementation shape:

1. **Auto-configuration topology** — one root `DebugTraceAutoConfiguration` containing nested `@Configuration` classes per area, each gated by its own conditions. This keeps the AOP path independent of the Servlet path so method logging works without `spring-web`.
2. **Aspect activation gate** — `@ConditionalOnProperty(prefix = "spring-debug-trace", name = "enabled", havingValue = "true")` PLUS a runtime guard in `MethodLogMatcher` that returns "no match" when `base-packages` is empty. This makes the "did I forget to set base-packages?" case silent rather than noisy or broken.
3. **Reflective serializer (no Jackson dependency for traversal)** — clarified in spec.md. Jackson stays on the dependency list only because PRD §12 lists it and it may be useful for the JSON-body masking helper in Phase 6, but the object walk uses pure reflection so DTOs from user code do not need to be Jackson-friendly.
4. **Exception stack-trace handling** — clarified: `method.log-exception-stack-trace=false` by default; when set, the throwable is passed through SLF4J so the user's appender configuration prints it.
5. **JSON body masking strategy** — clarified: regex/string substitution on `"<key>"\s*:\s*"..."` patterns. Never a full JSON parse. Operates on the captured-for-logging copy, never on the response delivered to the client.
6. **Per-call overhead target** — clarified: ≤ 5 % on methods ≥ 1 ms in default config. Sub-ms methods are intentionally out of scope.
7. **Trace-ID resolution order** — existing MDC → configured request headers (`X-Request-Id`, `X-Correlation-Id`, `traceparent`) → generated UUID (if `generate-if-missing=true`). MDC cleanup removes only keys the library set.
8. **AspectJ proxy mode** — rely on Spring Boot's default (`spring.aop.proxy-target-class=true`); do not set `@EnableAspectJAutoProxy` manually.
9. **Sample app placement** — `examples/spring-boot-demo` as a sibling Maven project. Not in a multi-module reactor so the starter `pom.xml` remains the single published artifact.

**Output**: `research.md` (decision log) — committed alongside this plan.

## Phase 1 — Design & Contracts

**Prerequisites**: research.md complete.

**Entity definitions** (`data-model.md`): documents the ephemeral runtime entities listed in spec.md "Key Entities" — Trace event, HTTP envelope event, Call-chain context, Trace ID, Masking rule set, Match decision — with their fields, lifecycle, owner component, and concurrency semantics. Nothing persists; the model is essentially the data carried by `LoggingContext`, `TraceIdManager`, and the cache inside `MethodLogMatcher`.

**Interface contracts** (`contracts/`): the library exposes three external surfaces. Each gets a contract document so adopters and downstream tasks (and the v1.0.0 acceptance gate) have a single source of truth:

1. **`configuration-properties.md`** — the canonical `spring-debug-trace.*` property tree. Each property has type, default, scope (which phase introduces it), and behavior under invalid input. Acts as the schema for `DebugTraceProperties` and the input to the `spring-boot-configuration-processor` metadata.
2. **`log-format.md`** — the visual contract for emitted log lines: entry (`→`), exit (`←`), exception (`×`), HTTP brackets (`HTTP →` / `HTTP ←`), trace-ID prefix, indentation rule (two spaces per depth level), and `took=Nms` suffix. This is what SC-005 ("filter by trace ID and get a contiguous chain") and the acceptance demos in PRD-phases.md commit us to.
3. **`public-api.md`** — the Java-level surface adopters touch. In v1 that is essentially zero — there are no public extension points for users to subclass; the library is configured via properties only. The contract documents the constraints this puts on us: `DebugTraceProperties` is public (it must be, for Spring's binder), the autoconfiguration class is public (it must be, for the imports file), everything else is package-private or `final`. The dedicated logger name `io.github.cerovskimatija.debugtrace` is the only behavioral hook adopters use day-to-day (to route or silence the library's output through standard logging-framework configuration).

**Quickstart** (`quickstart.md`): adopter walkthrough mirroring PRD §8 — Maven coordinates, two YAML properties, sample request, expected output (success and exception variants). Doubles as the regression script for SC-001 ("install + first log in under 5 minutes") and as the seed for the Phase 8 README.

**Agent context update**: the SPECKIT marker block in `CLAUDE.md` is updated in this command run to point to `specs/001-v1-starter/plan.md` so future Claude Code sessions land here when checking "what's the current plan."

**Output**: `data-model.md`, `contracts/configuration-properties.md`, `contracts/log-format.md`, `contracts/public-api.md`, `quickstart.md`, updated `CLAUDE.md` SPECKIT block.

## Complexity Tracking

> No constitution violations to justify. This table is intentionally empty.

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|--------------------------------------|
| — | — | — |
