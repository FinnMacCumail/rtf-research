# ADR-0033 — Reference-Grounded Correctness Evaluator

## Status

**Accepted** — Adopted (July 2026)
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents) (`tests/eval/`)

## Context

The Phase 5 model-matrix harness (ADR-0030) scored answers on three axes: `entity_coverage`
(deterministic keyword match), `completeness_judge` (LLM-as-judge), and `tool_call_efficiency`. None
of them was ever shown the correct answer. `entity_coverage` matches a fixed list of *positive*
keywords; `completeness_judge` was prompted with the question, the answer, and the
`expected_entities` list — **not** the dataset's `reference_answer` field, which existed but was
consumed by no evaluator.

The consequence surfaced on a cross-domain query asking for IP-allocation percentages across three
sites. The verified answer is **0%** (the demo instance's 180 IP addresses all live in an unrelated
`172.16.0.0/24` range; none are in the sites' `10.112.x` prefixes). Models repeatedly *found* the
180 IPs and misattributed them — producing a different fabricated figure per run (7.7%, 17.6%, 100%,
23.2%). Because the figure was specific and on-topic, `completeness_judge` scored one such answer
**0.9**; its recorded rationale credited the *presence* of a percentage, never its truth. The
harness was certifying hallucination as success.

## Decision

**Add a reference-grounded `correctness_judge` to the evaluator set, fed the dataset's
`reference_answer`, and re-score the existing answers with it.**

- The judge compares every factual claim (counts, percentages, IP/prefix allocations) against the
  reference and scores **contradiction**, not completeness. Extra correct detail does not lower the
  score; a specific value that disagrees with the reference scores low and is named in the rationale.
- Its rubric is primed on the observed failure mode ("a non-zero utilization % when the reference
  says 0% is a fabrication → score ≤ 0.3").
- It returns `None` (not `0.0`) when a reference is absent, so missing ground truth is
  distinguishable from a wrong answer.
- Re-scoring runs over the **already-stored** answers (judge calls only, no agent re-runs), so the
  correction is measured against the exact answers the old metric scored.

## Evidence

Re-scoring the 9-model June sweep surfaced **9 answers rated complete (≥0.7) that factually
contradict the reference**. Completeness and correctness diverge, and the ranking changes:

| Model | completeness (old) | correctness (new) |
|---|---|---|
| deepseek-v4-**pro** | 0.717 | **0.833** (→ #1) |
| deepseek-v4-**flash** | **0.800** (was #1) | 0.583 |
| minimax-m3 | 0.717 | 0.417 |

The June "flash ≥ pro" headline was a completeness artefact: pro was more conservative on the
negative findings, which completeness penalised and correctness rewards.

Two guardrails kept the result honest:

- **The judge is only as fair as its reference.** It initially penalised a defensible "4 devices per
  site" (3 tenant devices + 1 tenant-less patch panel) as contradicting a terse reference.
  Clarifying the reference wording lifted both models by removing *false* contradictions while
  genuine errors still scored low — producing the corrected `netbox-benchmark-v4` dataset.
- **Single-run variance is real.** Correctness on 6 questions × 1 run swings ±0.15–0.25; the
  per-question signal is robust, an aggregate ranking needs ≥3 runs. The harness records the caveat.

## Consequences

### Positive
- The harness now measures truth, not just fluency — directly serving the programme's
  anti-hallucination premise.
- A latent dataset field (`reference_answer`) became load-bearing with a one-evaluator change.
- Model selection is now grounded: the more trustworthy model is no longer ranked below the more
  fluent one.

### Negative / limitations
- Correctness depends on high-quality reference answers; an ambiguous reference produces false
  contradictions (mitigated by the reference-clarification pass).
- The default judge is a local model; contradiction-detection is good but coarse — a stronger judge
  would sharpen borderline scores.
- Aggregate rankings require replication (≥3 runs) to exceed single-run noise.

## Relationship to ADR-0030

Extends the model-matrix harness (ADR-0030): completeness measures *coverage*; correctness measures
*truth*. Both are retained — coverage still catches under-answering; correctness catches confident
fabrication that coverage rewards.

## References

- [Phase 5 → Evaluating for Correctness](../phases/phase-5-production-deepagents/evaluation-correctness.md)
- [Research Methods → Benchmarking](../methods/benchmarking.md)
- [ADR-0030 — Model-Matrix Evaluation Harness](0030-model-matrix-evaluation-harness.md)
- [ADR-0029 — LangSmith Observability Platform](0029-langsmith-observability-platform.md)
- Repository: `tests/eval/evaluators.py` (`correctness_judge`), `tests/eval/dataset_v4.py`,
  `tests/eval/rescore_v4.py`
