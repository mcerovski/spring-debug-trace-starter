---
description: "Task list for Spring Debug Trace Starter v1"
---

# Tasks: Spring Debug Trace Starter v1

**Input**: Design documents from `/specs/001-v1-starter/`

**Prerequisites**: plan.md ✓, spec.md ✓, research.md ✓, data-model.md ✓, contracts/ ✓, quickstart.md ✓

**Tests**: Included. PRD §20 requires the six named unit-test classes plus an integration suite, and the constitution (Principle VI) makes each phase's acceptance demo the integration assertion for that phase. Tests are written first for each story and must fail before implementation begins.

**Organization**: Tasks are grouped by the seven user stories in `spec.md` (P1–P7). Each user-story phase ends with the spec's "Independent Test" criterion as the gate.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Different file, no dependency on incomplete tasks — safe to run in parallel.
- **[Story]**: `US1`–`US7`, matches the spec's priorities (P1 → US1, etc.).
- All file paths are relative to repo root `/Users/agents/projects/spring-debug-trace-starter`.

## Path Conventions

Single Maven module (per plan.md "Structure Decision"). Roots used in every path below:

- Production: `src/main/java/io/github/cerovskimatija/debugtrace/`
- Resources: `src/main/resources/`
- Tests: `src/test/java/io/github/cerovskimatija/debugtrace/`
- Sample app (Phase 8 / Polish): `examples/spring-boot-demo/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Maven module bootstrap and the auto-configuration registration scaffolding. Maps to PRD-phases.md Phase 0.

- [ ] T001 Create `pom.xml` at repo root with groupId `io.github.cerovskimatija`, artifactId `spring-debug-trace-starter`, version `1.0.0-SNAPSHOT`, packaging `jar`, java.source/target `17`. Declare dependencies: `spring-boot-starter-aop`, `spring-boot-autoconfigure`, `slf4j-api` (compile); `spring-web` and `jackson-databind` BOTH with `<optional>true</optional>` (neither propagates transitively to adopters — spec.md §263 Assumption requires Jackson to not be a hard runtime dependency); `spring-boot-configuration-processor` (provided/annotation processor); `spring-boot-starter-test` (test). Configure `maven-compiler-plugin`, `maven-surefire-plugin`, `maven-source-plugin`, `maven-javadoc-plugin`. Per plan.md "Primary Dependencies" + contracts/public-api.md §4.
- [ ] T002 [P] Create the by-feature package skeleton with empty `package-info.java` (Javadoc-only) files: `src/main/java/io/github/cerovskimatija/debugtrace/autoconfigure/package-info.java`, `properties/package-info.java`, `aspect/package-info.java`, `context/package-info.java`, `format/package-info.java`, `serializer/package-info.java`, `masking/package-info.java`, `trace/package-info.java`, `http/package-info.java`. Per plan.md "Project Structure" / Principle VII.
- [ ] T003 [P] Create `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` containing the single line `io.github.cerovskimatija.debugtrace.autoconfigure.DebugTraceAutoConfiguration`. Per research.md R12 and contracts/public-api.md §6.
- [ ] T004 [P] Add `LICENSE` at repo root (Apache 2.0 — required for Maven Central per Phase 8). Update `.gitignore` if needed for `target/`, `*.iml`, `.idea/`.
- [ ] T005 [P] Add the dedicated SLF4J logger name as a `public static final String LOGGER_NAME = "io.github.cerovskimatija.debugtrace"` constant on `DebugTraceProperties` (created in T006). Library classes obtain their logger via `LoggerFactory.getLogger(DebugTraceProperties.LOGGER_NAME)` — never `getClass()`, never a hard-coded string literal at the call site. Sequence-wise this is a one-line addition to T006's file; keeping it on `DebugTraceProperties` avoids introducing a separate utility class for a single constant. Per research.md R11.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Configuration-binding type, the empty auto-config class, and the dedicated logger conventions every later phase depends on. Maps to the rest of PRD-phases.md Phase 0.

**⚠️ CRITICAL**: No user story work can begin until this phase is complete.

- [ ] T006 [P] Create `src/main/java/io/github/cerovskimatija/debugtrace/properties/DebugTraceProperties.java` annotated `@ConfigurationProperties("spring-debug-trace")` with a top-level `enabled: boolean` (default `false`) and the `LOGGER_NAME` constant from T005. NO `format` property — a `format` configuration key would scaffold the post-v1 JSON-format non-goal (Constitution Principle III + contracts/configuration-properties.md). Nested static classes: `Method`, `Http`, `TraceId`, `Serialization`, `Masking` — all initially empty placeholders to be grown per phase. Constructor injection, `private final`, full Javadoc. Per contracts/configuration-properties.md, plan.md Principle VII.
- [ ] T007 Create `src/main/java/io/github/cerovskimatija/debugtrace/autoconfigure/DebugTraceAutoConfiguration.java` annotated `@AutoConfiguration` and `@EnableConfigurationProperties(DebugTraceProperties.class)`. Define empty nested `@Configuration(proxyBeanMethods = false)` classes `MethodLoggingConfiguration` (gated `@ConditionalOnProperty(prefix = "spring-debug-trace", name = "enabled", havingValue = "true")`) and `HttpLoggingConfiguration` (gated by both the `enabled` property and `@ConditionalOnClass(jakarta.servlet.Filter.class)`). Per research.md R1 + contracts/public-api.md §1.1. Depends on T006.
- [ ] T008 [P] Create the integration-test harness file `src/test/java/io/github/cerovskimatija/debugtrace/autoconfigure/DebugTraceAutoConfigurationIT.java` using `ApplicationContextRunner`. Add the baseline assertion: when `spring-debug-trace.enabled` is absent or `false`, `DebugTraceProperties` is bound but no method-logging or HTTP beans are present in the context. Per plan.md "Testing" + Principle VI.
- [ ] T009 [P] Add a `WARN`-on-startup hook in `DebugTraceAutoConfiguration` (via `@PostConstruct` on a small bean or `ApplicationContextInitializer`) that emits `base-packages is empty while enabled=true; no method logs will be emitted` exactly once when both conditions hold. Per research.md R2 + contracts/log-format.md §7.

**Checkpoint**: Auto-config wiring is live. The library builds, the `enabled=false` context-runner test passes, and the `enabled=true` path is ready for User Story 1.

---

## Phase 3: User Story 1 - See the bean method chain with one dependency (Priority: P1) 🎯 MVP

**Goal**: With the starter added and only `spring-debug-trace.enabled=true` + `base-packages` set, a Spring Boot 3 app emits indented `→`/`←` debug logs of every Spring bean method invocation inside the configured packages, with elapsed time on each exit. Library self-package and default framework packages are excluded automatically. Maps to PRD-phases.md Phase 1.

**Independent Test**: Install the JAR in a minimal Spring Boot 3 sample with a controller calling a service calling another service. Set `enabled=true` and `base-packages` to the app's root. Hit the endpoint and verify three nested `→` lines, three nested `←` lines, two-space-per-depth indent, and a non-zero `took=Nms` on every exit. Verify `enabled=false` produces zero library log lines (spec acceptance scenarios US1.1–US1.4).

### Tests for User Story 1 ⚠️

> Write these tests FIRST. Each must FAIL before implementation begins.

- [ ] T010 [P] [US1] Create `src/test/java/io/github/cerovskimatija/debugtrace/aspect/MethodLogMatcherTest.java` (PRD §20). Cases per data-model.md E6: include match for a class under `base-packages`; skip for `io.github.cerovskimatija.debugtrace.*` (self-exclusion); skip for each default-exclude prefix (`org.springframework`, `org.hibernate`, `jakarta.servlet`, `javax.servlet`, `com.fasterxml.jackson`, `org.slf4j`, `ch.qos.logback`); skip when `base-packages` is empty; user `excluded-packages` appends to defaults.
- [ ] T011 [P] [US1] Create `src/test/java/io/github/cerovskimatija/debugtrace/context/LoggingContextTest.java` (PRD §20). Cases per data-model.md E1: `push` then `pop` returns to 0; balanced push/pop across multiple iterations; `ThreadLocal.remove()` is invoked when returning to depth 0; two threads have independent depth counters.
- [ ] T012 [P] [US1] Create `src/test/java/io/github/cerovskimatija/debugtrace/format/LogFormatterTest.java` (PRD §20). Cases per contracts/log-format.md §2.3–2.4 (no trace prefix in Phase 1): entry line `→ ClassName.method` with no indent at depth 0; entry line with two-space indent at depth 1; entry line with four-space indent at depth 2; exit line `← ClassName.method took=Nms`; verify exact whitespace and that the no-`input=`/no-`output=` variant is used when `log-input=false`/`log-output=false`.
- [ ] T013 [US1] Extend `DebugTraceAutoConfigurationIT` with a Phase 1 assertion: when `spring-debug-trace.enabled=true` and `base-packages=[com.example]`, the context contains exactly one bean each of `DebugTraceProperties`, `MethodLogMatcher`, `LoggingContext`, `LogFormatter`, `MethodLoggingAspect`. Per contracts/public-api.md §1.1.
- [ ] T014 [US1] Create `src/test/java/io/github/cerovskimatija/debugtrace/aspect/MethodLoggingAspectIT.java` — a `@SpringBootTest` with a controller → service → downstream-service chain under `com.example.us1` and a `@TestConfiguration` enabling the library. Use a `ListAppender<ILoggingEvent>` attached to logger `io.github.cerovskimatija.debugtrace` to assert: exactly three entry lines, three exit lines, correct depth ordering, non-zero `took=` on each exit, library self-classes never appear, AND every captured event's `level == Level.DEBUG` (FR-039). Spec acceptance scenarios US1.1, US1.4.

### Implementation for User Story 1

- [ ] T015 [P] [US1] Implement `src/main/java/io/github/cerovskimatija/debugtrace/context/LoggingContext.java` — a `ThreadLocal<Integer>` wrapper with `push()`, `pop()`, `depth()`, `clearIfRoot()`. Public for Spring instantiation, package-internal API per contracts/public-api.md §2. Per data-model.md E1.
- [ ] T016 [P] [US1] Implement `src/main/java/io/github/cerovskimatija/debugtrace/aspect/MethodLogMatcher.java` — Phase 1 version: takes `DebugTraceProperties` via constructor; method `match(Class<?> targetClass)` returns `MatchVerdict.INCLUDE` or `SKIP_OUT_OF_SCOPE`. Algorithm follows data-model.md E6 steps 1, 2, 3, 6, 7, 8 (steps 4, 5 are Phase 7). Built-in default-exclude prefix list lives as a constant in this file. CGLIB proxy unwrap via `class.getSuperclass()` when the simple name contains `$$EnhancerBySpringCGLIB$$`.
- [ ] T017 [P] [US1] Implement `src/main/java/io/github/cerovskimatija/debugtrace/format/LogFormatter.java` — Phase 1 surface: `formatEntry(int depth, Class<?> targetClass, String methodName, String argSummaryOrNull)`, `formatExit(int depth, Class<?> targetClass, String methodName, String outputSummaryOrNull, long elapsedMillis)`. Pure string assembly per contracts/log-format.md §1 with two-space indent. No trace-id prefix in this phase (that arrives in US5). All output via the dedicated logger from T005.
- [ ] T018 [US1] Implement `src/main/java/io/github/cerovskimatija/debugtrace/aspect/MethodLoggingAspect.java` annotated `@Aspect`. Pointcut targets `Bean*` execution (`execution(public * *(..))`); around-advice does: matcher check (short-circuit on `SKIP_OUT_OF_SCOPE`), `LoggingContext.push()`, log entry via `LogFormatter`, `System.nanoTime()` start, `proceed()`, success: log exit with elapsed; finally: `LoggingContext.pop()` + `clearIfRoot()`. Constructor-inject `MethodLogMatcher`, `LoggingContext`, `LogFormatter`, `DebugTraceProperties`. Depends on T015, T016, T017.
- [ ] T019 [US1] Wire `MethodLogMatcher`, `LoggingContext`, `LogFormatter`, `MethodLoggingAspect` as `@Bean @ConditionalOnMissingBean` inside `MethodLoggingConfiguration` in `DebugTraceAutoConfiguration`. Per contracts/public-api.md §1.1 bean contract. Depends on T015–T018.
- [ ] T020 [US1] Extend `DebugTraceProperties` with the Phase 1 keys: `basePackages: List<String>` (default `[]`) and `excludedPackages: List<String>` (default `[]`). NO `Method.enabled` sub-property — there is one master switch (`spring-debug-trace.enabled`); adding a per-area enable toggle is scope expansion not in spec.md. Add validation: trim entries, drop nulls. Per contracts/configuration-properties.md top-level.

**Checkpoint**: User Story 1 is fully functional. Tests T010–T014 pass. The Phase 1 acceptance demo in PRD-phases.md runs.

---

## Phase 4: User Story 2 - See what each method received and returned (Priority: P2)

**Goal**: With `method.log-input=true` and `method.log-output=true`, entry lines show `(input=[...])` and exit lines show `(output=...)`. Reflective `SafeLogSerializer` walks DTOs with depth/string/collection/total-output caps, replaces servlet/stream/multipart/principal types with short placeholders, detects circular references, and never throws. Maps to PRD-phases.md Phase 2.

**Independent Test**: With US1 working, enable `log-input`/`log-output` and exercise endpoints with primitives, DTOs, collections, a multipart upload, and a self-referential graph. Verify all inputs and outputs render readably, the upload appears as `<MultipartFile ...>`, the cyclic graph terminates with `<cycle: ClassName>`, and oversize strings/collections show their `... (truncated)` / `... (+M more)` markers (spec US2.1–US2.5).

### Tests for User Story 2 ⚠️

- [ ] T021 [P] [US2] Create `src/test/java/io/github/cerovskimatija/debugtrace/serializer/SafeLogSerializerTest.java` (PRD §20). Cases per data-model.md E7 and contracts/log-format.md §2.8: primitives, strings (under and over `max-string-length`), enums, records, DTOs with simple fields, nested DTOs (depth under and over `max-depth`), collections (under and over `max-collection-size`), maps, arrays, byte arrays > 128 bytes, cyclic graphs (`a.b = b; b.a = a` style), DTOs whose getter throws (→ `<serialization-failed: ClassName>`), `null` field with `includeNullFields=false` (omitted) and `=true` (`field=null`), total output cap (`<... (object truncated)>`).
- [ ] T022 [P] [US2] Add cases to `SafeLogSerializerTest` for the `SkipTypeDetector` skip categories per contracts/log-format.md §2.9: `MultipartFile`, `HttpServletRequest`, `HttpServletResponse`, `InputStream`, `OutputStream`, `File`, `Resource`, `BindingResult`, `Principal`, `Authentication`. Each MUST render the documented `<TypeName ...>` placeholder.
- [ ] T023 [US2] Extend `MethodLoggingAspectIT` (or add a sibling IT) with US2 assertions: when `method.log-input=true` the entry line contains `input=[...]`; with `method.log-output=true` the exit line contains `output=...`; a method returning `void` omits the `output=` segment entirely (spec US2.5); a method returning `null` renders as `output=null`.

### Implementation for User Story 2

- [ ] T024 [P] [US2] Implement `src/main/java/io/github/cerovskimatija/debugtrace/serializer/SkipTypeDetector.java` — small class with `Optional<String> describeIfSkippable(Object value)` returning the formatted `<TypeName ...>` placeholder when the runtime type matches one of the categories in contracts/log-format.md §2.9. `MultipartFile` / `HttpServletRequest` / `HttpServletResponse` / `BindingResult` / `Authentication` / `Principal` references obtained via `Class.forName` so the file does not hard-depend on `spring-web` / Spring Security.
- [ ] T025 [P] [US2] Implement `src/main/java/io/github/cerovskimatija/debugtrace/serializer/SafeLogSerializer.java` — pure-reflection walker with `String summarize(Object value)`. Honors `serialization.max-depth`, `max-string-length`, `max-collection-size`, `max-object-length`, `include-null-fields`. Uses an `IdentityHashMap<Object, Boolean>` for cycle detection. Consults `SkipTypeDetector` first at every node. All field-walking and value-rendering is wrapped in `try`/`catch (Throwable)` returning `<serialization-failed: ClassName>` — never throws. No mutable fields (per data-model.md E7 thread-affinity invariant). No Jackson usage in the traversal path (research.md R3).
- [ ] T026 [US2] Extend `DebugTraceProperties` with `Method.logInput: boolean` (default `true`), `Method.logOutput: boolean` (default `true`), and the full `Serialization` nested class (`maxDepth: int = 3`, `maxStringLength: int = 1000`, `maxCollectionSize: int = 20`, `maxObjectLength: int = 5000`, `includeNullFields: boolean = false`) with the clamps listed in contracts/configuration-properties.md. Add Javadoc per Principle VII.
- [ ] T027 [US2] Wire `SkipTypeDetector` and `SafeLogSerializer` as `@Bean @ConditionalOnMissingBean` inside `MethodLoggingConfiguration`. Update `MethodLoggingAspect` to invoke `SafeLogSerializer.summarize(args)` (joined by `, `) on entry when `method.log-input=true`, and `SafeLogSerializer.summarize(returnValue)` on exit when `method.log-output=true`. Wrap every serializer call in `try`/`catch (Throwable)` with `<serialization-failed: ClassName>` fallback. Void returns OMIT the segment entirely. Depends on T024–T026.
- [ ] T028 [US2] Update `LogFormatter` to accept the optional `input=[...]` and `output=...` segments in `formatEntry` / `formatExit` per contracts/log-format.md §1 grammar. Whitespace rules from §3 are non-negotiable; assert in the existing `LogFormatterTest`.

**Checkpoint**: Stories 1 and 2 both work. Tests T021–T023 pass alongside US1's. The Phase 2 acceptance demo runs.

---

## Phase 5: User Story 3 - See exceptions inline, never break the application (Priority: P3)

**Goal**: When an intercepted method throws, the library emits `× ClassName.method threw ExceptionType(message=...) after Nms` at the right indent, then rethrows the original exception unchanged. If the library's own machinery fails anywhere, a safe placeholder replaces the affected segment and the request continues. Per-thread depth resets cleanly. Maps to PRD-phases.md Phase 3.

**Independent Test**: Throw a custom exception from a deeply nested service; verify the `×` line appears at the correct indent, the original exception still reaches `@ControllerAdvice`, and depth is zero on the next request handled by the same thread. Plant a DTO whose getter throws; verify the log shows `<serialization-failed: ...>` and the request still completes (spec US3.1–US3.4, SC-010).

### Tests for User Story 3 ⚠️

- [ ] T029 [P] [US3] Extend `LogFormatterTest` with exception-line cases per contracts/log-format.md §2.2: `× ClassName.method threw ExceptionType(message=...) after Nms`, with two-space indent at depth 1, and an exception line at depth 0; verify `message=` renders empty string when `getMessage()` is `null`.
- [ ] T030 [P] [US3] Extend `SafeLogSerializerTest` with a "never throws" property test: feed 20+ adversarial inputs (throwing getters, throwing `toString`, throwing `equals`/`hashCode`, anonymous classes with no field accessors, JPA-style lazy proxies whose access throws) and assert that every call returns a non-null string and the placeholder is `<serialization-failed: ClassName>` or `<lazy-access-failed: ClassName>`.
- [ ] T031 [US3] Add to `MethodLoggingAspectIT`: a service that throws a custom unchecked exception; assert the exception line appears at the correct indent, the next request on the same thread starts at depth 0 (spec US3.4), and the controller's `@ExceptionHandler` receives the original exception unwrapped (FR-018, US3.2). Use `ListAppender` + reflection on `LoggingContext` (test-scope access) to verify depth reset.
- [ ] T032 [US3] Add an IT case where a logged DTO's getter throws on a non-erroring controller path: assert the HTTP response is the controller's normal `200 OK` (the broken accessor is in the DTO graph the library tried to render, NOT in business logic), and that the captured log line contains `<serialization-failed: BrokenDtoSimpleName>` exactly where the value would have rendered (SC-010). The point of this test is to prove the library swallows its own failure without changing observable behavior — pin both halves.

### Implementation for User Story 3

- [ ] T033 [US3] Extend `MethodLoggingAspect` with the exception branch: catch `Throwable` around `proceed()`, emit the `×` exception line via `LogFormatter.formatException(depth, class, method, throwable, elapsed)` ONLY when `method.log-exceptions=true`, then rethrow unchanged. The `finally` block from US1 still pops and clears. When `method.log-exception-stack-trace=true`, pass the throwable as the last SLF4J argument so the user's appender prints the stack. Per research.md R4, contracts/log-format.md §6.
- [ ] T034 [P] [US3] Add `formatException` to `LogFormatter` per contracts/log-format.md §1 grammar. Wrap the entire formatter body in `try`/`catch (Throwable)` returning a constant safe fallback (e.g. `× <log-format-failed>`).
- [ ] T035 [P] [US3] Add the `lazy-access-failed` fallback path inside `SafeLogSerializer`'s field walker: catch `Throwable` when reading a field, attempt one retry through the getter if present, then emit `<lazy-access-failed: SimpleClassName>` per data-model.md E7 invariant 3.
- [ ] T036 [US3] Extend `DebugTraceProperties.Method` with `logExceptions: boolean` (default `true`) and `logExceptionStackTrace: boolean` (default `false`) per contracts/configuration-properties.md.
- [ ] T037 [P] [US3] Audit every existing call site of `SafeLogSerializer.summarize`, `LogFormatter.formatXxx`, and SLF4J `log.debug` in the aspect: each MUST be wrapped in `try`/`catch (Throwable)` with the documented placeholder fallback. Add a comment ONLY if the safety wrapping is non-obvious (it usually is — keep silent otherwise). Per Constitution Principle I and FR-019.

**Checkpoint**: Stories 1–3 all work. Tests T029–T032 pass. The Phase 3 acceptance demo runs (`SC-010` verified).

---

## Phase 6: User Story 4 - Common secrets are hidden in logs by default (Priority: P4)

**Goal**: `SensitiveDataMasker` is wired into the serializer's field-name path. Default field list (PRD §10) is loaded; user-supplied `masking.fields` APPEND to defaults; matching is case-insensitive; the configured `masking.replacement` placeholder substitutes the value. Masking failure NEVER allows the original value to escape. Maps to PRD-phases.md Phase 4.

**Independent Test**: Send a `LoginRequest{username, password}` through a logged controller — assert `password=****` appears and the raw value never does. Add a custom field name; assert it is masked alongside the defaults (spec US4.1–US4.4).

### Tests for User Story 4 ⚠️

- [ ] T038 [P] [US4] Create `src/test/java/io/github/cerovskimatija/debugtrace/masking/SensitiveDataMaskerTest.java` (PRD §20). Cases per data-model.md E5: every default field name is masked; case-insensitive (`Password`, `PASSWORD`, `password` all masked); user-supplied additions are masked AND default list still masked; null/blank entries in `masking.fields` are dropped; `masking.enabled=false` returns `false` from `shouldMask`; `maskValue` returns `replacement` for any input (including `null`); construction with null `masking.fields` does not throw.
- [ ] T039 [P] [US4] Add to `SafeLogSerializerTest` (or a sibling test): when a DTO has a field named in the masking list, the rendered output shows `password=****` (the configured placeholder) and the original value appears nowhere in the rendered output. Add a paranoid scan that the test's plaintext value is absent from the entire rendered string.
- [ ] T040 [US4] Add a `MethodLoggingMaskingIT` (or extend the existing IT) that exercises the controller-to-service path with a real `LoginRequest`, asserting via `ListAppender` that the rendered log lines mask the password (spec US4.1–US4.2) and that adding a custom field via `application.yml` masks that too while the defaults remain masked (spec US4.2).

### Implementation for User Story 4

- [ ] T041 [P] [US4] Implement `src/main/java/io/github/cerovskimatija/debugtrace/masking/SensitiveDataMasker.java` per data-model.md E5: immutable `Set<String> fieldNames` constructed at injection time as `Set.copyOf(union(defaults, userSupplied))` all lowercased via `Locale.ROOT`. Public methods: `boolean shouldMask(String fieldName)`, `String maskValue(Object originalValue)`. Defaults list = `password, pass, token, accessToken, refreshToken, authorization, cookie, secret, apiKey, cardNumber, iban`. Construction NEVER throws on null/blank input. JSON-body masking method is Phase 6 / US6 — not yet.
- [ ] T042 [US4] Extend `DebugTraceProperties` with `Masking` nested: `enabled: boolean` (default `true`), `replacement: String` (default `"****"`), `fields: List<String>` (default `[]` from the user — defaults are constants inside the masker). Null replacement falls back to `"****"`. Per contracts/configuration-properties.md `masking.*`.
- [ ] T043 [US4] Wire `SensitiveDataMasker` as `@Bean @ConditionalOnMissingBean` inside `MethodLoggingConfiguration`. Inject it into `SafeLogSerializer` and consult `shouldMask(fieldName)` at each field-render step; replace the rendered value with `masker.maskValue(...)` BEFORE applying any other transformation. Wrap the masker call in `try`/`catch (Throwable)` returning the configured replacement — the original value MUST NEVER reach the log on failure (FR-019, US4.4). Depends on T041, T042.

**Checkpoint**: Stories 1–4 all work. Tests T038–T040 pass. The Phase 4 acceptance demo runs (SC-004 verified).

---

## Phase 7: User Story 5 - Correlate logs with HTTP context and a trace ID (Priority: P5)

**Goal**: When Spring MVC is on the classpath, `HttpLoggingFilter` emits `HTTP →` and `HTTP ←` bracket lines around each request. `TraceIdManager` resolves a trace ID per request (existing MDC → configured headers → generated UUID), writes it into MDC under the configured key, and removes ONLY the key it placed. `LogFormatter` prefixes every line with `[traceId=<value>] `. Without Spring MVC on the classpath, the filter is silently absent and method logging still works. Maps to PRD-phases.md Phase 5.

**Independent Test**: Hit an endpoint twice from two clients in quick succession — assert each request's log lines share one trace ID and the two trace IDs differ. Send a third request with `X-Request-Id: my-id` and assert `[traceId=my-id]` appears across its lines. Remove `spring-web` from the test classpath and confirm the app still starts (spec US5.1–US5.5).

### Tests for User Story 5 ⚠️

- [ ] T044 [P] [US5] Create `src/test/java/io/github/cerovskimatija/debugtrace/trace/TraceIdManagerTest.java` (PRD §20). Cases per data-model.md E4: existing MDC value is returned and is NOT removed on cleanup; configured header is promoted to MDC and IS removed on cleanup; generated UUID is written and IS removed on cleanup; resolution order MDC → headers → generated; `generate-if-missing=false` returns `null` and prefix is omitted; blank header values are skipped; case-insensitive header matching.
- [ ] T045 [P] [US5] Extend `LogFormatterTest` with trace-prefix cases per contracts/log-format.md §4: prefix appears IFF MDC has a non-blank value; prefix is at column 0 (precedes indent); prefix shape `[traceId=<value>] ` with one trailing space; absence of MDC → no `[traceId=null]`, no `[traceId=]`, just no prefix at all.
- [ ] T046 [US5] Create `src/test/java/io/github/cerovskimatija/debugtrace/http/HttpLoggingFilterIT.java` — `@SpringBootTest(webEnvironment = RANDOM_PORT)` + MockMvc. Cases per spec US5.1–US5.4: HTTP entry line precedes the controller's entry line; HTTP exit line follows the controller's exit; both lines render at depth 0; every log line for one request shares the trace ID; two concurrent requests have different trace IDs; supplied `X-Request-Id` header is preserved; MDC is cleared at filter exit.
- [ ] T047 [US5] Add a "no servlet API" test scenario using `ApplicationContextRunner.withClassLoader(FilteredClassLoader.class.getName(), "jakarta.servlet"...)` — assert the app context starts, `MethodLoggingAspect` is present, `HttpLoggingFilter` is absent (spec US5.5, FR-026).

### Implementation for User Story 5

- [ ] T048 [P] [US5] Implement `src/main/java/io/github/cerovskimatija/debugtrace/trace/TraceIdManager.java` per data-model.md E4. Methods: `String resolve(HttpServletRequest req)`, `void cleanup()`. Tracks source (`EXISTING_MDC` / `REQUEST_HEADER` / `GENERATED`) per-thread via a small `ThreadLocal<TraceIdEntry>` so cleanup can decide whether to call `MDC.remove(key)`. Headers iterated in configured order, case-insensitive. UUID generation only when `generate-if-missing=true`.
- [ ] T049 [P] [US5] Implement `src/main/java/io/github/cerovskimatija/debugtrace/http/HttpLoggingFilter.java` extending `OncePerRequestFilter`. Phase 5 shape (no body wrappers yet): resolve trace ID via `TraceIdManager` BEFORE the entry log; emit `HTTP → METHOD URI?query` at depth 0; `chain.doFilter(...)` inside `try`; in `finally`, emit `HTTP ← METHOD URI status=N took=Nms`, then `traceIdManager.cleanup()`. Logger is the dedicated logger. Order: `Ordered.HIGHEST_PRECEDENCE + 10` so it brackets all other filters but does not pre-empt Spring's required infrastructure.
- [ ] T050 [US5] Update `LogFormatter` to prepend `[traceId=<value>] ` at column 0 whenever `MDC.get(properties.getTraceId().getMdcKey())` is non-blank, per contracts/log-format.md §4. Indent and arrow follow the prefix.
- [ ] T051 [US5] Add HTTP entry/exit formatting methods to `LogFormatter` per contracts/log-format.md §5 grammar (`HTTP → ...`, `HTTP ← ...`). Both render at depth 0 (no indent).
- [ ] T052 [US5] Extend `DebugTraceProperties` with `Http.enabled: boolean = true`, `TraceId.enabled: boolean = true`, `TraceId.mdcKey: String = "traceId"` (blank → fallback to `"traceId"`), `TraceId.generateIfMissing: boolean = true`, `TraceId.requestHeaders: List<String> = [X-Request-Id, X-Correlation-Id, traceparent]`. Per contracts/configuration-properties.md `http.*` / `trace-id.*`.
- [ ] T053 [US5] Wire `TraceIdManager` and `HttpLoggingFilter` as `@Bean @ConditionalOnMissingBean` inside `HttpLoggingConfiguration`. Filter registration uses `FilterRegistrationBean` with the order from T049. Conditional gates already in place from T007. Depends on T048, T049, T052.

**Checkpoint**: Stories 1–5 all work. Tests T044–T047 pass. The Phase 5 acceptance demo runs (SC-005 verified).

---

## Phase 8: User Story 6 - Optionally see HTTP headers and bodies, safely (Priority: P6)

**Goal**: Opt-in `http.log-headers`, `http.log-request-body`, `http.log-response-body` toggles. Sensitive headers masked. Bodies captured via Spring's `ContentCachingRequestWrapper` / `ContentCachingResponseWrapper`, truncated to `max-body-length`, with binary/multipart summarized rather than dumped. JSON-style key masking applied to the captured copy via regex (research.md R5). `copyBodyToResponse()` MUST run so the client sees the unmodified bytes. Maps to PRD-phases.md Phase 6.

**Independent Test**: Enable both body toggles. POST a JSON body containing `password` — assert the logged body shows `"password":"****"` and the truncation marker if the body exceeds the cap. Assert the HTTP response delivered to the client is byte-for-byte unchanged (SC-006). POST a binary upload — assert the log shows `body=<binary, N bytes>` (spec US6.1–US6.5).

### Tests for User Story 6 ⚠️

- [ ] T054 [P] [US6] Add to `SensitiveDataMaskerTest`: `maskJsonBodyKeys` cases — single key in a JSON body is replaced; multiple keys; nested JSON keys; numeric values; key inside a free-text field (accepted false-positive); malformed JSON does NOT throw and returns input as-is or with best-effort substitution; the replacement uses the configured placeholder.
- [ ] T055 [P] [US6] Add to `HttpLoggingFilterIT` (or a sibling IT): when `http.log-headers=true`, sensitive headers are masked (US6.1); when `http.log-request-body=true`, body appears with sensitive JSON keys masked (US6.2); when `http.log-response-body=true`, the response delivered to the client is byte-for-byte identical to a run with body logging off (US6.3, SC-006); binary/multipart bodies render as the summary placeholders (US6.4); with both flags OFF, no body bytes ever appear in any log line (US6.5).
- [ ] T056 [P] [US6] Add a regression IT that asserts `copyBodyToResponse()` is invoked even when the body-logging code path throws — e.g., inject a fault into the masker mid-flight and verify the client still receives the full response (SC-006 + Constitution Principle I).

### Implementation for User Story 6

- [ ] T057 [P] [US6] Add `String maskJsonBodyKeys(String body)` to `SensitiveDataMasker`. Iterates the field list and applies `Pattern.compile("\"" + Pattern.quote(field) + "\"\\s*:\\s*\"[^\"]*\"")` and the unquoted-numeric variant; replaces only the value portion with the configured replacement. Per research.md R5. Wrapped in `try`/`catch (Throwable)` returning the original input on internal failure (NOT on regex non-match — that returns unchanged because no replacement happened).
- [ ] T058 [P] [US6] Add the header masking path inside `SensitiveDataMasker`. The masker now exposes a single `DEFAULT_FIELD_NAMES` constant (the eleven names from T041); the header set is constructed as `Set.copyOf(union(fieldNames, Set.of("set-cookie")))` lowercased — `authorization` and `cookie` already live in `DEFAULT_FIELD_NAMES`, so we extend with `set-cookie` only and avoid two drift-prone defaults lists. Method: `boolean shouldMaskHeader(String name)`.
- [ ] T059 [US6] Update `HttpLoggingFilter` to wrap the request in `ContentCachingRequestWrapper` and the response in `ContentCachingResponseWrapper` ONLY when at least one of the body flags is on. After `chain.doFilter`, build the log line: header map (with masked sensitive headers per T058) IFF `log-headers=true`; truncated body bytes (with binary/multipart short-circuit per `Content-Type` check and `skip-binary-content` / `skip-multipart-content` flags) IFF the corresponding `log-*-body` flag is on; JSON masking applied to the captured copy via T057. In `finally`, ALWAYS invoke `responseWrapper.copyBodyToResponse()` BEFORE `traceIdManager.cleanup()`. Per data-model.md E8 invariants. Depends on T057, T058.
- [ ] T060 [US6] Extend `LogFormatter` HTTP methods to accept optional `headers={...}` and `body=...` segments per contracts/log-format.md §5. Header map uses single-line `{name=value, name=value}` format with servlet-enumeration order. Body segment is the masked, truncated string.
- [ ] T061 [US6] Extend `DebugTraceProperties.Http` with `logHeaders: boolean = false`, `logRequestBody: boolean = false`, `logResponseBody: boolean = false`, `maxBodyLength: int = 5000` (≤ 0 → no body logged), `skipBinaryContent: boolean = true`, `skipMultipartContent: boolean = true`. Per contracts/configuration-properties.md `http.*`.

**Checkpoint**: Stories 1–6 all work. Tests T054–T056 pass. The Phase 6 acceptance demo runs (SC-006 verified by byte-comparison test).

---

## Phase 9: User Story 7 - Tune verbosity for staging and cautious production (Priority: P7)

**Goal**: `method.log-only-slow=true` + `slow-threshold-ms`: methods completing under the threshold produce zero log lines; methods over the threshold produce both entry and exit; exceptions still emit regardless of duration. `excluded-class-name-patterns` regex list (defaults `.*Configuration`, `.*Properties`) skips matched classes. `method.include-repositories=false` skips Spring `@Repository` beans. Per-class `MatchVerdict` cache short-circuits hot paths. With `enabled=false`, overhead is within 5 % of a no-library baseline (SC-007). Maps to PRD-phases.md Phase 7.

**Independent Test**: With `log-only-slow=true, threshold=100`: a 10 ms method produces no output; a 250 ms method produces entry + exit; a throw at any duration produces the exception line. With `excluded-class-name-patterns=[.*Mapper]`: a `FooMapper` is silent while `FooService` still logs. With `include-repositories=false`: a `JpaRepository`-implementing bean is silent. Run a 10 000-call benchmark with `enabled=false` and verify ≤ 5 % overhead vs. a no-library baseline (spec US7.1–US7.4, SC-007, SC-011).

### Tests for User Story 7 ⚠️

- [ ] T062 [P] [US7] Extend `MethodLogMatcherTest` with the Phase 7 cases per data-model.md E6 steps 4 and 5: invalid regex in `excluded-class-name-patterns` is dropped with a `WARN` (not throw); valid regex matches by FQN; `SKIP_REPOSITORY` returned for `@Repository`-annotated classes and for classes implementing `org.springframework.data.repository.Repository` when `include-repositories=false`; cache returns the same verdict on subsequent calls (use a `ConcurrentHashMap` spy or a memoization assertion).
- [ ] T063 [P] [US7] Add slow-only IT cases to `MethodLoggingAspectIT`: a fast method (~5 ms) produces zero lines when `log-only-slow=true` and `threshold=100`; a slow method (use `Thread.sleep(150)` in a `@TestComponent`) produces entry + exit lines; a throwing method produces the `×` line regardless of duration; the entry line's indent value MUST match the depth at the moment the call was entered, not the moment it was emitted (contracts/log-format.md §2.10).
- [ ] T064 [P] [US7] Create a microbenchmark harness `src/test/java/io/github/cerovskimatija/debugtrace/perf/DisabledModeOverheadBenchmark.java` (JUnit-driven, in-process loop is fine for v1 — JMH is optional per research.md "Open issues"). Runs a 10 000-call loop on a representative bean method with `enabled=true` and `enabled=false`, comparing to a no-aspect baseline run in the same JVM. Asserts SC-007 (`enabled=false` ≤ 5 %) and SC-011 (default-config overhead ≤ 5 % on ≥ 1 ms methods). Test is marked `@Tag("perf")` so the main Surefire run can skip it and a `verify`-bound execution runs it on demand.

### Implementation for User Story 7

- [ ] T065 [P] [US7] Add the `ConcurrentHashMap<Class<?>, MatchVerdict>` cache to `MethodLogMatcher` (data-model.md E6, research.md R10). `computeIfAbsent` populates lazily. Reads on the hot path become a single map lookup. CGLIB proxy unwrap continues to be the cache-key normalization step.
- [ ] T066 [P] [US7] Add the `excluded-class-name-patterns` evaluation step to `MethodLogMatcher` (data-model.md E6 step 4). At construction time, parse each regex via `Pattern.compile`; on a `PatternSyntaxException`, log one `WARN` line through the dedicated logger and DROP that entry — never throw. Match the class FQN against each compiled pattern.
- [ ] T067 [P] [US7] Add the repository-skip step to `MethodLogMatcher` (data-model.md E6 step 5). When `method.include-repositories=false`, skip classes annotated `@Repository` and classes implementing `org.springframework.data.repository.Repository` (resolved via `Class.forName` so the library does not hard-depend on Spring Data).
- [ ] T068 [US7] Implement slow-only mode in `MethodLoggingAspect`. Strategy: when `log-only-slow=true`, defer the entry-line emission until after `proceed()`. Buffer the entry parameters (depth, class, method, optional arg summary) in a small stack-local record; on normal exit, if `elapsedMillis ≥ slowThresholdMs`, emit BOTH the entry and the exit lines using the buffered values; on exception, emit the exception line regardless of duration (when `log-exceptions=true`) and skip the entry line. Per contracts/log-format.md §2.10 — including the **nesting rule**: each frame's slow/fast decision is independent of its parent or children. A slow outer whose inner calls were all fast emits only the outer pair (at the outer's captured depth, e.g. 0); the fast inner calls leave gaps in the visible chain. A fast outer with a slow inner emits only the inner pair at the inner's captured depth (e.g. 2 spaces) with no parent visible above it. Indent values are the values captured at call entry, never recomputed at emission time. Document the "buffered argument serialization runs on every call when `log-input=true`" overhead in the test or a code comment so the SC-011 budget caveat from contracts/log-format.md §2.10 is visible to future maintainers. Depends on T069 for the new properties.
- [ ] T069 [US7] Extend `DebugTraceProperties` with `Method.logOnlySlow: boolean = false`, `Method.slowThresholdMs: long = 500` (negative clamped to 0), `Method.includeRepositories: boolean = true`, top-level `excludedClassNamePatterns: List<String> = [".*Configuration", ".*Properties"]`. Per contracts/configuration-properties.md.

**Checkpoint**: All seven user stories are independently functional. Tests T062–T064 pass. Phase 7 acceptance demo runs (SC-007 and SC-011 verified by the benchmark harness).

---

## Phase 10: Polish & Cross-Cutting Concerns

**Purpose**: Phase 8 release work — sample app, README expansion, Javadoc audit, Maven Central publishing wiring, and the full PRD §20 integration suite. None of these add features; they prepare v1.0.0 for adopters.

- [ ] T070 [P] Create the sample application at `examples/spring-boot-demo/` per plan.md "Project Structure" / research.md R9: `pom.xml` (sibling project, NOT a child module of the starter), `src/main/java/.../OrdersDemoApplication.java`, `web/OrderController.java`, `service/OrderService.java`, `service/PaymentService.java`, `service/InventoryService.java`, `repository/OrderRepository.java`, `dto/CreateOrderRequest.java`, `dto/Order.java`, `dto/LoginRequest.java`, `src/main/resources/application.yml`. The YAML enables `spring-debug-trace` with the user's app package and demonstrates the quickstart's four most-customized properties (SC-009).
- [ ] T071 [P] Expand `README.md` to mirror `specs/001-v1-starter/quickstart.md`: install snippet, two-property minimum config, expected output, security warning, troubleshooting, and the SC-009 four-property panel. The Phase 8 README is what a fresh adopter reads — make sure it stands alone without the spec.
- [ ] T072 [P] Javadoc audit on every public and protected type and member in `io.github.cerovskimatija.debugtrace.*` per Principle VII. The two `@ConfigurationProperties` and `DebugTraceAutoConfiguration` are the highest priority. Configure `maven-javadoc-plugin` to fail the build on missing Javadoc on public/protected members.
- [ ] T073 [P] Wire Maven Central publishing in `pom.xml`: `distributionManagement`, `maven-source-plugin`, `maven-javadoc-plugin`, `maven-gpg-plugin`, `central-publishing-maven-plugin` (or `nexus-staging-maven-plugin`). Add a `release` profile so day-to-day builds do not require GPG. Sample app is excluded from the release. Per contracts/public-api.md §4 and Phase 8.
- [ ] T074 [P] Add the integration-test matrix per Constitution Principle VI: a `spring-boot-starter-web` smoke test (HTTP entry/exit + method chain), a no-`spring-web` smoke test (method chain only), a `@ControllerAdvice` interaction test (US3.2), a Spring Data JPA repository-toggle test (US7.3), a Spring Security `Principal`-skipping test (covered earlier but assert end-to-end here), and a Micrometer / Sleuth coexistence test (no bean conflicts per contracts/public-api.md §4.4).
- [ ] T075 [P] Wire CI to run `mvn verify` against the Java 17 + Java 21 matrix on the latest Spring Boot 3.x patch line (and one prior minor for safety). Per plan.md "Compatibility matrix" and Principle VI. Place workflow under `.github/workflows/build.yml`.
- [ ] T076 Update `CLAUDE.md` "Repository State" line to reflect that the Maven module now exists, list `mvn verify` as the build/test command, and add the sample-app run command. Replace the "pre-implementation state" framing with a brief "v1 shipped" summary once the rest of Phase 10 is done.
- [ ] T077 Run the `specs/001-v1-starter/quickstart.md` walkthrough end-to-end against the published-locally JAR (`mvn -DskipTests install` then `cd examples/spring-boot-demo && ./mvnw spring-boot:run`). This is the SC-001 regression script — confirm install + first log under 5 minutes. Capture the actual console output and compare against quickstart.md §5.
- [ ] T078 [P] Create `src/test/java/io/github/cerovskimatija/debugtrace/autoconfigure/DualModeIT.java` — a `@SpringBootTest` parameterized over `spring-debug-trace.enabled=true` and `=false` (e.g. via `@ParameterizedTest` driving two `ApplicationContextRunner` configurations) that exercises the full happy-path controller chain, the `@ControllerAdvice` exception-translation path, and a body-roundtrip path. The same assertion set runs in both modes: HTTP status codes match, response body bytes match, exception type reaching `@ExceptionHandler` matches. This is the SC-003 regression script ("zero failures caused by the library: every assertion on application behavior holds identically in both modes") and FR-020 — turning the "library must not alter observable behavior" promise into a single dual-run test rather than relying on hand-pairing tests across the suite. Per spec.md SC-003 / FR-020.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: no dependencies — start immediately.
- **Phase 2 (Foundational)**: depends on Phase 1 completion. **Blocks all user stories.**
- **Phase 3–9 (US1–US7)**: each depends on Phase 2 completion. The user stories are intentionally ordered to match `PRD-phases.md` Phases 1–7. PRD-phases.md notes that the order is load-bearing — earlier phases are foundations for later ones — so although the stories are independently testable, they should be implemented in P1 → P7 order.
- **Phase 10 (Polish)**: depends on all desired user stories being complete. T070 and T071 can begin once US1 lands (sample app and README evolve with each phase).

### User Story Dependencies

- **US1 (P1)**: depends only on Phase 2.
- **US2 (P2)**: depends on US1 (the aspect must exist before input/output rendering plugs into it; `LogFormatter` grows the `input=`/`output=` grammar on top of US1's shapes).
- **US3 (P3)**: depends on US1 (exception branch lives in the same aspect). Hardens the serializer added in US2 but can be implemented even with US2's tests passing — the resilience wrappers protect every existing call site.
- **US4 (P4)**: depends on US2 (the masker is consulted by `SafeLogSerializer`).
- **US5 (P5)**: depends on US1 (`LogFormatter` gains the trace-id prefix; the HTTP filter brackets US1's method chain). Independent of US2/US3/US4 — could be implemented in parallel with them after US1 is done.
- **US6 (P6)**: depends on US4 (masker.maskJsonBodyKeys + header masking) and US5 (the HTTP filter is the host for body wrappers).
- **US7 (P7)**: depends on US1 (matcher and aspect). Slow-only mode interacts with US3's exception branch (exceptions still emit), so US3 should be in place first.

### Within Each User Story

- Tests are written and verified failing BEFORE the implementation tasks begin (Principle VI).
- Within implementation: properties first → matcher/formatter/serializer/masker primitives → aspect/filter integration → autoconfig wiring last.
- Each story's checkpoint MUST run its acceptance demo from PRD-phases.md before moving to the next.

### Parallel Opportunities

- All Setup tasks marked `[P]` (T002, T003, T004, T005) run in parallel; T001 must land first because it creates `pom.xml`.
- All Foundational `[P]` tasks (T006, T008, T009) can run in parallel after T001; T007 depends on T006.
- Within each user story: tests `[P]` run together; implementation tasks marked `[P]` are independent files and run together.
- Across user stories: US5 (HTTP/trace-id) can be developed in parallel with US3/US4 once US1 lands, because the HTTP filter and `TraceIdManager` touch different files than the serializer/masker.
- Polish: T070, T071, T072, T073, T074, T075 are all `[P]` — different files, different concerns.

---

## Parallel Example: User Story 1

```bash
# Launch all US1 unit tests together (write them all before any implementation):
Task: "T010 [P] [US1] MethodLogMatcherTest in src/test/java/io/github/cerovskimatija/debugtrace/aspect/MethodLogMatcherTest.java"
Task: "T011 [P] [US1] LoggingContextTest in src/test/java/io/github/cerovskimatija/debugtrace/context/LoggingContextTest.java"
Task: "T012 [P] [US1] LogFormatterTest in src/test/java/io/github/cerovskimatija/debugtrace/format/LogFormatterTest.java"

# Then launch the three independent primitives together:
Task: "T015 [P] [US1] LoggingContext in src/main/java/io/github/cerovskimatija/debugtrace/context/LoggingContext.java"
Task: "T016 [P] [US1] MethodLogMatcher in src/main/java/io/github/cerovskimatija/debugtrace/aspect/MethodLogMatcher.java"
Task: "T017 [P] [US1] LogFormatter in src/main/java/io/github/cerovskimatija/debugtrace/format/LogFormatter.java"

# T018 (MethodLoggingAspect) waits on T015/T016/T017.
# T019 (autoconfig wiring) waits on T018.
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Phase 1 (Setup) — bootstrap Maven, autoconfig imports file, logger constant.
2. Phase 2 (Foundational) — `DebugTraceProperties`, empty `DebugTraceAutoConfiguration`, baseline IT.
3. Phase 3 (US1) — matcher, context, formatter, aspect, autoconfig wiring.
4. **STOP and VALIDATE**: run `MethodLoggingAspectIT`, then run the live Phase 1 acceptance demo from `PRD-phases.md` against the sample app skeleton. If both pass, MVP is shippable as `0.1.0-SNAPSHOT`.

### Incremental Delivery

1. Setup + Foundational → foundation ready.
2. + US1 → MVP `0.1.0-SNAPSHOT`: nested method chain visible.
3. + US2 → `0.2.0-SNAPSHOT`: inputs and outputs.
4. + US3 → `0.3.0-SNAPSHOT`: exceptions + resilience hardening across all earlier code.
5. + US4 → `0.4.0-SNAPSHOT`: masking on by default.
6. + US5 → `0.5.0-SNAPSHOT`: HTTP brackets + trace ID.
7. + US6 → `0.6.0-SNAPSHOT`: HTTP headers + bodies (opt-in).
8. + US7 → `0.7.0-SNAPSHOT`: slow-only + repo toggle + perf budget verified.
9. + Polish → `1.0.0`: sample app, README, Javadoc, Maven Central wiring, CI matrix.

Each tagged version is a credible stopping point — a user installing at any version still gets a working, safe-by-default library.

### Parallel Team Strategy

With multiple developers (after Phase 2 completes):

- Developer A: US1 → US2 → US4 (the serializer / masker stack).
- Developer B: US5 → US6 (the HTTP / trace-id stack — can start in parallel with A once US1 is in `main`).
- Developer C: US3 (resilience audit across A's code) → US7 (tuning + perf benchmark).
- Polish phase is split: A owns the sample app, B owns the README, C owns Maven Central wiring + CI.

---

## Notes

- `[P]` tasks operate on different files with no incomplete-task dependency — safe to run in parallel.
- `[Story]` labels (`US1`–`US7`) trace every task back to a `spec.md` user story and a PRD-phases.md phase.
- Test files are created BEFORE implementation files in every user story; assert they fail before moving on (Principle VI red-then-green).
- Commit at task boundaries or at the end of each user story checkpoint — both are sane points (the `before_*` / `after_*` hooks in `.specify/extensions.yml` will offer this automatically).
- Constitution Principle I ("Safety First") applies to every implementation task — wrap external calls into the library's own serializer / masker / formatter / SLF4J in `try`/`catch (Throwable)` with the documented fallback, and rethrow user exceptions unchanged.
- Avoid pulling work forward across phase boundaries: each PRD-phases.md phase is sized to land one acceptance demo. Skipping ahead loses the demo as the integration assertion for the prior layer.
