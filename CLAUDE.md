# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository State

This repository is in **planning / pre-implementation state**. The only files are three planning documents and a `.gitignore` — there is no `pom.xml`, no `src/`, no build configuration yet. Phase 0 of `PRD-phases.md` is what produces the Maven build; build/test commands will be added to this file once that lands.

The three docs:

- **[PRD.md](./PRD.md)** — full v1 product spec: functional requirements, config tree, component responsibilities, acceptance criteria.
- **[PRD-phases.md](./PRD-phases.md)** — v1 work split into 9 incremental phases (0–8). The phase order is load-bearing — read this before writing code.
- **[README.md](./README.md)** — user-facing positioning, intended install snippet, example output.

## Project

`spring-debug-trace-starter` is a Spring Boot 3+ auto-configuration library that adds nested method-chain debug logging to user code via Spring AOP — no annotations required. User adds the dependency, sets `spring-debug-trace.enabled=true` and `spring-debug-trace.base-packages`, and gets indented `→`/`←` logs of bean method calls, with optional inputs, outputs, exceptions, HTTP context, and trace-ID correlation.

Core promise: *Add dependency. Configure base package. See your Spring bean method chain.*

## Fixed Identifiers (Keep Consistent Everywhere)

These are referenced across PRD, README, and intended config — drift will break user installs:

- Maven coordinates: `io.github.cerovskimatija:spring-debug-trace-starter`
- Root Java package: `io.github.cerovskimatija.debugtrace`
- Dedicated logger name: `io.github.cerovskimatija.debugtrace` (emits at `DEBUG`)
- Configuration prefix: `spring-debug-trace`
- Java baseline: 17 (compatibility-test on 21)
- Spring Boot baseline: 3.x

## Architectural Intent

The component map lives in PRD §11. Key couplings that take more than one file to see:

- **Aspect + LoggingContext + LogFormatter are coupled.** `MethodLoggingAspect` increments/decrements `LoggingContext` depth around `proceed()`; `LogFormatter` reads that depth to indent. Both the success branch and the exception branch must decrement depth and clean root thread-local state — this is easy to get wrong.
- **`SafeLogSerializer` is the only place user-domain objects are touched.** It walks DTOs reflectively and consults `SkipTypeDetector` (servlet/IO/multipart/Principal/etc. → safe placeholder) and `SensitiveDataMasker` (field-name match → `****`). It must never throw; every call site wraps it and falls back to `<serialization-failed: ClassName>`.
- **`TraceIdManager` owns MDC lifecycle.** Reads existing MDC first, then configured request headers, then generates a UUID. Cleans up only what it put there. The MDC trace ID is the join key for HTTP logs and method logs — both `HttpLoggingFilter` and `LogFormatter` consume it.
- **`HttpLoggingFilter` is conditional on the Servlet API.** Method logging must work without `spring-web`. Use `@ConditionalOnClass(jakarta.servlet.Filter)` or equivalent — never a hard dependency.

## Invariants That Constrain Every Change

These come from PRD §6/§14/§17 and shape almost every file:

1. **The library must never break the user application.** All serialization, masking, formatting, and logging calls are wrapped in try/catch with safe fallbacks. Exceptions from intercepted user methods are rethrown unchanged — never wrapped, never swallowed.
2. **Disabled by default.** No aspect bean, no filter bean when `enabled=false`. Disabled-mode overhead must be near-zero (Phase 7 enforces this).
3. **Safe-by-default for sensitive data.** Masking on; HTTP headers off; HTTP bodies off. Default sensitive field list lives in `SensitiveDataMasker`; user-defined `masking.fields` **append** to defaults, they don't replace them.
4. **Library self-exclusion.** `MethodLogMatcher` must always exclude `io.github.cerovskimatija.debugtrace.*` to prevent recursive self-logging. Spring, Hibernate, Jackson, Servlet, SLF4J, Logback are excluded by default.
5. **Spring AOP only in v1.** No AspectJ load-time weaving, no Java agents, no bytecode instrumentation. Self-invocation and private methods are accepted limitations — document them, don't try to work around them.
6. **Response delivery is sacred.** When body logging is on, the content-caching response wrapper must call `copyBodyToResponse()` so the client still receives the full unmodified response.

## How to Approach Implementation Work

1. **Find the phase.** Identify which phase in `PRD-phases.md` the requested work belongs to. Phases are ordered to minimize risk — don't pull work forward across phase boundaries unless asked.
2. **Cross-reference the PRD.** Each phase lists the PRD sections (FR-#, §#) it covers. Read those for acceptance criteria and edge cases before writing code.
3. **Keep phases demoable.** Each phase ends with an "Acceptance demo" — the work for that phase isn't done until that demo runs.
4. **Defer non-goals.** PRD §6 lists explicit v1 non-goals (WebFlux body, OpenTelemetry, async context propagation, JSON format, etc.). Don't add scaffolding for them — they're tracked in PRD §22 for future versions.

## PRD Sections to Re-Read by Task Type

`PRD.md` is ~1200 lines. Shortlists for common change types:

- Config / properties work → §10 (full property tree)
- Aspect / matching / nesting → FR-3, FR-4, FR-5, §11
- Serialization / I/O logging → FR-6, FR-7, FR-13
- Masking → FR-14, §14
- HTTP logging → FR-9, FR-10, FR-11
- Trace IDs → FR-12
- Error handling → FR-8, §17
- Visual log format → §8 (example output is the visual contract)
- "What's in v1?" decisions → §6 (non-goals), §21 (acceptance), §24 (resolved decisions — trust these unless explicitly revisited)

<!-- SPECKIT START -->
For additional context about technologies to be used, project structure,
shell commands, and other important information, read the current plan
at [specs/001-v1-starter/plan.md](./specs/001-v1-starter/plan.md). Companion
artifacts in the same directory: `spec.md`, `research.md`, `data-model.md`,
`quickstart.md`, and `contracts/{configuration-properties,log-format,public-api}.md`.
<!-- SPECKIT END -->
