# ADR-0031 — DeepAgents 0.6 Upgrade and HarnessProfile Workaround

## Status

**Accepted** — Implemented (June 2026)
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)

## Context

Phase 5 moved from the Phase 4 vendored DeepAgents 0.0.5-era fork to the packaged **DeepAgents
0.6.10** release. A major version jump on the framework the agent is built on is a maintenance
event with real risk: APIs, default prompts, and default middleware can all change.

## Decision

Upgrade to DeepAgents 0.6.10, and manage the two behavioural changes the upgrade introduced:

1. **Remove an obsolete workaround.** On 0.5.x, a framework documentation bug (the `read_file` tool
   advertised the wrong argument name) silently broke skill loading; the project carried a
   `HarnessProfile.tool_description_overrides` workaround. 0.6 ships the upstream fix, so the
   workaround was deleted (~50 lines).

2. **Add a new workaround for a new regression.** 0.6 silently appends ~9.6K characters of default
   system-prompt content (`BASE_AGENT_PROMPT` + a sub-agent `task` prompt + `WRITE_TODOS_SYSTEM_PROMPT`)
   and enables a `write_todos` tool by default. On *negative-finding* queries this regressed quality:
   the "keep iterating until done" framing drove redundant search-hedging, and the
   "write the final answer after your last `write_todos` call" instruction caused the model to
   overwrite a correct answer with an "All done…" filler in a post-processing turn. The fix is a
   project `HarnessProfile` that suppresses the harmful defaults:

   ```python
   HarnessProfile(
       base_system_prompt="",                                  # drop BASE_AGENT_PROMPT
       excluded_middleware=frozenset({"TodoListMiddleware"}),  # remove write_todos + its prompt
   )
   ```

## How the regression was found and fixed

The model-matrix evaluation harness (ADR-0030) flagged the VLAN-100 query collapsing to 0.00
completeness after the upgrade. Trace sub-run inspection showed the *penultimate* LLM call already
held the correct answer, which a later turn overwrote — implicating a default middleware, not the
model. The fix restored the query to 1.00/1.00 and the aggregate to the pre-upgrade baseline,
**verified by the same harness that caught the regression**. Full case study:
[Phase 5 → Observability](../phases/phase-5-production-deepagents/observability-and-monitoring.md).

## Consequences

### Positive
- On the current packaged framework with its fixes and features (incl. the QuickJS interpreter
  investigated in ADR-0032).
- Net code reduction from removing the obsolete workaround.
- The upgrade was gated by an automated quality check rather than shipped blind.

### Negative / lessons
- **Major releases both remove and create the need for local adaptations** — workarounds must be
  re-tested on every upgrade.
- **Silent default-behaviour changes are a real risk** — ~9.6K chars of injected prompt and a new
  default tool changed agent behaviour without any API change. Without the eval harness the
  regression would have shipped.

## References

- [Phase 5 → Lessons Learned](../phases/phase-5-production-deepagents/lessons-learned.md)
- [Phase 5 → Observability & Monitoring](../phases/phase-5-production-deepagents/observability-and-monitoring.md)
- [ADR-0030 — Model-Matrix Evaluation Harness](0030-model-matrix-evaluation-harness.md)
- [ADR-0032 — QuickJS / PTC Deferral](0032-quickjs-ptc-deferral.md)
