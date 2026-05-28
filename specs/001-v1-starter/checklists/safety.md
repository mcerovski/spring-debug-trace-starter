# Safety & Resilience Checklist: Spring Debug Trace Starter v1

**Purpose**: Pre-implementation spec gate — interrogate whether the safety-first requirements (Constitution Principle I, FR-014/015/016/018/019/020/029, SC-003/006/010, US3, the resilience-related Edge Cases) are written precisely enough that an implementer cannot reasonably introduce a regression by following them literally. This checklist tests the **requirements**, not the code.

**Created**: 2026-05-28

**Feature**: [Link to spec.md](../spec.md) · [constitution.md](../../../.specify/memory/constitution.md) · [log-format.md](../contracts/log-format.md)

**Scope**: Safety and resilience only. Sensitive-data masking quality, performance budgets beyond near-zero overhead, public-API stability, and release-gate items are out of scope and belong in their own checklists.

## Requirement Completeness

- [ ] CHK001 - Is every call site the library makes into *its own* helpers (`SafeLogSerializer`, `SensitiveDataMasker`, `LogFormatter`, SLF4J) enumerated somewhere with the fallback text it must produce on failure? [Completeness, Spec §FR-019, Constitution Principle I]
- [ ] CHK002 - Are the cleanup responsibilities for `LoggingContext` depth AND `MDC` trace-ID state defined for **both** the normal-exit branch and the exception branch of the aspect, including the case where the cleanup itself throws? [Completeness, Spec §FR-010, §FR-011, §FR-033]
- [ ] CHK003 - Does the spec define what "library failure" *includes* (serialization, masking, formatting, SLF4J, MDC, HTTP body wrapping) and what it *excludes* (downstream user exceptions, infrastructure faults the application would see anyway)? [Completeness, Gap, Spec §FR-019]
- [ ] CHK004 - Are requirements stated for what happens when `ContentCachingResponseWrapper.copyBodyToResponse()` itself throws — does the library still rethrow the user's original exception, swallow, or surface? [Completeness, Gap, Spec §FR-029]
- [ ] CHK005 - Are requirements defined for the "exception thrown by user code AND simultaneous failure of the library while formatting that exception" path? Spec §17 Edge Cases hints at it; is the contract explicit? [Completeness, Edge Case, Spec §17 last bullet]

## Requirement Clarity

- [ ] CHK006 - Is "rethrow the original exception unchanged" (FR-018) defined precisely — same instance, same cause chain, no `addSuppressed`, no stack-trace mutation? [Clarity, Spec §FR-018]
- [ ] CHK007 - Is "the application's externally observable behavior … MUST be identical" (FR-020) enumerated as a closed list (HTTP status, response bytes, exception type reaching handler, header values, retry counts, ordering of side-effects)? Without an enumeration the requirement is not testable. [Clarity, Spec §FR-020]
- [ ] CHK008 - Is "near-zero runtime overhead per call relative to a baseline" (FR-037) quantified, or does it rely solely on SC-007's ≤ 5 % budget? If both, are they consistent? [Clarity, Spec §FR-037, §SC-007]
- [ ] CHK009 - Is the placeholder grammar `<serialization-failed: ClassName>` specified for the `ClassName` portion — is it the simple name of the value's runtime class, the declared field type, or the failing accessor's owner? [Clarity, Ambiguity, Spec §FR-019, contracts/log-format.md §2.7]

## Requirement Consistency

- [ ] CHK010 - Do the spec (FR-018), the constitution (Principle I bullet 2), and contracts/log-format.md §6 agree on **what** rethrow means when `method.log-exceptions=false`? Each currently uses slightly different phrasing; verify they cannot be read into conflict. [Consistency, Spec §FR-018, Constitution Principle I, log-format.md §6]
- [ ] CHK011 - Are the response-body-unmodified requirements consistent between FR-029 (general), SC-006 (byte-comparison), and the contracts/configuration-properties.md `http.log-response-body` behavior note? In particular, do all three rule out *any* mutation (encoding, charset, length-header recomputation) or only "the bytes the application produced"? [Consistency, Spec §FR-029, §SC-006]
- [ ] CHK012 - Does the contract for "library swallows its own failure" (FR-019) conflict with the test-discipline requirement that internal warnings emit at `WARN` (Constitution Principle VII)? If a serializer call fails 1000 times in a request, is the spec silent on whether the WARN line fires 1000 times or is rate-limited? [Consistency, Gap, Spec §FR-019, Constitution Principle VII]

## Acceptance Criteria Quality (Measurability)

- [ ] CHK013 - Is "byte-for-byte identical" (SC-006) measurable for chunked transfer-encoded responses, gzip-compressed responses, and responses written via `ServletOutputStream.write(byte[], int, int)` partial writes? Without addressing these, the criterion is testable only on trivial responses. [Measurability, Spec §SC-006]
- [ ] CHK014 - Is the SC-003 "every test passes both with the library disabled and with the library enabled" criterion measurable in CI — i.e., does the spec say which "every test" (all unit + integration? only the v1 acceptance suite?) and what tolerance is allowed (none, or `± wall-clock`)? [Measurability, Spec §SC-003]
- [ ] CHK015 - Is SC-010 ("calling request still completes successfully") objectively verifiable — does "successfully" mean `200 OK`, "the controller method returned normally", or "no exception escaped the `@ControllerAdvice` handler"? [Measurability, Spec §SC-010]
- [ ] CHK016 - Is the "depth has been reset to zero" claim (US3.4) measurable from outside the library, given that `LoggingContext` is intentionally package-private? Without an observable handle, the requirement can only be checked by white-box test access. [Measurability, Spec §US3 Scenario 4, §FR-011]

## Scenario Coverage

- [ ] CHK017 - Are requirements specified for the "library + `@ControllerAdvice` exception translation" interaction (US3.2)? In particular, does the spec require the `@ExceptionHandler` to receive the *same instance* the user code threw, or only the *same type*? [Coverage, Spec §US3 Scenario 2, §FR-020]
- [ ] CHK018 - Are requirements specified for the "library + Spring's `@Retryable` (or any user-supplied retry)" interaction — does retry count, retry delay, or failure-after-attempts behavior have to be identical with and without the library? FR-020 hints at it; is it pinned? [Coverage, Gap, Spec §FR-020]

## Edge Case Coverage

- [ ] CHK019 - Are requirements defined for *concurrent depth corruption* — a request's exception path runs `finally { pop(); clearIfRoot(); }` but pop throws, leaving depth > 0 on the thread before it returns to the pool? Spec §17 mentions concurrent requests must not corrupt each other; the specific "cleanup throws" failure mode is not addressed. [Edge Case, Gap, Spec §17 "Concurrent requests", §17 "Very deep call chains"]
- [ ] CHK020 - Are requirements defined for the *self-referential exception cause chain* — `Throwable e1 = new RuntimeException(); e1.initCause(e1)` — when the formatter tries to render the cause chain? `SafeLogSerializer` covers cyclic object graphs (FR-015) but the exception line formatter is a separate code path; the spec does not extend the "terminate cleanly" guarantee to exception chains explicitly. [Edge Case, Gap, Spec §FR-015, §FR-017]

## Dependencies & Assumptions

- [ ] CHK021 - Is the assumption "SLF4J `Logger.debug(...)` does not throw" stated explicitly? Constitution Principle I requires *every* call into SLF4J to be wrapped in try/catch — if the SLF4J no-throw assumption is load-bearing, naming it lets reviewers challenge it. [Assumption, Gap, Constitution Principle I bullet 1]

## Ambiguities & Conflicts

- [ ] CHK022 - Is the term "intercepted method" used consistently across the spec — does it mean *any* public method on an in-scope bean, or only methods whose package + class-name + repository filters all pass? The "rethrow unchanged" promise (FR-018) and "identical behavior" promise (FR-020) are scoped by this definition; an unclear scope is an unclear safety contract. [Ambiguity, Spec §FR-002, §FR-018, §FR-020]

## Notes

- Items reference spec sections by `[Spec §X]`, the constitution by principle, and contract files by file name + section. The minimum-80%-traceability target is met (every item carries at least one reference or marker).
- `[Gap]` markers (CHK003, CHK004, CHK012, CHK018, CHK019, CHK020, CHK021) call out requirements that are currently missing or implied-only — resolving each is a small spec edit, not a code task.
- This checklist is the safety lens only. Sensitive-data masking, performance, and public-API quality belong in `masking.md`, `performance.md`, and `api.md` if/when those are generated.
- Use `[x]` to check items off as each is resolved (either by an inline spec edit confirming the requirement is precise enough, or by a deliberate "accept the gap, document it" decision).
