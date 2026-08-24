# Validate Region Epoch Before Reusing a Cached Physical Plan

This ExecPlan is a living document. Keep `Progress`, `Surprises & Discoveries`, `Decision Log`, and `Outcomes & Retrospective` up to date as work proceeds.

Reference: `PLANS.md` at the TiDB repository root; this plan must be maintained according to it.

## Purpose / Big Picture

TiDB must not execute a physical plan whose region-routing assumptions became stale after a Region split, merge, or equivalent epoch-changing topology update. A prepared-plan cache hit must therefore be validated before the cached physical plan is handed back for execution. If the referenced region epoch is no longer current, TiDB must refresh the route, discard the stale reusable plan state, and build a valid plan instead.

After this work, a statement that would otherwise hit the physical plan cache immediately after a topology change either executes against current regions or takes the existing retry/replan path. It must never send a stale region read because the cache hit bypassed the epoch fence. Normal cache hits with unchanged routing must remain cache hits without unnecessary invalidation.

This plan deliberately does not fold in the separate store-side fixes for stale region-cache entries during lock resolution or pessimistic locking. Those paths share the same underlying topology condition, but they have independent correctness contracts and regression coverage.

## Progress

- [x] (2026-08-24) Established the scope boundary: planner physical-plan-cache reuse versus store routing during read, resolve-lock, and pessimistic-lock operations.
- [x] (2026-08-24) Identified repository policy: bug fixes require a regression test that fails before the fix and passes after it; PR metadata must retain the repository template structure.
- [ ] Reconcile Rafael's referenced review request with the actual source branch and commit. The public `pingcap/tidb#56601` currently resolves to an unrelated historical planner change, so the implementation must use the intended branch/PR SHA rather than assume that public PR number identifies the patch.
- [ ] Map the current physical-plan-cache hit path and the store region-epoch validation/refresh API, including the exact point at which a cached plan becomes executable work.
- [ ] Write focused red tests covering a cached-plan hit followed by an epoch-changing topology event.
- [ ] Implement the smallest shared validation/refresh handoff needed by planner and store; do not duplicate epoch comparison logic in the planner.
- [ ] Run required targeted checks, document exact commands and outcomes, and update the PR metadata according to the template.

## Surprises & Discoveries

- Observation: Rafael's notification described a planner/store change, but `pingcap/tidb#56601` on the public repository is a merged, unrelated planner-binder PR. The notification's PR number or repository target cannot be used as implementation evidence.
  Evidence: GitHub metadata for `pingcap/tidb#56601` identifies the title as `planner: add stack based pattern binder logic.`

- Observation: the existing store work is broader in operational surface but narrower in planner scope. It addresses stale region cache after a split and follow-up correctness paths for resolve-lock and pessimistic-lock routing; it does not make physical-plan-cache reuse safe.
  Evidence: review discussion describes stale post-split cache entries, resolve-lock epoch revalidation, and lock-manager/region-cache coherence, with no executor or plan-cache changes.

## Decision Log

- Decision: Treat planner cache-hit validation and the open store-side work as complementary, not duplicate implementations.
  Rationale: a store cache may refresh correctly while an already-built physical plan still carries obsolete region assumptions; conversely a planner guard cannot make lock recovery or pessimistic-lock routing coherent.
  Date/Author: 2026-08-24 / Nadia

- Decision: Validate the cache entry before it is returned for execution; do not rely on cache-key normalization or add the region epoch to the plan-cache key.
  Rationale: cache keys establish SQL/plan equivalence, while an epoch is a volatile routing validity condition. Adding it to the key creates topology-driven cache churn; validating after reuse leaves the unsafe window open.
  Date/Author: 2026-08-24 / Nadia

- Decision: Prefer a store-owned validation/refresh primitive consumed by planner over a second planner-local epoch check.
  Rationale: region epoch semantics and cache invalidation belong to the store layer. One primitive prevents divergent retry, invalidation, and backoff behavior across paths.
  Date/Author: 2026-08-24 / Nadia

- Decision: Do not merge, rebase, or modify either existing branch until the true review branch and overlap analysis are recorded in this plan.
  Rationale: the public PR-number mismatch makes it unsafe to infer code ownership or duplicate work from notification metadata alone.
  Date/Author: 2026-08-24 / Nadia

## Outcomes & Retrospective

Not started. At completion, record the final planner/store contract, the triggering test topology, validation results, any measured cache-hit overhead, and remaining paths that intentionally stay outside this change.

## Context and Orientation

Region routing is cached client-side. Each region has an epoch that changes when its topology changes, such as a split. A physical plan can contain work whose routing was resolved while that epoch was current. The physical plan cache avoids rebuilding such a plan on repeated execution. The defect under investigation is the composition of those two otherwise-valid optimizations: a cache hit may reuse a plan after its routing assumptions are obsolete.

The relevant repository areas are `/pkg/planner/` for physical-plan construction and plan-cache reuse, and `/pkg/store/` plus `/pkg/kv/` for region routing, epoch validation, invalidation, and request retry. Before editing a package, read its nearest `doc.go` if one exists, then use `docs/agents/architecture-index.md` to identify the intended test surface.

The desired contract is:

1. A candidate physical-plan-cache hit exposes the region references it depends on, or an equivalent validity token sufficient for the store layer to validate those references.
2. Planner asks the store-owned primitive to validate that dependency before returning the cached plan for execution.
3. If validation succeeds, execution follows the existing cache-hit behavior.
4. If validation detects a stale epoch, it refreshes/invalidates the relevant route by the existing store mechanism and forces the current statement onto the existing safe replan or retry path. It must not execute the cached plan first.
5. A validation failure must retain existing error and retry semantics; it must not turn a transient topology update into a silently swallowed error or a permanently poisoned cache entry.

## Plan of Work

### Milestone 1: Establish the implementation baseline

Resolve the intended private branch, commit SHA, or current draft patch from Rafael before reviewing or editing. Record the exact base and head SHAs in this plan. Fetch the current open store-side PR and record its touched files, behavior, and tests. Compare the two file lists and call sites; this is the final duplicate-work check.

From the intended base, locate the cache-hit branch and trace it until the cached physical plan is returned to the caller. Locate the region-routing calls made by the plan or its executors. Separately locate the canonical store path that receives epoch-not-match, invalidates the stale region cache entry, refreshes the route, and decides whether to retry. Read package documentation before source files and note all concrete function/type names here.

The milestone is complete only when the plan names: (a) cache-hit function, (b) last safe point before cached-plan reuse, (c) store validation/refresh entry point, (d) existing retry/replan owner, and (e) test packages for the two layers. Do not infer these from issue wording.

Run from repository root:

    rg -n --glob 'doc.go' 'package (planner|store|kv)' pkg/planner pkg/store pkg/kv
    rg -n -i 'plan.?cache|physical.*cache|cache.*physical' pkg/planner
    rg -n -i 'epoch.?not.?match|region.*epoch|invalidate.*region|region.*invalidate' pkg/store pkg/kv
    rg -n -i 'resolve.?lock|pessimistic.?lock' pkg/store pkg/kv
    git status --short
    git diff --check

Expected result: a short call-path note in this document and no unrelated worktree changes attributed to this task.

### Milestone 2: Prove the planner cache-hit failure before changing production code

Extend the nearest existing planner plan-cache test rather than introduce a new test harness. Build a deterministic scenario that first creates a cacheable physical plan, then causes the relevant region route to become stale, then executes the same statement as a cache hit. The unmodified code must prove the defect through an observable stale-routing outcome: an epoch-not-match request escaping to the wrong layer, incorrect cache reuse, or a test hook/controlled route assertion that records reuse of the obsolete mapping.

Avoid a test that merely checks the store cache in isolation. The assertion must prove the planner took the physical-plan-cache hit path and that no stale route was dispatched. Keep the topology event deterministic using the existing test utilities or failpoints only when the target package already uses them. If an in-process unit test cannot produce an authentic split/epoch transition, retain the unit test for cache-hit control flow and add a scoped RealTiKV regression test to prove the integration behavior.

Record the exact test name, package, failing command, and red output in `Surprises & Discoveries`.

### Milestone 3: Implement the narrow handoff

Add the smallest explicit interface or helper required for the planner cache-hit code to ask whether the cached plan's region dependencies remain valid. The store implementation must own comparison against current routing state and the invalidation/refresh response. The planner must not copy region-cache mutation logic, maintain a second epoch cache, or include the epoch in its SQL plan-cache key.

Place the validation before the cache hit is returned or made executable. On a stale result, discard or invalidate only the affected reusable plan state and enter the existing safe rebuild/retry flow. Preserve the normal cache-hit fast path and make any extra lookup bounded to the regions actually referenced by the cached plan. Add comments only for the cross-layer invariant and the reason the check precedes reuse.

If the current plan representation cannot expose a valid dependency set without a broad refactor, stop and record the limitation. Choose a narrow validity token or a store-supplied plan-level validation hook only after measuring the changed API surface and updating the Decision Log.

### Milestone 4: Add regression and negative coverage

Make the red scenario pass. Add the smallest complementary cases necessary to establish the contract:

- unchanged region epoch: the same statement retains a normal physical-plan-cache hit and does not take the refresh/replan path;
- topology change before the second execution: cached physical-plan reuse is rejected before dispatch, route refresh occurs through the store-owned mechanism, and the statement succeeds via the safe path;
- topology change after the initial cache lookup but before dispatch, if the chosen implementation exposes this window: the plan must still not dispatch stale routing;
- unrelated store paths: existing resolve-lock and pessimistic-lock regression tests remain independent and pass unchanged.

For every new test, establish the expected-red proof against the pre-fix baseline before declaring the fix complete. If a new top-level Go test function, an import change, a Go-file move/addition, or a Bazel target change triggers the repository gate, run `make bazel_prepare` and include generated Bazel metadata changes.

### Milestone 5: Validate, review, and prepare the write-up

Use the narrowest test set that proves each touched package, then a RealTiKV test if required by the actual topology behavior. Before final PR-readiness claims, use the repository's `Ready` verification profile and `make lint`; do not claim that all tests pass unless those commands have actually completed.

Prepare the PR using the repository template and preserve headings and hidden HTML comments. Use an English, module-oriented title such as `planner, store: validate region epoch before reusing cached physical plans` only if the final touched modules support that scope. Link the existing issue with correct `close` or `ref` syntax. State the behavioral regression, the pre-reuse fence, the store-owned invalidation handoff, test commands, and any benchmark result. Keep the release-note block present; use `None` only if the final repository guidance and user-facing impact justify it.

## Validation and Acceptance

Acceptance is behavioral, not merely compilation:

1. Before the production change, the focused regression fails while proving that a physical-plan-cache hit can retain stale routing after an epoch-changing event.
2. After the production change, the same test passes: the cache hit is rejected or rebuilt before any stale region request is dispatched.
3. An unchanged-epoch control remains a cache hit and does not regress into unconditional replanning.
4. Existing store-side resolve-lock and pessimistic-lock tests pass without their behavior being folded into the planner change.
5. Targeted planner and store tests pass. A scoped RealTiKV test passes when authentic split/epoch behavior is required.
6. `git diff --check` passes; generated Bazel metadata is current whenever the repository gate requires `make bazel_prepare`.
7. The final PR body preserves the official template and lists exact test commands that were actually run.

Initial command forms, to be replaced in this section with concrete resolved packages and test names during Milestone 1:

    pushd pkg/<resolved-planner-package>
    go test -run <resolved-cache-hit-regression-test> -tags=intest,deadlock
    popd

    pushd pkg/<resolved-store-package>
    go test -run <resolved-region-validation-test> -tags=intest,deadlock
    popd

If the package contains failpoints, first use the repository-prescribed decision check and then run the failpoint wrapper rather than plain `go test`:

    rg -n --fixed-strings -- 'failpoint.' pkg/<resolved-package>
    rg -n --fixed-strings -- 'testfailpoint.' pkg/<resolved-package>
    ./tools/check/failpoint-go-test.sh pkg/<resolved-package> -run <resolved-test-name>

When the test requires a real topology transition, follow `docs/agents/testing-flow.md` exactly: start TiUP Playground in the background, run only the resolved RealTiKV test, shut it down, remove its tagged data directory, and confirm PD is unreachable after cleanup.

## Idempotence and Recovery

Source discovery, targeted test runs, and `git diff --check` are safe to rerun. Keep each production change in a small commit or clearly separable diff so the planner/store handoff can be reviewed independently from tests.

If a topology test is flaky, do not add sleeps or broaden retries until the race is understood. First replace timing with an existing deterministic hook, barrier, or test utility; record the prior nondeterminism and the new synchronization in `Surprises & Discoveries`.

If validation exposes that the planner cannot obtain a sound region-dependency set, stop before adding a best-effort check. Record the missing contract, preserve the failing regression, and prototype the smallest explicit validity token. Do not ship a guard that can return success for a plan with unknown region dependencies.

If a RealTiKV run fails or is interrupted, always run the documented cleanup and confirm the PD endpoint is unreachable before retrying. Do not delete unrelated local data or modify a shared branch to recover a test environment.

## Artifacts and Notes

The final PR description should contain a concise evidence record:

- the exact scenario: cache a physical plan, advance the region epoch, execute the same statement;
- expected prior failure and fixed behavior;
- the selected planner/store handoff and why it occurs before reuse;
- exact targeted test and, if applicable, RealTiKV commands with outcomes;
- cache-hit control and any measured overhead;
- confirmation that the store-side resolve-lock and pessimistic-lock fixes remain separately owned and tested.

Do not include speculative implementation names, assumed protocol fields, or unverified claims about TiKV responses. Replace all `<resolved-...>` placeholders only after Milestone 1 identifies them from the intended source branch.

## Interfaces and Dependencies

The final interface should be expressed in repository-native terms after discovery. Its minimum semantics are:

    ValidateCachedPlanRegions(ctx, cachedPlanDependencies) -> valid | stale-and-refreshed | error

This is behavioral pseudocode, not a required Go signature. `cachedPlanDependencies` must be sufficient to identify every region route that makes the cached plan unsafe when stale. `valid` permits normal reuse. `stale-and-refreshed` causes the caller to invalidate/rebuild without dispatching the old plan. `error` follows established error/retry semantics and must not be converted to a successful cache hit.

The planner depends on this contract but does not own region-cache mutation. The store layer depends on its existing PD/region-cache and retry machinery. The implementation must remain compatible with current physical-plan-cache and executor ownership boundaries discovered in Milestone 1.

---

Plan created 2026-08-24 to resolve the planner physical-plan-cache epoch-staleness defect without duplicating the open store-side split-routing work.
