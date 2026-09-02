# Phase 5: The DeepAgents 0.7.5 Upgrade — a Framework Bump Behind the Eval Gate

## Research Overview

**Research Question**: When the agent framework ships a major version with breaking changes, how do
you adopt it *without* silently regressing a production agent — and can the same evaluation harness
that measures models also gate a framework upgrade?

**Finding (short version)**: Yes. Crossing the DeepAgents **0.7.0 major** (to 0.7.5) was run behind
the `netbox-benchmark-v4` eval gate exactly as the earlier 0.6 upgrade was — one breaking change hit,
was diagnosed and reconciled, and the upgrade shipped **regression-neutral**. A bonus: 0.7's leaner
defaults **dissolved the GraphQL A/B's earlier ~2× tool-call cost** ([GraphQL Read Path](graphql-read-path.md)).

**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)

## Why upgrade

Two drivers. First, the near-term roadmap — a **model-handoff routing** feature (a fast local model
fielding simple queries, escalating to a heavier model + a GraphQL subagent for multi-hop) — needs
the current **subagent** and **`RubricMiddleware`** APIs, which track the 0.7.x line. Second, the
[LangChain-ecosystem appraisal](langchain-ecosystem.md) flagged the upgrade as the concrete
next step. The framework had moved five releases and one major since the project's 0.6.10 baseline.

## The one breaking change — and the reconciliation it forced

0.7.0 makes **planning opt-in**: `TodoListMiddleware` is no longer bundled by default, *and* 0.7.x
now **strictly errors** when an `excluded_middleware` entry matches no assembled middleware. That
collided directly with **Workaround B** — the HarnessProfile introduced on the 0.6 upgrade
([ADR-0031](../../adr/0031-deepagents-0.6-upgrade-and-harnessprofile-workaround.md)) that suppressed
0.6's default-prompt/middleware regressions. Its `excluded_middleware={"TodoListMiddleware"}` now
matched nothing, and the agent failed to build.

The fix was a *reconciliation*, not a patch: 0.7's leaner defaults already do most of what
Workaround B did.

- **TodoList suppression is now inherent** — planning is opt-in, so the middleware simply isn't there
  to exclude. The exclusion was removed.
- **Empty base prompt is now the default** — so `base_system_prompt=""` was *kept* only as
  belt-and-suspenders. Workaround B shrank to a single line.

A neat illustration of a healthy framework upgrade: the workaround a prior version needed became
*inherent* in the next, and adopting it made the codebase simpler, not more complex.

## Gating the upgrade on the eval harness

The same discipline as the 0.6 upgrade — a framework change goes through the model-matrix eval before
it is committed:

- **API survival check** — the Workaround-B symbols (`HarnessProfile`, `register_harness_profile`,
  `create_deep_agent`, `FilesystemBackend`) all still import; the agent builds; skills load clean.
- **Regression-neutral (unit suite)** — the failing tests fail *identically* on 0.6.10 (verified by
  reinstalling the old version and comparing). They are stale test expectations, not upgrade damage.
- **Eval gate (`netbox-benchmark-v4`)** — no correctness regression vs the committed 0.6.10 baseline;
  the negative-finding queries Workaround B protected stayed healthy (Jimbob VLAN 100 = 1.0/1.0 on
  both models). One eval run was thrown out because NetBox happened to be down — and notably the
  agent answered *"NetBox API unavailable"* instead of hallucinating, a small no-hallucination win in
  its own right — then re-run with the instance live.

## The bonus finding

Re-measuring the GraphQL A/B under 0.7.5 showed the **~2× tool-call cost from 0.6.10 had vanished** —
the leaner default prompts made the agent cost-neutral (sometimes cheaper) with the GraphQL path.
This turned the GraphQL verdict from "correctness win *at a cost*" into "correctness win, cost-neutral"
— a reminder that a measured downside can be an artifact of the *layer around* the model, not the
feature itself (a recurring Phase 5 theme, see [Observability & Monitoring](observability-and-monitoring.md)).

## Takeaway

A major framework bump is not a leap of faith when a fixed benchmark stands behind it: the eval gate
turns "did this break anything?" into a measured pass/fail, and the reinstall-and-compare technique
cleanly separates pre-existing test debt from real regressions. And the healthiest sign of a maturing
framework is when last version's workaround becomes this version's default.

**See also**: [GraphQL Read Path](graphql-read-path.md) ·
[LangChain Ecosystem vs Cloud Platform MCP](langchain-ecosystem.md) ·
[Observability & Monitoring](observability-and-monitoring.md) ·
[Lessons Learned](lessons-learned.md) ·
[ADR-0035](../../adr/0035-deepagents-0.7.5-upgrade.md) ·
[ADR-0031](../../adr/0031-deepagents-0.6-upgrade-and-harnessprofile-workaround.md)
