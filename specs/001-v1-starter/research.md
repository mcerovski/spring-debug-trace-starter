# Phase 0 Research — Spring Debug Trace Starter v1

**Date**: 2026-05-28
**Branch**: `001-v1-starter`
**Status**: Complete — no open `NEEDS CLARIFICATION` items remain.

This file records the implementation decisions taken before Phase 1 design. Most decisions are already locked by the Clarifications session in `spec.md` (2026-05-28) and by PRD §24. This document collects them, plus the small number of additional research items the implementation needs answered before writing code.

For each item: **Decision**, **Rationale**, and **Alternatives considered**.

---

## R1. Auto-configuration topology

**Decision**: One public root class `DebugTraceAutoConfiguration` in package `io.github.cerovskimatija.debugtrace.autoconfigure`. It owns `@EnableConfigurationProperties(DebugTraceProperties.class)` and groups nested `@Configuration(proxyBeanMethods = false)` classes by area:

- `MethodLoggingConfiguration` — registers `MethodLogMatcher`, `LoggingContext`, `LogFormatter`, `MethodLoggingAspect`. Gated by `@ConditionalOnProperty(prefix = "spring-debug-trace", name = "enabled", havingValue = "true")`. From Phase 2 on, also registers `SkipTypeDetector` and `SafeLogSerializer`. From Phase 4 on, also registers `SensitiveDataMasker`.
- `HttpLoggingConfiguration` — registers `TraceIdManager` and `HttpLoggingFilter`. Gated by `@ConditionalOnClass(jakarta.servlet.Filter.class)` AND `@ConditionalOnProperty(prefix = "spring-debug-trace", name = "enabled", havingValue = "true")`. Further gated on `http.enabled` / `trace-id.enabled` as appropriate.

All beans are `@ConditionalOnMissingBean` so user-provided overrides win.

**Rationale**: Splitting the HTTP-only beans into a nested class lets `@ConditionalOnClass` evaluate per-group rather than at the root level. If `spring-web` is missing, only `HttpLoggingConfiguration` is silently skipped — the method-logging path still loads. This matches FR-026 ("HTTP filter MUST simply not register") and the Phase 5 acceptance demo of working without `spring-web`.

**Alternatives considered**:
- *One flat autoconfig class with conditions on each `@Bean` method.* Works, but `@ConditionalOnClass` evaluated per-bean still loads the class containing the bean methods, which can trigger `NoClassDefFoundError` on JVMs where the Servlet API is absent. Nested `@Configuration` classes are the Spring-recommended workaround.
- *Two top-level autoconfig classes registered separately in `AutoConfiguration.imports`.* Equivalent in behavior, but one root class with nested groups reads more cleanly and produces one `DebugTraceAutoConfiguration` entry in the imports file. Single-entry registration is the Spring Boot convention for starters.

---

## R2. Aspect activation gate

**Decision**: `MethodLoggingAspect` is registered when `spring-debug-trace.enabled=true`. A second runtime check inside `MethodLogMatcher` returns `MatchVerdict.SKIP` for every class when `base-packages` is empty. The aspect bean exists but emits nothing.

**Rationale**:
- The Spring Boot idiom is `@ConditionalOnProperty(... havingValue="true")` to gate bean creation. Adding a second `@ConditionalOnExpression` to also require non-empty `base-packages` would require SpEL, which is harder to read and harder to test.
- The runtime-only guard in `MethodLogMatcher` keeps the failure mode quiet: a user who sets `enabled=true` and forgets `base-packages` sees no logs and no warnings, which matches the "Beans outside included packages are not logged" acceptance criterion (FR-3) and Story 1 acceptance scenario 3.
- The aspect's very first action in `proceed()` is `matcher.match(target.class)`; with empty packages this returns `SKIP` and the aspect short-circuits before any allocation, satisfying the disabled-mode overhead invariant (Principle II, FR-037).

**Alternatives considered**:
- *Hard-fail on empty `base-packages` when `enabled=true`.* Rejected — too aggressive for a library that promises "never break the user app." A warning-only log at startup is the safer message; Phase 1 will emit one `WARN` line from the autoconfig when this combination is detected.
- *Auto-derive `base-packages` from `@SpringBootApplication`'s base package.* Rejected — too magical; users would be surprised to discover their `org.example.config.*` classes were being logged just because their main class lives in `org.example`. Explicit opt-in matches PRD §24 decision 3.

---

## R3. Serializer engine

**Decision**: Pure reflection. `SafeLogSerializer` walks fields (and respects `Records` accessors) using `java.lang.reflect`, applying masking, skip-type detection, depth/length caps, and identity-set circular-reference detection at every node. Jackson Databind is NOT consulted for traversal in v1.

**Rationale**: This is the Clarifications session 2026-05-28 answer. Reflection-only traversal means:
- Users who do not pull `jackson-databind` themselves still get full functionality.
- DTOs without Jackson-friendly shapes (no public getters, sealed records, hand-rolled `toString` that throws) are still safe — we control every step.
- We never inadvertently trigger Jackson serializers/deserializers that might call into external systems or have side effects.

Jackson Databind is declared with `<optional>true</optional>` (Maven "optional" scope) — present on the compile classpath while the library builds, but NOT pulled transitively into the user's runtime when they depend on `spring-debug-trace-starter`. Adopters who already include Jackson (through `spring-boot-starter-web` or otherwise) are unaffected; adopters without it still get full functionality because the v1 codebase does not actually import Jackson anywhere. This honors spec.md Assumption §263 ("Jackson Databind is NOT a hard runtime dependency"). Phase 6's JSON body masking helper uses regex (decision R5) — Jackson's streaming parser was considered and rejected in R5, so the original "Phase 6 may need Jackson" hedge from earlier drafts no longer applies.

**Alternatives considered**:
- *Jackson `ObjectMapper.valueToTree(...)` then walk the JsonNode tree.* Faster on the happy path but exposes the library to every Jackson configuration choice the user has made and every custom serializer they've registered. Cyclic objects throw before the library can intercept. Rejected on resilience grounds.
- *Hybrid: try Jackson, fall back to reflection on failure.* Rejected — doubles the test matrix and produces two different log outputs for "the same" object depending on which path succeeded. Hard to specify the log-format contract under this scheme.

---

## R4. Exception stack-trace handling

**Decision**: Default `method.log-exception-stack-trace=false`. The exception line shows `× ClassName.method threw ExceptionType(message=...) after Nms` (matching PRD §8). When the flag is `true`, the library passes the `Throwable` as the last argument to `log.debug(format, throwable)` so SLF4J routes the stack trace through the user's configured appender.

**Rationale**: Clarifications session 2026-05-28 answer. Routing through SLF4J — rather than calling `printStackTrace()` or formatting the trace ourselves — means:
- Logback / Log4j2 configuration governs where the stack ends up (file vs. console).
- Pattern-based filtering and packaging-condensing the user already set up still work.
- The library does not become opinionated about stack-trace presentation.

**Alternatives considered**:
- *Always emit stack traces by default.* Rejected — too noisy for the "first 5 minutes" experience.
- *Always omit stack traces.* Rejected — debugging mid-chain failures sometimes requires the trace; making it opt-in is the safer middle.

---

## R5. JSON body masking strategy

**Decision**: Best-effort regex/string substitution on JSON-style key patterns. The library scans the captured-for-logging body copy for patterns of the form `"<key>"\s*:\s*"..."` (and the unquoted-numeric variant) and replaces the value portion with the configured masking replacement. The substitution operates only on the copy held for the log line; the bytes returned to the HTTP client are untouched.

**Rationale**: Clarifications session 2026-05-28 answer. Trade-offs:
- A full JSON parse would catch every case correctly but adds a hard dependency on a JSON parser, fails on malformed bodies (which we still want to log a snippet of), and is slower.
- Regex substitution accepts rare false positives (e.g., the word `"password"` inside a free-text field with a JSON-like neighbor) in exchange for never throwing.
- FR-030 explicitly mandates the best-effort, no-parse approach.

**Implementation note**: The same `SensitiveDataMasker` instance owns both field-name matching for DTOs and the JSON-body regex (Phase 6 adds the regex method); the field list is the single source of truth.

**Alternatives considered**:
- *Jackson streaming parser (`JsonParser`) with keys-only mode.* More accurate but couples body logging to Jackson and adds failure modes when Jackson can't parse the body.
- *Mask only when the `Content-Type: application/json` header is present.* Already implied — we only run the regex when the body is logged at all, and binary/multipart bodies are skipped by default.

---

## R6. Per-call overhead target

**Decision**: With default configuration (`enabled=true`, masking on, `method.log-input=false`, `method.log-output=false`, HTTP logging off), library overhead on intercepted methods whose own runtime is ≥ 1 ms MUST be ≤ 5 % of that runtime (SC-011). Sub-ms methods are intentionally out of scope for the budget — slow-only mode filters them out of the realistic debug target. With `enabled=false`, the 10 000-call loop wall-clock MUST be within 5 % of a no-library baseline (SC-007).

**Rationale**: Clarifications session 2026-05-28 answer. A relative budget (% of method runtime) is what users care about; an absolute microsecond budget would be either trivially achievable on slow methods or unachievable on fast ones. Phase 7 owns the microbenchmark harness that verifies both numbers.

**Alternatives considered**:
- *Absolute overhead in microseconds (e.g., "< 50 µs per call").* Rejected — meaningless on a 1 ms method and impossible on a 5 µs method.
- *No formal budget; "as fast as we can make it".* Rejected — Principle I demands a measurable resilience commitment, and the Phase 7 microbenchmark needs a target to assert against.

---

## R7. Trace-ID resolution order and MDC lifecycle

**Decision**: `TraceIdManager.resolve()` returns the first non-blank value from:

1. Existing MDC entry under `trace-id.mdc-key` (default `traceId`).
2. Each configured request header in order (`X-Request-Id`, `X-Correlation-Id`, `traceparent`).
3. Newly generated `UUID.randomUUID().toString()` if `trace-id.generate-if-missing=true`.
4. `null` otherwise (no trace ID — log lines simply omit the `[traceId=...]` prefix).

`TraceIdManager` records, per request, whether *it* placed the value in MDC. On cleanup it removes the key only if it placed it. Existing externally-managed MDC values are never erased.

**Rationale**: Matches FR-031, FR-032, FR-033 and Story 5 acceptance scenarios 2, 3, 4. Reading MDC first respects any upstream tracing library (Sleuth, Brave, Micrometer Tracing) that ran before the filter.

**Alternatives considered**:
- *Always generate a new ID and overwrite MDC.* Rejected — would clobber upstream-managed tracing.
- *Read header first, MDC second.* Rejected — MDC represents the most-recently-resolved value within the same request and should win over a possibly stale header value (e.g., if an inner filter already promoted it).

---

## R8. AspectJ proxy mode

**Decision**: Rely on Spring Boot's default `spring.aop.proxy-target-class=true`. Do not add `@EnableAspectJAutoProxy` to `DebugTraceAutoConfiguration`.

**Rationale**:
- Spring Boot 2+ ships with CGLIB proxies on by default, so service beans without interfaces (the common case) work out of the box. The user could explicitly disable this with `spring.aop.proxy-target-class=false`, in which case interface-only proxies are used and method matching still works for methods declared on interfaces.
- Adding `@EnableAspectJAutoProxy(proxyTargetClass = true)` would override the user's setting silently. The Constitution Principle I/V boundary says we accept proxy-only interception; we don't claim more.
- The starter declares `spring-boot-starter-aop`, which transitively pulls AspectJ weaver for `@Aspect` pointcut parsing — no JDK proxy gymnastics needed in our code.

**Alternatives considered**:
- *Force `proxyTargetClass=true` regardless of user setting.* Rejected — silent override violates Principle I (don't change observable behavior).
- *Document that users must set `spring.aop.proxy-target-class=true` explicitly.* Rejected — it's already the default.

---

## R9. Sample app placement

**Decision**: `examples/spring-boot-demo` is a sibling Maven project (its own `pom.xml`) with `spring-debug-trace-starter` declared as a regular dependency (`<scope>compile</scope>`). It is NOT a `<module>` of the starter's pom and the starter's pom is NOT a `pom`-packaged parent.

**Rationale**:
- The published artifact stays a single-jar deliverable; `mvn deploy` from the root produces exactly the starter jar/sources/javadoc — no demo confusion.
- The demo can be opened independently in an IDE, has its own dependency tree, and can stay on a different Spring Boot 3.x patch version if needed for compatibility testing.
- Maven Central reactors get unhappy when a `jar`-packaged module is also a parent; keeping them sibling projects avoids that class of build error.

**Alternatives considered**:
- *Multi-module parent + child reactor with `parent/`, `starter/`, `examples/spring-boot-demo/`.* Rejected for v1 — adds three poms instead of one for no v1 benefit (PRD §24 decision 11: "Implement a single Maven module first for speed"). Easy to migrate later.

---

## R10. Method matcher cache and short-circuit ordering

**Decision** (Phase 7 implementation detail decided here for cross-phase consistency):

- `MethodLogMatcher` keeps a `ConcurrentHashMap<Class<?>, MatchVerdict>` cache populated lazily on the first method invocation for each target class. `MatchVerdict` is `INCLUDE`, `SKIP` (out of scope), or `SKIP_REPOSITORY`.
- The aspect performs checks in this order, fastest first, so the no-match path returns in nanoseconds:
  1. `properties.isEnabled()` — single boolean read.
  2. `LoggingContext.depth() == 0 && base-packages empty` — single read.
  3. `matcher.match(targetClass)` — cache lookup.
  4. (Phase 7) slow-only branch: defer log emission until after `proceed()` so we know the duration.

**Rationale**:
- Cache key is `Class<?>`, not the proxy instance or the bean name, so two beans of the same class share one decision.
- `ConcurrentHashMap.computeIfAbsent` is the standard memoization pattern; the verdict computation is deterministic so concurrent computes are idempotent.
- Ordering puts cheapest checks first so the disabled and out-of-scope cases pay the least.

**Alternatives considered**:
- *No cache — recompute matches every call.* Rejected on performance grounds. Pointcut evaluation is already cached by Spring AOP, but our package/regex match is on top of that.
- *Eager precomputation at bean post-processing time.* Rejected — adds startup time for an unclear benefit, and the cache populates fast in practice (~one entry per unique bean class on its first call).

---

## R11. Logger naming and routing

**Decision**: Every log line emitted by the library — entry, exit, exception, HTTP entry/exit, internal warnings — uses an SLF4J logger obtained via `LoggerFactory.getLogger("io.github.cerovskimatija.debugtrace")` (the dedicated name, NOT the per-class default `LoggerFactory.getLogger(MyClass.class)`).

**Rationale**:
- Adopters need ONE name to gate or silence the library: setting `logging.level.io.github.cerovskimatija.debugtrace=OFF` must silence everything we emit.
- Per-class loggers would scatter trace lines across many SLF4J logger names, defeating the gating goal and bloating log-config files.
- Internal warnings (e.g., "serializer fell back to placeholder") at `WARN` on the same logger still respect the user's level setting (`WARN > OFF` is suppressed).

**Implementation rule**: Library classes still declare `private static final Logger LOG = LoggerFactory.getLogger("io.github.cerovskimatija.debugtrace")` — the fixed string — not `LoggerFactory.getLogger(getClass())`. Principle VII's Javadoc and parameterized-message rules still apply.

**Alternatives considered**:
- *Per-class loggers.* Rejected for the gating reason above.
- *Single static `Logger` in a `TraceLogs` helper class.* Equivalent to the chosen approach; the helper class would just hold the constant. Phase 1 may introduce this helper if call sites multiply.

---

## R12. AutoConfiguration registration file

**Decision**: Place the registration in `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` with a single line: `io.github.cerovskimatija.debugtrace.autoconfigure.DebugTraceAutoConfiguration`.

**Rationale**: This is the Spring Boot 3 mechanism (deprecating the legacy `spring.factories`). No legacy fallback is included — v1 supports Spring Boot 3.x only.

**Alternatives considered**:
- *Dual-register in `spring.factories` for Spring Boot 2 compatibility.* Rejected — Spring Boot 2.x is an explicit non-goal (PRD §6 item 1, Constitution Principle IV).

---

## Open issues, deferred to later phases

None block v1. Items intentionally deferred to specific phases per `PRD-phases.md`:

- **JSON-body masking false-positive rate** measurement — Phase 6 acceptance check.
- **JMH harness shape** (in-process loop vs. JMH plugin) — Phase 7 implementation choice.
- **Native-image (GraalVM) reachability metadata** — explicit non-goal in PRD §13.
- **Kotlin source/test fixtures** — explicit non-goal in PRD §13.

---

## Summary table of unresolved-to-resolved items

| Original question | Resolution | Source |
|---|---|---|
| Serializer engine | Pure reflection, no Jackson dependency for traversal | spec Clarifications Q1, decision R3 |
| Stack-trace in exception logs | Off by default; flag routes via SLF4J | spec Clarifications Q2, decision R4 |
| JSON body masking | Regex/string substitution on key patterns | spec Clarifications Q3, decision R5 |
| Per-call overhead budget | ≤ 5 % on ≥ 1 ms methods in default config | spec Clarifications Q4, decision R6 |
| Disabled by default? | Yes | PRD §24 decision 2 |
| base-packages required? | Yes — empty means silent | PRD §24 decision 3 / R2 |
| Trace-ID format | UUID when generating | PRD §24 / R7 |
| HTTP defaults | Metadata yes, headers + bodies opt-in | PRD §24 decisions 5/6 |
| Repositories included? | Yes by default, toggle to exclude | PRD §24 decision 10 |
| Module shape | Single Maven module | PRD §24 decision 11 / R9 |
| Log format | Pretty only in v1 | PRD §24 decision 8 |
| Default log level | DEBUG | PRD §24 decision 9 |
