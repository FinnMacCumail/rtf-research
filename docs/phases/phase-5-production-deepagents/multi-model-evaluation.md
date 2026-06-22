# Multi-Model Evaluation

## Research Question

Across a matrix of local and cloud models on the same DeepAgents NetBox agent, **which models
are production-viable**, and **does a frontier open/cloud model match Claude-class quality** —
the question ADR-0027 left open after its proxy-based local-model attempt failed?

## Method — a fixed benchmark, scored automatically

The evaluation replaces the prior "run a query, eyeball the trace, write a markdown report" loop
with a reproducible harness (`tests/eval/` in the repository, built on LangSmith). See
[Research Methods → Benchmarking](../../methods/benchmarking.md) for the methodology in full.

- **Dataset** (`netbox-benchmark-v2`): four NetBox query classes with ground-truth
  `expected_entities` drawn from real traces — single-object lookup, device-detail, multi-aspect
  tenant enumeration (14 sites), and a cross-relationship VLAN-deployment query (the hardest).
- **Evaluators**: `entity_coverage` (deterministic, code-based), `completeness_judge`
  (LLM-as-judge), `tool_call_efficiency` (trajectory cost).
- **Runner**: one experiment per model, with skip-completed and fail-fast-on-quota logic;
  models selectable via `EVAL_MODELS=...`.

Model identity flows through config, so the same harness scores local Ollama models, Ollama
Cloud frontier models, and (as an occasional reference) Claude — without code changes.

## Results — 10-model cloud sweep (`netbox-benchmark-v2`)

Scores are averaged across the four benchmark queries. `entity` and `completeness` are 0–1
(higher is better); `tools` is mean tool-calls per query; `lat` is mean wall-clock seconds.

| Rank | Model | entity | completeness | tools | lat (s) |
|---|---|---|---|---|---|
| 1 | `deepseek-v4-flash:cloud` | **0.95** | **1.00** | 7.2 | **34.6** |
| 2 | `deepseek-v4-pro:cloud` | 0.95 | 1.00 | 7.8 | 54.5 |
| 3 | `minimax-m3:cloud` | 0.94 | 1.00 | 18.8 | 102.0 |
| 4 | `nemotron-3-ultra:cloud` | 0.91 | 0.82 | 8.2 | 197.1 |
| 5 | `glm-5:cloud` | 0.90 | 1.00 | **4.5** | **25.9** |
| 6 | `qwen3.5:397b-cloud` | 0.90 | 0.75 | 17.8 | 55.8 |
| 7 | `nemotron-3-super:cloud` | 0.87 | 0.75 | 10.0 | 347.5 |
| 8 | `kimi-k2.6:cloud` | 0.82 | 0.80 | 7.5 | 34.2 |
| 9 | `gpt-oss:120b-cloud` | 0.52 | 0.50 | 6.5 | 44.1 |
| 10 | `gemini-3-flash-preview:cloud` | — | — | — | (4/4 errored) |

### Headline findings

- **A frontier open/cloud model matches Claude-class quality.** `deepseek-v4-flash:cloud` scores
  0.95 / 1.00 — and does so at **~36% lower latency than its own larger sibling** `deepseek-v4-pro`
  (34.6s vs 54.5s) with fewer tool calls. The smaller model is the right production default; the
  1.6T-parameter `pro` is overkill for read-only infrastructure queries.
- **Efficiency king:** `glm-5:cloud` is fastest (25.9s) with the fewest tool calls (4.5) at near-top
  quality — notable, as GLM-5 is LangChain's own DeepAgents reference model.
- **A surprise underperformer:** `gpt-oss:120b-cloud` scored 0.52 / 0.50 — concise but wrong;
  below every other working model.
- **A protocol break:** `gemini-3-flash-preview:cloud` failed all four queries with a
  `thought_signature` tool-call protocol error — a serialization incompatibility, not a capability
  gap.

### Local models — corroborating *and* refuting ADR-0027

Local models tested separately on the same harness:

| Model | Outcome |
|---|---|
| `qwen2.5:32b-instruct-q4_K_M` (local) | entity ~0.20 — produced malformed tool arguments and hallucinated answers; **too weak for the multi-step queries**. |
| `deepseek-r1:14b` (local) | **No tool-calling capability** — the Ollama API rejected every call (`does not support tools`, 400). A pure reasoning model; excluded. |

This is the nuanced result that closes the ADR-0027 loop:

- **Corroborates** ADR-0027: a small local model (Qwen-class 14–32B) genuinely struggles to
  reliably synthesise tool results into answers on this workload.
- **Refutes** ADR-0027's broader conclusion: that was framed as "local models don't work / aren't
  worth it." Phase 5 shows the *frontier* tier works excellently via a **native** backend with no
  proxy — and that the prior failure was substantially an artefact of the LiteLLM-proxy-vs-SDK
  architecture mismatch, compounded by using a small model. **The bottleneck is model scale, not
  the framework or local-vs-cloud per se.**

## Methodological value beyond the scores

Two side-findings emerged from running the matrix as a controlled experiment rather than ad-hoc:

1. **Tightened ground truth changes rankings.** An early dataset version (`v1`) used a permissive
   `expected_entities` list; a model could score 1.0 by naming 3 of 14 sites. Anchoring the
   benchmark to full trace-derived ground truth (`v2`) made the leaderboard discriminate properly
   — e.g. `minimax-m3` dropped from a flattering score once partial answers were penalised.
2. **Tool-call efficiency is a real signal.** `minimax-m3` reached top quality but at 18.8 tool
   calls/query (vs deepseek-flash's 7.2) — it emits a malformed `{"item": [...]}` argument shape
   that the validator bounces, forcing retry loops. Invisible to a quality-only score; obvious in
   the trajectory metric.

## Operational conclusion

| Decision | Outcome |
|---|---|
| Production default | `deepseek-v4-flash:cloud` — Claude-class quality, lowest latency in its family |
| Strong secondary | `glm-5:cloud` — fastest, tightest tool use |
| Local viability | Frontier-only for the hardest queries; 14–32B local models remain insufficient |
| ADR-0027 status | Superseded by **ADR-0028** — local+cloud works on DeepAgents via a native backend |

**See also**: [Observability & Monitoring](observability-and-monitoring.md) ·
[Research Methods → Benchmarking](../../methods/benchmarking.md) ·
[ADR-0028](../../adr/0028-native-local-and-cloud-models-on-deepagents.md) ·
[ADR-0030](../../adr/0030-model-matrix-evaluation-harness.md)
