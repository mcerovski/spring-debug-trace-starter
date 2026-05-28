# Contract: Public API

**Date**: 2026-05-28
**Branch**: `001-v1-starter`
**Scope**: The Java-level surface that adopters (and other Spring Boot starters) interact with in v1.

The library's user-facing contract is overwhelmingly the **configuration tree** (`contracts/configuration-properties.md`) and the **log format** (`contracts/log-format.md`). The Java surface is intentionally tiny: there are no public extension points users are expected to subclass or implement in v1. This document lists the few types that MUST stay public (because Spring's binder or `AutoConfiguration.imports` requires them), and explicitly states what is package-private or internal.

This contract exists so:

- Adding a public method to a library class triggers a deliberate review against this list.
- The Phase 8 release-gate Javadoc audit has a concrete checklist.
- Downstream library authors who want to embed `spring-debug-trace-starter` know which class names are stable.

---

## 1. Public types

These types MUST exist with these names and packages. Adding methods to them is a minor-version-compatible change; removing or renaming is a major-version-breaking change.

### 1.1 `io.github.cerovskimatija.debugtrace.autoconfigure.DebugTraceAutoConfiguration`

- **Why public**: Listed in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`. Spring Boot's auto-config loader needs to instantiate it reflectively.
- **Stability**: The class name is part of the public API. Nested `@Configuration` classes inside it (e.g. `MethodLoggingConfiguration`, `HttpLoggingConfiguration`) are NOT public API — they may be renamed or restructured between minor versions as long as the resulting bean wiring stays the same.
- **Bean contract**: At a minimum, when the autoconfig is active, it MUST register beans of these types as `@ConditionalOnMissingBean`, so user overrides win:
  - `DebugTraceProperties`
  - `MethodLogMatcher`
  - `LoggingContext`
  - `LogFormatter`
  - `MethodLoggingAspect`
  - (Phase 2+) `SkipTypeDetector`, `SafeLogSerializer`
  - (Phase 4+) `SensitiveDataMasker`
  - (Phase 5+) `TraceIdManager`, `HttpLoggingFilter` (latter conditional on `jakarta.servlet.Filter`)
- **Public methods**: none required for users to call. The class is purely a Spring meta-class.

### 1.2 `io.github.cerovskimatija.debugtrace.properties.DebugTraceProperties`

- **Why public**: Spring's `@ConfigurationProperties` binder must instantiate it via a public constructor and call public setters (or use the Java records / immutable variant — implementation detail).
- **Stability**: The bound property prefix `spring-debug-trace` is locked by Constitution Principle IV. The internal Java structure (nested classes `Method`, `Http`, `TraceId`, `Serialization`, `Masking`) is implementation detail — users never type these class names; they type the property keys. Renaming the nested Java classes does NOT break the property tree as long as the binder still produces the same key tree.
- **Bean access**: Users CAN inject `DebugTraceProperties` to read the live configuration, but this is not a recommended pattern. The supported way to react to config is to define one's own bean conditional on the same property keys.

### 1.3 The dedicated logger name `io.github.cerovskimatija.debugtrace`

- **Why "public"**: This is the SLF4J name adopters use in their `logback.xml` or `application.yml` to route or silence the library:

  ```yaml
  logging:
    level:
      io.github.cerovskimatija.debugtrace: DEBUG
  ```

- **Stability**: This name is part of the v1 contract (Constitution Principle IV). It MUST NOT change without a major version bump.
- **Not a Java type**: it's a string the user types in their logging config. Listed here because changes are equally breaking.

---

## 2. Package-private / internal types

These types exist but are NOT public API. They are package-private (or public-for-Spring-instantiation but undocumented for user code). User code MUST NOT depend on:

| Type | Notes |
|---|---|
| `MethodLoggingAspect` | Public for Spring to instantiate; not intended for user subclassing. |
| `MethodLogMatcher` | Same. |
| `LoggingContext` | Same. Users have no reason to read its depth from app code. |
| `LogFormatter` | Public for Spring; replacing it is a Phase-9+ extension story, not v1. |
| `SafeLogSerializer` | Internal — Phase 6+ might split this; users never see it. |
| `SkipTypeDetector` | Internal. |
| `SensitiveDataMasker` | Public for Spring; the field list is configured via `masking.fields`, not by subclassing. |
| `TraceIdManager` | Public for Spring; user code uses MDC directly if it wants to read the trace ID. |
| `HttpLoggingFilter` | Public for Spring's filter registration; not intended for user composition in v1. |

A user who wants to "extend" the library's behavior in v1 has two supported levers:
1. **Configuration** (the property tree).
2. **Bean override** — define their own `@Bean` of the same type with `@Primary` or rely on the `@ConditionalOnMissingBean` guard. This is a power-user escape hatch, not a maintained public extension point.

---

## 3. No annotations exposed

The library exposes **zero** user-facing annotations in v1. This is by design (FR-006, PRD §3, README §25):

- No `@DebugTrace`, no `@LogMethod`, no `@TraceSensitive`. Method logging is package-based, not annotation-based.
- The library DOES use `@Aspect`, `@ConfigurationProperties`, `@ConditionalOnProperty`, etc. internally — those are Spring/AspectJ annotations on internal classes.
- The library does NOT scan for user-defined annotations.

If a future version adds annotations (PRD §22 future work), they would live in a new package, be backward-compatible additions, and not change the v1 "annotation-free" promise.

---

## 4. Spring Boot starter contract

Adopters consume the library as a standard Spring Boot starter. The contract:

1. **Maven coordinates** (locked): `io.github.cerovskimatija:spring-debug-trace-starter`.
2. **One dependency** is sufficient — no companion `spring-debug-trace-core` to also include in v1.
3. **No `@EnableXxx` annotation** required on the user's `@SpringBootApplication` — auto-config alone is enough (FR-1, FR-006).
4. **No bean conflicts** with `spring-boot-starter-web`, `spring-boot-starter-aop`, common JPA / JDBC starters, Spring Security, Micrometer, Spring Cloud Sleuth — verified by integration tests in Phase 8.
5. **No leaked transitive `provided` dependencies** — `spring-web` MUST be marked `<optional>true</optional>` in the published pom so it does not pull into apps that don't already have it.
6. **`spring-boot-configuration-processor`** runs at compile time only; `META-INF/spring-configuration-metadata.json` ships in the jar so IDE autocomplete works for the property tree.

---

## 5. SemVer expectations

The project follows semantic versioning. Concretely, for the surfaces in this document:

| Change | Version impact |
|---|---|
| Add a new property under `spring-debug-trace.*` with a safe default | MINOR |
| Add a public method to `DebugTraceAutoConfiguration` or `DebugTraceProperties` | MINOR |
| Add a new emitted log segment to an existing line | MINOR |
| Rename `DebugTraceAutoConfiguration` or `DebugTraceProperties` | MAJOR |
| Move `io.github.cerovskimatija.debugtrace.*` to a different root package | MAJOR |
| Change the dedicated logger name | MAJOR |
| Change the `spring-debug-trace.*` property prefix | MAJOR |
| Change the visual log shapes in `log-format.md` §2.1 / §2.2 / §5 | MAJOR |
| Flip a safe default (e.g., default `http.log-request-body` to `true`) | MAJOR |
| Remove a property | MAJOR |
| Change a property's type incompatibly | MAJOR |
| Internal refactor of `MethodLogMatcher`, `SafeLogSerializer`, etc. | PATCH (unless it changes observable log output, in which case MINOR or MAJOR per the matrix above) |
| Internal performance optimization with identical output | PATCH |

This matrix governs v1.x. Going from v1.x to v2 follows the constitution amendment procedure (Constitution § Governance / Amendment procedure).

---

## 6. Acceptance for this contract

This document is satisfied for v1.0.0 when:

- `DebugTraceAutoConfiguration` and `DebugTraceProperties` exist with the names and packages above.
- The `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file contains exactly one line: `io.github.cerovskimatija.debugtrace.autoconfigure.DebugTraceAutoConfiguration`.
- The dedicated logger name appears in the README and in the sample app's `application.yml`.
- The library has zero `public` annotation types in `io.github.cerovskimatija.debugtrace.*`.
- The pom marks `spring-web` as `<optional>true</optional>`.
- The integration test suite confirms a fresh Spring Boot 3 app with only the starter dependency and two YAML properties produces the documented Phase 1 output (the Phase 1 acceptance demo).
