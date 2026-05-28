<!--
SYNC IMPACT REPORT
==================
Version change: 1.0.0 → 1.0.1
Rationale: PATCH clarification. Technical Standards & Build Baseline
previously listed `jackson-databind` under "Direct dependencies (compile
scope)", which contradicted spec.md Assumption §263 ("Jackson Databind is
NOT a hard runtime dependency"). This edit moves the dependency under
"Optional (provided/optional scope)" to make the constitution and the spec
consistent. No principle is added, removed, or reversed; no fixed
identifier in Principle IV changes — hence PATCH, not MINOR or MAJOR.

Principles unchanged (1.0.0):
  I.   Safety First — Never Break the User Application (NON-NEGOTIABLE)
  II.  Safe-By-Default Configuration
  III. Phase-Gated, PRD-Anchored Scope
  IV.  Stable Public Surface & Fixed Identifiers
  V.   Spring AOP Only; Accepted Limitations Are Documented, Not Worked Around
  VI.  Test Discipline — JUnit 5 Unit + Spring Integration Coverage Per Phase
  VII. Idiomatic Spring Boot Conventions

Sections changed in 1.0.1:
  - Technical Standards & Build Baseline — Jackson moved compile → optional.

Templates requiring updates:
  - .specify/templates/*                     ✅ unaffected (no principle
    or identifier change)
  - CLAUDE.md                                ✅ unaffected
  - README.md                                ✅ unaffected
  - specs/001-v1-starter/plan.md             ✅ updated alongside (Jackson
    moved to optional in "Primary Dependencies")
  - specs/001-v1-starter/research.md         ✅ updated alongside (R3
    narrative reconciled — Jackson optional, regex masking confirmed)
  - specs/001-v1-starter/spec.md             ✅ updated alongside
    (Assumption §263 wording clarified — Jackson is optional-scope)
  - specs/001-v1-starter/tasks.md            ✅ updated alongside
    (T001 declares jackson-databind optional, not compile)

Deferred items: none.

Prior versions:
  1.0.0 (2026-05-28) — initial ratification.
-->

# Spring Debug Trace Starter Constitution

## Core Principles

### I. Safety First — Never Break the User Application (NON-NEGOTIABLE)

The library MUST NOT alter the behavior or stability of the host application
under any input. Concretely:

- Every call into `SafeLogSerializer`, `SensitiveDataMasker`, `LogFormatter`,
  and the underlying SLF4J logger MUST be wrapped in `try`/`catch` with a
  safe fallback (e.g., `<serialization-failed: ClassName>`). A logger fault
  MUST NOT propagate.
- Exceptions thrown by intercepted user methods MUST be rethrown unchanged —
  never wrapped, never swallowed, never reclassified.
- Both the success and exception branches of `MethodLoggingAspect` MUST
  decrement `LoggingContext` depth and clean root thread-local state.
- Response delivery is sacred: when HTTP body logging is enabled,
  `ContentCachingResponseWrapper.copyBodyToResponse()` MUST be invoked so the
  client receives the full, unmodified response byte-for-byte.

**Rationale:** This library installs cross-cutting interception in someone
else's production code. A logger that causes outages is worse than no logger.
PRD §17 and PRD invariant 1 make this the highest priority.

### II. Safe-By-Default Configuration

Defaults MUST favor minimum surface area and minimum sensitive-data exposure:

- `spring-debug-trace.enabled` defaults to `false`. No aspect bean, no HTTP
  filter bean is registered when disabled.
- `masking.enabled` defaults to `true`. The default sensitive-field list is
  the canonical one in PRD §10. User-supplied `masking.fields` **append to**
  the defaults — they MUST NOT replace them.
- `http.log-headers`, `http.log-request-body`, `http.log-response-body`
  default to `false`. Sensitive headers (`Authorization`, `Cookie`,
  `Set-Cookie`) MUST be masked when header logging is enabled.
- Binary and multipart bodies MUST be skipped/summarized by default.
- Disabled-mode overhead MUST be near-zero; the aspect MUST short-circuit at
  its entry point when `enabled=false`.

**Rationale:** This library can dump arbitrary user data into logs. Conservative
defaults limit the blast radius of misconfiguration. PRD §14 and invariants 2/3.

### III. Phase-Gated, PRD-Anchored Scope

All work MUST be traceable to a phase in `PRD-phases.md` and a requirement in
`PRD.md`. Phase ordering is load-bearing:

- A change MUST identify its phase (0–8) and the PRD section(s) it implements
  before code is written.
- Work MUST NOT pull deliverables forward across phase boundaries unless
  explicitly requested in writing.
- v1 non-goals listed in PRD §6 (WebFlux body logging, OpenTelemetry,
  AspectJ load-time weaving, Java agent instrumentation, async context
  propagation, JSON log format, distributed tracing, automatic query tracing,
  etc.) MUST NOT receive scaffolding, configuration keys, or extension points
  in v1. They belong to PRD §22.
- A phase is "done" only when its Acceptance Demo in `PRD-phases.md`
  runs end-to-end.

**Rationale:** The phase order minimizes integration risk and keeps the
library in a "show a friend" state after Phase 1. Phase skipping reintroduces
the risk the ordering was designed to remove.

### IV. Stable Public Surface & Fixed Identifiers

The following identifiers form the user-visible contract and MUST NOT drift
across PRD, README, code, or configuration:

- Maven coordinates: `io.github.cerovskimatija:spring-debug-trace-starter`
- Root Java package: `io.github.cerovskimatija.debugtrace`
- Dedicated logger name: `io.github.cerovskimatija.debugtrace`
- Configuration prefix: `spring-debug-trace`
- Auto-configuration registration path:
  `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- Java baseline: 17 (compatibility-tested on 21)
- Spring Boot baseline: 3.x

Any change to these identifiers is a breaking change and requires a MAJOR
constitution amendment plus coordinated edits in PRD, README, CLAUDE.md,
sample app, and published Maven metadata.

**Rationale:** Drift in any of these breaks user installs silently. A user
who pasted `spring-debug-trace.enabled=true` into `application.yml` should
never have to debug "why is no aspect activating" because the prefix moved.

### V. Spring AOP Only; Accepted Limitations Are Documented, Not Worked Around

v1 uses Spring AOP exclusively. The following MUST hold:

- No AspectJ load-time weaving, no Java agents, no bytecode instrumentation,
  no compile-time weaving.
- Self-invocation (`this.method()` inside the same bean) and private methods
  are accepted limitations. They MUST be documented in the README
  Troubleshooting section and MUST NOT receive workarounds (no
  `AopContext.currentProxy()` recipes baked into the library, no LTW fallback).
- `MethodLogMatcher` MUST always exclude `io.github.cerovskimatija.debugtrace.*`
  to prevent recursive self-logging. The default-exclude list MUST also cover
  `org.springframework`, `org.hibernate`, `jakarta.servlet`, `javax.servlet`,
  `com.fasterxml.jackson`, `org.slf4j`, `ch.qos.logback`.
- `HttpLoggingFilter` MUST be guarded by `@ConditionalOnClass(jakarta.servlet.Filter)`
  (or equivalent). Method logging MUST work without `spring-web` on the
  classpath.

**Rationale:** AOP is enough for the v1 user promise and keeps the dependency
footprint small. Trying to extend Spring AOP's reach with proxy gymnastics
produces fragile, surprising behavior — better to document the boundary.

### VI. Test Discipline — JUnit 5 Unit + Spring Integration Coverage Per Phase

Every phase MUST land with executable tests in `src/test/java`. Tests are
not optional for this project — they are the only mechanism that catches
regressions in cross-cutting interception. Concretely:

- Unit tests MUST use JUnit 5 (`junit-jupiter-api/engine/params`). Test
  classes MUST be suffixed `Test`. Tests MUST follow Arrange–Act–Assert and
  be independent and idempotent.
- Integration tests for the auto-configuration MUST use `@SpringBootTest`
  (or a `WebApplicationContextRunner`/`ApplicationContextRunner` for
  conditional-bean verification) against a real Spring context — not a hand-
  rolled mock context.
- The minimum unit test set from PRD §20 (`MethodLogMatcherTest`,
  `SensitiveDataMaskerTest`, `SafeLogSerializerTest`, `LoggingContextTest`,
  `TraceIdManagerTest`, `LogFormatterTest`) MUST exist by end of the phase
  that introduces the corresponding component.
- The integration suite from PRD §20 (auto-config loads/skips per `enabled`,
  controller-service chain logging, excluded packages silent, masking
  end-to-end, exceptions logged + rethrown, HTTP metadata, HTTP body logging
  without breaking response delivery) MUST be green before v1.0.0 is tagged.
- The CI matrix MUST run on Java 17 and Java 21 against the latest Spring
  Boot 3.x release.

**Rationale:** The library mutates cross-cutting behavior; manual smoke
testing cannot catch the matrix of "AOP target × serialization shape ×
masking × HTTP variant". PRD §20 is the floor, not the ceiling.

### VII. Idiomatic Spring Boot Conventions

Code MUST follow standard Spring Boot 3 idioms so contributors and downstream
debuggers recognize the patterns immediately:

- **Dependency injection:** constructor injection only. Dependency fields
  MUST be `private final`. Field and setter injection are prohibited.
- **Configuration binding:** all `spring-debug-trace.*` keys MUST be bound
  through a `@ConfigurationProperties("spring-debug-trace")` class
  (`DebugTraceProperties`). String-keyed `Environment` lookups in business
  code are prohibited. The
  `spring-boot-configuration-processor` MUST be on the build so IDE metadata
  is generated.
- **Conditional registration:** beans MUST use `@ConditionalOnProperty`,
  `@ConditionalOnClass`, and `@ConditionalOnMissingBean` as appropriate so
  the library composes cleanly with user-provided beans.
- **Logging:** SLF4J only. Loggers MUST be declared
  `private static final Logger log = LoggerFactory.getLogger(<Class>.class);`
  and use parameterized messages (`log.debug("... {} ...", value)`) — never
  string concatenation. All library-emitted method/HTTP trace logs MUST go
  through the dedicated logger named `io.github.cerovskimatija.debugtrace`
  at `DEBUG` level; internal warnings emit at `WARN`.
- **Javadoc:** all public and protected types and members MUST have Javadoc.
  The first sentence MUST be a complete summary ending with a period.
  `@param`, `@return`, and `@throws` MUST be used where applicable. Complex
  package-private and private members SHOULD also be documented.
- **Package layout:** by feature/responsibility (`aspect`, `properties`,
  `serializer`, `masking`, `http`, `trace`, `format`, `context`,
  `autoconfigure`), not by Spring stereotype.

**Rationale:** A starter library is read more than it is written. Idiomatic
code lowers the barrier for users debugging unexpected behavior and for
future contributors. The java-springboot and java-docs skill guidance is
adopted verbatim where applicable to a starter (and intentionally not where
it applies only to applications, e.g., JPA / Spring Security / Docker
Compose setup).

## Technical Standards & Build Baseline

- **Build tool:** Maven. Single module in v1 (per PRD §24 decision 11);
  splitting into core/starter modules is deferred until justified by size.
- **Java toolchain:** baseline source/target Java 17; CI MUST additionally
  verify Java 21 compatibility.
- **Spring Boot:** managed via `spring-boot-dependencies` BOM at the latest
  3.x at time of release. Spring Boot 2.x is NOT supported.
- **Direct dependencies (compile scope):** `spring-boot-starter-aop`,
  `spring-boot-autoconfigure`, `slf4j-api`. `spring-web` and
  `jackson-databind` are declared `<optional>true</optional>` — present at
  build time but NOT propagated transitively to adopters. `jackson-databind`
  is optional because spec.md Assumption §263 requires it not be a hard
  runtime dependency; v1 does not import it anywhere (Phase 6 body masking
  uses regex per R5).
- **Build dependencies (provided/optional scope):**
  `spring-boot-configuration-processor` (annotation processor for IDE
  metadata).
- **Test dependencies:** `spring-boot-starter-test` (brings JUnit 5,
  AssertJ, Mockito, Spring Test). Additional libraries (e.g., Testcontainers)
  are allowed only if a phase explicitly needs them.
- **Distribution:** Maven Central via the standard publishing pipeline
  (GPG-signed artifacts, source + javadoc jars, `LICENSE` packaged). Set up
  in Phase 8.
- **Out of scope as compile-time dependencies:** Spring WebFlux, Spring
  Security, OpenTelemetry, Micrometer, AspectJ runtime, any database driver.
- **Performance budget:** when disabled, per-call overhead MUST be measured
  in the Phase 7 microbenchmark and MUST be statistically indistinguishable
  from the baseline (no-aspect) loop. When enabled with input/output logging
  off, per-call overhead MUST stay within one order of magnitude of the
  baseline. Phase 7 owns the harness and the published numbers.

## Development Workflow & Quality Gates

- **Phase discipline:** every change set declares the phase number from
  `PRD-phases.md` it advances. PRs that cross phases MUST explain why.
- **Constitution Check:** the `/speckit-plan` Constitution Check gate MUST
  enumerate which principles (I–VII) the planned change touches and how it
  complies. Any deviation is recorded in the plan's Complexity Tracking
  table with justification and the rejected simpler alternative.
- **Code review checklist (all PRs):**
  1. Principle I: every new external call wrapped in `try`/`catch` with a
     defined fallback.
  2. Principle II: any new property defaults to the safest value; user
     extensions append to defaults.
  3. Principle IV: no drift in fixed identifiers.
  4. Principle V: no new dependency on AspectJ LTW, agents, or bytecode
     tooling; library self-exclusion list unchanged or only extended.
  5. Principle VI: tests added for new logic; integration test added when
     the change affects auto-config wiring or HTTP behavior.
  6. Principle VII: constructor injection, `private final` fields,
     parameterized SLF4J logging, Javadoc on new public/protected API.
- **Acceptance demos:** the Acceptance Demo listed for the phase MUST be
  reproducible against the sample app in `examples/spring-boot-demo` before
  the phase is marked complete.
- **Release gate (v1.0.0):** PRD §21 acceptance criteria all green; CI
  matrix (Java 17 + 21) green; sample app demonstrates documented output;
  Maven Central publishing dry-run succeeds.

## Governance

This constitution supersedes ad-hoc conventions and informal preferences.
Where this document and `CLAUDE.md` overlap (safety invariants, fixed
identifiers, AOP-only scope, self-exclusion), `CLAUDE.md` is the runtime
guidance restatement — both MUST remain consistent. If they disagree, this
constitution wins and `CLAUDE.md` MUST be updated in the same change.

**Amendment procedure:**

1. Open a PR that edits this file and clearly states the proposed change,
   the principle(s) affected, and the version bump (MAJOR/MINOR/PATCH) with
   rationale.
2. Update any propagating artifacts in the same PR:
   `.specify/templates/plan-template.md`, `.specify/templates/spec-template.md`,
   `.specify/templates/tasks-template.md`, `CLAUDE.md`, `README.md`, and the
   sample app's `application.yml` when relevant.
3. Bump `Version` per semantic versioning and update `Last Amended`.
4. Prepend a Sync Impact Report as an HTML comment at the top of this file
   (replacing the previous one) recording the version change, modified
   principles, added/removed sections, template propagation status, and any
   deferred TODOs.

**Versioning policy:**

- **MAJOR**: a principle is removed, renamed in a backward-incompatible way,
  or its prescription is reversed (e.g., flipping a default, broadening v1
  scope to include a §6 non-goal); a fixed identifier in Principle IV
  changes.
- **MINOR**: a new principle or governance section is added, or an existing
  principle is materially expanded.
- **PATCH**: clarifications, wording fixes, non-semantic refinements,
  example tweaks.

**Compliance review:** at the start of each phase and at every PR review,
contributors verify the change against Principles I–VII and the Quality
Gates checklist above. Reviewers SHOULD reject changes whose justification
relies on "we'll fix it in a later phase" when the fix is feasible now.

Runtime contributor guidance for AI-assisted work lives in
[CLAUDE.md](../../CLAUDE.md). The product spec lives in
[PRD.md](../../PRD.md). The phase plan lives in
[PRD-phases.md](../../PRD-phases.md). When this constitution and any of
those documents diverge, this constitution governs and the other documents
MUST be reconciled.

**Version**: 1.0.1 | **Ratified**: 2026-05-28 | **Last Amended**: 2026-05-28
