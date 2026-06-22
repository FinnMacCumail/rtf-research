# Model-Matrix Benchmarking

Methodology for systematically comparing many models on the same agent task. Introduced in
[Phase 5](../phases/phase-5-production-deepagents/multi-model-evaluation.md); this page is the
reusable design.

## Goal

Replace ad-hoc "run a query, eyeball the trace, write a report" comparison with a **reproducible,
automatically-scored matrix**: N models × a fixed dataset × a fixed set of evaluators, producing a
sortable leaderboard. The comparison surface becomes "capability and cost per query class," not
"which framework feels better."

## Components

### 1. A fixed dataset with ground truth

A small, curated set of representative queries (four NetBox query classes in Phase 5), each with a
machine-checkable reference. The reference used here is **`expected_entities`** — the concrete
facts a correct answer must mention — rather than a full reference answer, because:

- Full reference answers churn as the underlying data changes; entity lists are stable.
- Entity coverage is deterministic and cheap to compute.

**Ground-truth tightness matters.** An early dataset version was too permissive — a model could
score 1.0 by naming 3 of 14 required sites. Anchoring `expected_entities` to *full*, trace-derived
ground truth (`v2`) was what made the leaderboard discriminate between strong and weak models.

### 2. Evaluators (three complementary signals)

| Evaluator | Type | What it measures |
|---|---|---|
| `entity_coverage` | Code-based, deterministic | Fraction of `expected_entities` present in the answer |
| `completeness_judge` | LLM-as-judge | Whether the answer fully addresses the query (catches partial/wrong answers that happen to contain the entities) |
| `tool_call_efficiency` | Trajectory metric | Tool calls per query — surfaces retry loops and over-exploration |

Using all three is deliberate: a quality-only score misses efficiency pathologies (e.g. a model
that reaches the right answer via 18 tool calls because its malformed argument shape keeps getting
rejected), and a deterministic-only score misses semantic incompleteness.

### 3. A runner

One experiment per model. Practical concerns the runner handles:

- **Model-agnostic targets** — model identity flows through config, so local, cloud, and reference
  models all run through the same harness without code changes.
- **Skip-completed** — re-runs only score models without an existing complete experiment.
- **Fail-fast on quota** — cloud-inference session limits surface as errors, not bad scores; the
  runner aborts cleanly rather than polluting the leaderboard with zeros, and a force-rerun flag
  exists for deliberate regression checks against a baseline.

## Reading the leaderboard

Rank primarily on quality (`entity_coverage`, `completeness`), then use `tool_call_efficiency` and
latency to separate models of similar quality. Watch for:

- **Quality ties broken by efficiency** — two models at 1.00 completeness can differ 2–4× in tool
  calls and latency.
- **Concise-but-wrong** — a low entity/completeness with *normal* tool counts indicates the model
  is confidently wrong, not merely under-exploring.
- **Protocol/capability failures** — all-errored or no-tool-support outcomes are categorical, not
  quality, results (e.g. a reasoning-only model that can't tool-call, or a model whose tool-call
  serialization is incompatible).

## Reuse and extension

The same harness extends to: comparing a framework upgrade (run the *same* model before/after),
A/B-testing a prompt or middleware change, and adding new models as they appear. It is the gate
referenced throughout [Phase 5](../phases/phase-5-production-deepagents/overview.md) and the
detection half of the [observability loop](observability.md).

**See also**: [Evaluation](evaluation.md) · [Observability](observability.md) ·
[ADR-0030 — Model-Matrix Evaluation Harness](../adr/0030-model-matrix-evaluation-harness.md)
