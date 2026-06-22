# Lessons Learned

Phase 5 was as much about **maintaining** a production agent across a framework version jump as
about building it. The lessons below are the durable takeaways.

## 1. Framework upgrades are research events, not chores

Moving from the vendored DeepAgents 0.0.5-era fork to packaged **0.6.10** produced two opposite
surprises, both worth recording:

- **A workaround became obsolete.** On 0.5.x, a documentation bug in the framework's `read_file`
  tool (it advertised the wrong argument name) silently broke skill loading; the project carried a
  `HarnessProfile`-based workaround for it. On 0.6 the upstream fix had shipped — so the workaround
  was *removed* (~50 lines deleted). Lesson: re-test your workarounds on every upgrade; some are now
  dead weight.
- **A new workaround became necessary.** 0.6 silently appends ~9.6K characters of default
  system-prompt content and enables a `write_todos` tool by default. On *negative-finding* queries
  this actively regressed quality (see the case study in
  [Observability & Monitoring](observability-and-monitoring.md)). The fix was a project
  `HarnessProfile` suppressing the harmful defaults (`base_system_prompt=""`,
  `excluded_middleware={"TodoListMiddleware"}`).

**Takeaway**: a major framework release can both *remove* and *create* the need for local
adaptations. Without an evaluation harness gating the upgrade, the regression would have shipped
silently. The upgrade is documented in ADR-0031.

## 2. The negative result, done right: QuickJS / Programmatic Tool Calling

DeepAgents 0.6 introduced a QuickJS-based **Code Interpreter / Programmatic Tool Calling (PTC)**
middleware — the agent can write JavaScript that composes tool calls inside a sandbox, instead of
round-tripping every intermediate result through the model. It was an attractive lever for the
slow multi-call queries, so it was investigated properly with three verification spikes:

| Spike | Question | Result |
|---|---|---|
| 1 | Do the NetBox MCP tools bridge into the JS sandbox? | ✅ Yes — auto-exposed as `tools.netboxGetObjects(...)`, real round-trip. |
| 2 | Does the existing filter-recovery middleware still catch errors made *inside* the sandbox? | ✅ Yes (an unexpected positive — it wraps the tool call itself). |
| 3 | Does it actually reduce wall time on the heavy VLAN query? | ❌ **No.** The model *never chose* to use the `eval` tool on this workload, and merely *having* it available cost **+13.7%** latency from prompt bloat. |

**Decision: deferred, not adopted.** The mechanism works, but PTC's value comes from
fan-out/parallelism across many tools or large-result filtering — none of which a single-source,
sequential, 4-tool NetBox agent has. This independently reproduces Anthropic's own finding that
*"sequential single-call workflows do not benefit"* from PTC. The re-trigger conditions (≥10 tools,
≥2 parallel data sources, cross-source joins) are recorded for a future multi-source phase.

**Takeaway**: spiking a feature to a clear "not now, and here's exactly when to revisit" is a
first-class research output. It also nicely corroborates the **no-subagents** finding inherited
from the ancestor build — the same workload that doesn't benefit from subagents doesn't benefit
from code-orchestrated tool calls, for the same reason (it's sequential and dependency-chained).
Full detail in ADR-0032.

## 3. Native backends beat proxy layers (the ADR-0027 lesson, confirmed from the other side)

ADR-0027 failed to add local models to the Claude SDK build via a LiteLLM proxy — the proxy broke
the SDK's native-protocol features. Phase 5 succeeded by using Ollama's **native** backend on a
flexible framework, with no proxy. The same lesson from two angles: **adapter/proxy layers that
sit between an agent framework and its model provider tend to break the framework's
provider-native features.** When you need multi-model flexibility, prefer a framework whose
backend abstraction is native to each provider over a universal proxy. (Recorded in ADR-0028.)

## 4. Frontier-cloud closes the quality gap; small-local does not (yet)

The [model matrix](multi-model-evaluation.md) is unambiguous: a frontier open/cloud model
(`deepseek-v4-flash:cloud`) matches Claude-class quality, while 14–32B local models remain too
weak for the hardest multi-step queries. For a privacy-or-cost-driven local deployment, the honest
status is "wait for the open-weights frontier to be runnable on affordable hardware" rather than
"local models work today." This tempers the original local-first privacy thesis with measured
evidence.

## 5. Operational hygiene is part of the work

Two non-glamorous but real lessons from taking the build to a public repository:

- **Secrets leak through git history, not just files.** A LangSmith API key had been committed to
  documentation in earlier history. Scrubbing the working tree is not enough — the key had to be
  removed from *all* commits (via `git filter-repo`) before the repo could be published, and the
  only fully-safe remedy for an exposed key is rotation. Treat a committed secret as compromised.
- **Documentation drift is a maintenance debt that compounds.** Bringing the project's own docs
  back in line with reality (framework version, default model, removed middleware) was a
  meaningful task in itself — and the cue to formalise a companion engineering doc (`AGENTS.md`)
  alongside the high-level `CLAUDE.md`.

## Summary

| Lesson | One-line takeaway |
|---|---|
| Framework upgrades | Re-test workarounds; a major release both removes and creates them. Gate upgrades with an eval harness. |
| QuickJS / PTC | Deferred with explicit re-trigger conditions — a clean negative result. |
| Native vs proxy | Native model backends preserve framework features; proxies break them (ADR-0027 confirmed from both sides). |
| Local vs frontier | Frontier cloud matches Claude; small local models don't (yet). Bottleneck is model scale. |
| Ops hygiene | Secrets live in history; rotate exposed keys; keep docs honest. |

**See also**: [Overview](overview.md) · [Multi-Model Evaluation](multi-model-evaluation.md) ·
[Observability & Monitoring](observability-and-monitoring.md) ·
ADR-[0028](../../adr/0028-native-local-and-cloud-models-on-deepagents.md)–[0032](../../adr/0032-quickjs-ptc-deferral.md)
