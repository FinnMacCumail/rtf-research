# ADR-0035 — DeepAgents 0.6.10 → 0.7.5 Upgrade

## Status

**Accepted** — Upgraded, eval-gated (September 2026)
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
**Supersedes**: the framework state in [ADR-0031](0031-deepagents-0.6-upgrade-and-harnessprofile-workaround.md) (0.6 upgrade + HarnessProfile Workaround B)

## Context

The production NetBox agent ran on DeepAgents 0.6.10 (ADR-0031). Two motivations to move to 0.7.5:
the current **subagent** and **`RubricMiddleware`** APIs (needed for the planned model-handoff
routing) track the 0.7.x line, and a LangChain-ecosystem appraisal (ADR-0036) flagged the upgrade as
the actionable near-term step. Crossing the **0.7.0 major** carried real breaking changes, so the
upgrade was run behind the model-matrix eval gate — the same discipline as the 0.6 upgrade.

## Decision

**Upgrade to 0.7.5, reconcile Workaround B against 0.7's leaner defaults, and gate the change on the
`netbox-benchmark-v4` eval before committing.**

- **The one breaking change that hit us:** 0.7.0 makes planning **opt-in** — `TodoListMiddleware` is
  no longer bundled by default, and 0.7.x now **strictly raises `ValueError`** when an
  `excluded_middleware` entry matches no assembled middleware. Workaround B's
  `excluded_middleware={"TodoListMiddleware"}` was exactly that case.
- **Workaround B reconciliation:** the TodoList exclusion was **removed** (its suppression is now
  *inherent* — planning is opt-in), and `base_system_prompt=""` was **kept** as belt-and-suspenders
  (0.7's base prompt is empty by default anyway). Workaround B is now largely inherent to the
  framework rather than a workaround.
- Also bumped: langchain 1.3.9 → 1.3.14, langchain-core 1.4.7 → 1.5.3, langsmith 0.8.15 → 0.10.17.

## Evidence

- **API survival:** `HarnessProfile`, `register_harness_profile`, `create_deep_agent`,
  `FilesystemBackend` all still present; the agent builds and skills load with no loader warnings.
- **Regression-neutral (unit suite):** the failing tests fail *identically* on 0.6.10 (proven by
  reinstall-and-compare — they are stale hardcoded test expectations, not upgrade-caused).
- **Eval gate (`netbox-benchmark-v4`, production pair):** no correctness regression vs the committed
  0.6.10 baseline — combined mean flat within the documented single-run variance, and the
  Workaround-B-sensitive negative-finding queries stayed healthy (Jimbob VLAN 100 = 1.0/1.0 on both
  models). A first eval run was invalidated because NetBox was down — notably, the agent answered
  *"NetBox API unavailable"* rather than hallucinating, itself a good no-hallucination signal — and
  was re-run with NetBox live.

## Consequences

### Positive
- Unblocks the current subagent + `RubricMiddleware` APIs for the model-handoff routing work.
- 0.7's empty base prompt (~65% base-token cut) and opt-in planning **simplify** the codebase —
  Workaround B shrank to a single line.
- **Dissolved the GraphQL A/B's ~2× tool-call cost** (ADR-0034): the leaner prompts made the GraphQL
  path cost-neutral on 0.7.5.

### Negative / limitations
- A major-version bump with genuine breaks (the strict `excluded_middleware` error); mitigated by the
  eval gate and reinstall-and-compare.
- `langchain-quickjs 0.2.0` now pins `deepagents<0.7` → flagged incompatible, but **harmless**: it is
  the deferred QuickJS/PTC package (ADR-0032), used only by `tests/spike/`, never by the app.

## Relationship to ADR-0031

ADR-0031 documented the 0.5.6 → 0.6.10 upgrade and *introduced* Workaround B to suppress 0.6's
default-prompt/middleware regressions. This ADR carries that forward: 0.7's defaults make most of
Workaround B unnecessary, so it was reconciled down to `base_system_prompt=""`. The eval-gate
methodology is unchanged from ADR-0031.

## References

- [Phase 5 → DeepAgents 0.7.5 Upgrade](../phases/phase-5-production-deepagents/0-7-5-upgrade.md)
- [ADR-0031 — DeepAgents 0.6 Upgrade & HarnessProfile](0031-deepagents-0.6-upgrade-and-harnessprofile-workaround.md)
- [ADR-0034 — Read-Only GraphQL Complementary Read Path](0034-read-only-graphql-complementary-read-path.md) (the ~2× cost this upgrade dissolved)
- [ADR-0036 — LangChain Ecosystem Reproduces the Cloud Platform MCP](0036-langchain-ecosystem-reproduces-cloud-platform-mcp.md) (flagged this upgrade)
- Repository: `docs/development/2026-08-11_deepagents-0.7.5-upgrade.md`,
  `docs/traces/2026-08-11_netbox-benchmark-v4_d075-baseline.md`
