# Contract: Configuration Properties

**Date**: 2026-05-28
**Branch**: `001-v1-starter`
**Scope**: The full `spring-debug-trace.*` property tree consumed by `DebugTraceProperties` (Phase 0 stub → grown through every phase).

This document is the canonical schema for the library's external interface. It defines, for every property:

- **Type** — the Java type Spring binds to.
- **Default** — value used when the property is absent.
- **Phase** — the `PRD-phases.md` phase that introduces or activates the property.
- **PRD ref** — the FR(s) and PRD §10 line(s) that govern it.
- **Behavior** — what changes when the value is non-default.
- **Validation** — accepted ranges; behavior on out-of-range / empty / null input.

All properties bind through one `@ConfigurationProperties("spring-debug-trace")` class with nested static classes per area (Constitution Principle VII). `spring-boot-configuration-processor` is on the build classpath so the IDE metadata `META-INF/spring-configuration-metadata.json` is generated automatically.

---

## Top-level

### `spring-debug-trace.enabled`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `false` |
| **Phase** | 0 (gate property) / 1 (effective) |
| **PRD ref** | FR-2, FR-001, FR-037, §10, §24 decision 2 |
| **Behavior** | `false` → no aspect, no filter, no MDC writes from the library. `true` → enables conditionally-registered beans per other properties. |
| **Validation** | None — Spring's `Boolean.parseBoolean`. Missing → `false`. |

### `spring-debug-trace.base-packages`

| | |
|---|---|
| **Type** | `List<String>` |
| **Default** | `[]` (empty) |
| **Phase** | 1 |
| **PRD ref** | FR-3, FR-002, §10 |
| **Behavior** | Methods are intercepted only when their declaring class's package starts with one of these prefixes (string-prefix match, dot-separated). Empty list → no method logging emitted even when `enabled=true` (a `WARN` line is logged once at startup). |
| **Validation** | Null entries dropped. Entries trimmed. Order does not matter for matching. |

### `spring-debug-trace.excluded-packages`

| | |
|---|---|
| **Type** | `List<String>` |
| **Default** | `[]` (user list); the matcher always also applies these built-in defaults non-overridable: `org.springframework`, `org.hibernate`, `jakarta.servlet`, `javax.servlet`, `com.fasterxml.jackson`, `org.slf4j`, `ch.qos.logback`, plus `io.github.cerovskimatija.debugtrace` (library self-exclusion). |
| **Phase** | 1 |
| **PRD ref** | FR-3, FR-004, FR-005, §10 |
| **Behavior** | User entries APPEND to the built-in defaults; they do NOT replace them. A class whose package matches any default OR user entry is skipped, even if it also matches `base-packages`. |
| **Validation** | Same as `base-packages`. |

### `spring-debug-trace.excluded-class-name-patterns`

| | |
|---|---|
| **Type** | `List<String>` (regex) |
| **Default** | `[".*Configuration", ".*Properties"]` (built-in regex defaults applied out-of-the-box; user entries APPEND, they do NOT replace — same merge semantics as `excluded-packages`) |
| **Phase** | 7 |
| **PRD ref** | FR-5, §10 |
| **Behavior** | A class whose fully-qualified name matches any of these regular expressions is skipped. Evaluated after package checks but before the base-package include check. The defaults silence Spring Boot configuration and properties classes — adopters who *want* to see their `@ConfigurationProperties` beans logged must override the list (and the README troubleshooting section calls this out). |
| **Validation** | Invalid regex → ignored with a `WARN` log line at startup; the rest of the list still applies. |

> v1 does not expose a `format` property. The library emits the pretty/human-readable shape from §2 of `log-format.md` only; a structured (JSON) format is a post-v1 enhancement (PRD §22) and adding the configuration key now would be scaffolding for a non-goal (Constitution Principle III).

---

## `spring-debug-trace.method.*`

### `method.log-input`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` (per PRD §10) |
| **Phase** | 2 |
| **PRD ref** | FR-6, FR-012, §10 |
| **Behavior** | `true` → entry lines include `(input=[...])` with arguments serialized through `SafeLogSerializer`. `false` → entry lines omit `(input=...)`. |
| **Validation** | Standard `Boolean`. |

### `method.log-output`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` (per PRD §10) |
| **Phase** | 2 |
| **PRD ref** | FR-7, FR-013, §10 |
| **Behavior** | `true` → exit lines include `(output=...)`. Return values of `void` show as no output segment; `null` shows as `(output=null)`. `false` → exit lines show duration only. |
| **Validation** | Standard `Boolean`. |

### `method.log-exceptions`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` (per PRD §10) |
| **Phase** | 3 |
| **PRD ref** | FR-8, FR-017, §10 |
| **Behavior** | `true` → exception line emitted with class + message + duration (`×` marker). `false` → exception is rethrown unchanged but no library log line is produced for it. |
| **Validation** | Standard `Boolean`. |

### `method.log-exception-stack-trace`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `false` |
| **Phase** | 3 |
| **PRD ref** | FR-017, spec Clarifications Q2 |
| **Behavior** | `true` → the `Throwable` is passed as the last argument to `log.debug(..., throwable)` so the user's SLF4J appender prints the stack trace. `false` → only class + message in the log line (no stack). The exception is always rethrown unchanged regardless of this flag. |
| **Validation** | Standard `Boolean`. |

### `method.log-only-slow`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `false` |
| **Phase** | 7 |
| **PRD ref** | FR-16, FR-035, §10 |
| **Behavior** | `true` → entry lines are buffered and emitted (with duration) only when the method's elapsed time exceeds `slow-threshold-ms`. Methods completing under the threshold produce zero log lines (no entry, no exit). Exceptions still emit (when `log-exceptions=true`) regardless of duration. |
| **Validation** | Standard `Boolean`. |

### `method.slow-threshold-ms`

| | |
|---|---|
| **Type** | `long` (ms) |
| **Default** | `500` |
| **Phase** | 7 |
| **PRD ref** | FR-16, §10 |
| **Behavior** | Only used when `method.log-only-slow=true`. |
| **Validation** | Negative values clamped to `0`. |

### `method.include-repositories`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` |
| **Phase** | 7 |
| **PRD ref** | FR-036, §10, §24 decision 10 |
| **Behavior** | `false` → Spring `@Repository` beans (and types implementing `org.springframework.data.repository.Repository`) are skipped by `MethodLogMatcher`. |
| **Validation** | Standard `Boolean`. |

---

## `spring-debug-trace.http.*`

### `http.enabled`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` (per PRD §10) — but the filter only registers when the Servlet API is on the classpath AND `spring-debug-trace.enabled=true`. |
| **Phase** | 5 |
| **PRD ref** | FR-9, FR-025, FR-026, §10 |
| **Behavior** | `true` + Servlet API present → `HttpLoggingFilter` is registered and emits HTTP entry/exit lines for every request. `false` → filter not registered. When Servlet API is absent, the filter is silently not registered regardless of this property. |
| **Validation** | Standard `Boolean`. |

### `http.log-headers`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `false` |
| **Phase** | 6 |
| **PRD ref** | FR-10, FR-027, §10, §14 |
| **Behavior** | `true` → request and response headers are logged. Sensitive headers (`Authorization`, `Cookie`, `Set-Cookie` plus any matching the field-name masking list, case-insensitive) are masked through `SensitiveDataMasker`. |
| **Validation** | Standard `Boolean`. |

### `http.log-request-body`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `false` |
| **Phase** | 6 |
| **PRD ref** | FR-11, FR-028, §10, §14 |
| **Behavior** | `true` → `ContentCachingRequestWrapper` captures the body, the filter emits it on the HTTP entry line truncated to `max-body-length`, with best-effort JSON masking applied (R5). |
| **Validation** | Standard `Boolean`. |

### `http.log-response-body`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `false` |
| **Phase** | 6 |
| **PRD ref** | FR-11, FR-029, §10, §14 |
| **Behavior** | `true` → `ContentCachingResponseWrapper` captures the body, the filter emits it on the HTTP exit line, `copyBodyToResponse()` is invoked before the wrapper unwinds (Constitution Principle I; SC-006). |
| **Validation** | Standard `Boolean`. |

### `http.max-body-length`

| | |
|---|---|
| **Type** | `int` (chars) |
| **Default** | `5000` |
| **Phase** | 6 |
| **PRD ref** | FR-11, §10 |
| **Behavior** | Caps the captured-body length in characters for the log line. The client still receives the full body. Truncation adds a `... (truncated)` suffix. |
| **Validation** | Negative or zero → no body logged even when the flags are on (treated as `0` cap). |

### `http.skip-binary-content`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` |
| **Phase** | 6 |
| **PRD ref** | FR-11, FR-028, §10, §14 |
| **Behavior** | `true` → bodies with binary `Content-Type` (e.g. `application/octet-stream`, `image/*`, `audio/*`, `video/*`, `application/pdf`) are summarized as `<binary, N bytes>` rather than logged. |
| **Validation** | Standard `Boolean`. |

### `http.skip-multipart-content`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` |
| **Phase** | 6 |
| **PRD ref** | FR-11, §10, §14 |
| **Behavior** | `true` → multipart bodies are summarized as `<multipart, N parts>`. The upload still reaches the controller intact. |
| **Validation** | Standard `Boolean`. |

---

## `spring-debug-trace.trace-id.*`

### `trace-id.enabled`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` |
| **Phase** | 5 |
| **PRD ref** | FR-12, FR-031, §10 |
| **Behavior** | `true` → `TraceIdManager` resolves a value per request and `LogFormatter` prepends `[traceId=...]`. `false` → MDC is not touched by the library and no prefix is added. |
| **Validation** | Standard `Boolean`. |

### `trace-id.mdc-key`

| | |
|---|---|
| **Type** | `String` |
| **Default** | `traceId` |
| **Phase** | 5 |
| **PRD ref** | FR-12, FR-032, §10 |
| **Behavior** | MDC slot where the resolved trace ID is stored and read. |
| **Validation** | Blank / null → falls back to `traceId`. |

### `trace-id.generate-if-missing`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` |
| **Phase** | 5 |
| **PRD ref** | FR-12, FR-032, §10 |
| **Behavior** | `true` → when neither MDC nor any configured request header yields a value, a `UUID.randomUUID().toString()` is generated. `false` → no trace ID for that request; the prefix is omitted. |
| **Validation** | Standard `Boolean`. |

### `trace-id.request-headers`

| | |
|---|---|
| **Type** | `List<String>` |
| **Default** | `[X-Request-Id, X-Correlation-Id, traceparent]` |
| **Phase** | 5 |
| **PRD ref** | FR-12, FR-032, §10 |
| **Behavior** | Header names checked in order; first non-blank value becomes the trace ID. Matching is case-insensitive (servlet API normalizes header names). |
| **Validation** | Null entries dropped. Empty list → only MDC and generation are consulted. |

---

## `spring-debug-trace.serialization.*`

### `serialization.max-depth`

| | |
|---|---|
| **Type** | `int` |
| **Default** | `3` |
| **Phase** | 2 |
| **PRD ref** | FR-13, FR-016, §10 |
| **Behavior** | Maximum recursion depth into nested objects. Beyond → `<truncated, depth>`. |
| **Validation** | Values < 1 clamped to `1`. |

### `serialization.max-string-length`

| | |
|---|---|
| **Type** | `int` (chars) |
| **Default** | `1000` |
| **Phase** | 2 |
| **PRD ref** | FR-13, FR-016, §10 |
| **Behavior** | Cap on individual string values. Longer strings cut with `... (truncated)` suffix. |
| **Validation** | Values < 0 clamped to `0` (empty rendered string). |

### `serialization.max-collection-size`

| | |
|---|---|
| **Type** | `int` (elements) |
| **Default** | `20` |
| **Phase** | 2 |
| **PRD ref** | FR-13, FR-016, §10 |
| **Behavior** | Only the first N elements of a `Collection`, `Map`, or array are walked. Remainder summarized as `... (+M more)`. |
| **Validation** | Values < 0 clamped to `0`. |

### `serialization.max-object-length`

| | |
|---|---|
| **Type** | `int` (chars in the rendered representation) |
| **Default** | `5000` |
| **Phase** | 2 |
| **PRD ref** | FR-13, FR-016, §10 |
| **Behavior** | Total output cap for a single argument or return value rendering. Exceeding → walk terminates with `... (object truncated)`. |
| **Validation** | Values < 100 clamped to `100`. |

### `serialization.include-null-fields`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `false` |
| **Phase** | 2 |
| **PRD ref** | §10 |
| **Behavior** | When `false`, DTO fields whose value is `null` are omitted from the rendered output. When `true`, they appear as `field=null`. |
| **Validation** | Standard `Boolean`. |

---

## `spring-debug-trace.masking.*`

### `masking.enabled`

| | |
|---|---|
| **Type** | `boolean` |
| **Default** | `true` |
| **Phase** | 4 |
| **PRD ref** | FR-14, FR-021, §10, §14 |
| **Behavior** | `false` → field-name masking and header masking are off. JSON body masking is also off. Strongly discouraged in any non-local environment (the README will say so). |
| **Validation** | Standard `Boolean`. |

### `masking.replacement`

| | |
|---|---|
| **Type** | `String` |
| **Default** | `****` |
| **Phase** | 4 |
| **PRD ref** | FR-024, §10 |
| **Behavior** | The placeholder substituted for masked values. Applies uniformly to DTO fields, headers, and JSON-body keys. |
| **Validation** | Null → falls back to `****`. |

### `masking.fields`

| | |
|---|---|
| **Type** | `List<String>` |
| **Default** | `[password, pass, token, accessToken, refreshToken, authorization, cookie, secret, apiKey, cardNumber, iban]` (built-in defaults; user values APPEND, they do NOT replace — Constitution Principle II, FR-022). |
| **Phase** | 4 |
| **PRD ref** | FR-14, FR-022, FR-023, §10, §14 |
| **Behavior** | All entries (defaults + user) are lowercased at construction. Field-name matching is case-insensitive (FR-023). |
| **Validation** | Null / blank entries dropped. |

---

## Invalid configuration handling

The library is loud at startup, quiet at runtime:

1. **At application startup**, any malformed regex, unparseable enum value, or empty `base-packages` when `enabled=true` produces ONE `WARN` line through the dedicated logger.
2. **At runtime**, configuration is never re-read; `@ConfigurationProperties` semantics. Invalid values fall back to their defaults silently because the bind happened at startup.
3. **At no point** does invalid configuration prevent the application from starting. The library degrades to "do less, log nothing" rather than failing.

---

## Cross-references

- See `data-model.md` E5 for how `masking.*` populates the rule set.
- See `data-model.md` E6 for how the package/regex properties feed `MethodLogMatcher`.
- See `data-model.md` E7 for how the `serialization.*` properties shape `SafeLogSerializer`.
- See `log-format.md` for what each property's behavior changes in the visible log.
- See `quickstart.md` for the four-property minimum-viable configuration from SC-009.
