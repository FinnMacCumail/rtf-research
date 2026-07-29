# Phase 5: Evaluating for Correctness — Catching the Hallucination the Metric Missed

## Research Overview

**Research Question**: Does an automated evaluation harness that scores *completeness* actually
measure whether an answer is *true*? And if not, what does a confidently-wrong answer do to a
leaderboard?

**Finding (short version)**: No — and it inflates the winners. The Phase 5 harness scored one
answer **0.9 on completeness** while it hallucinated the exact figure the question asked for
(reporting "~7.7% IP utilization / 180 IPs allocated" where the verified truth was **0%**). The
judge never saw the ground truth. Adding a **reference-grounded correctness evaluator** caught
**9 such hallucinations** across the model matrix that completeness had rated as complete — and
**reordered the leaderboard**.

This is the anti-hallucination thesis of the whole programme, observed *inside its own evaluation
tooling*: a metric that rewards fluent, specific, on-topic answers will reward confident fabrication
unless it is anchored to truth.

**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`tests/eval/`)

## The blind spot

The original Phase 5 harness (see [Multi-Model Evaluation](multi-model-evaluation.md)) scored each
answer on three axes: `entity_coverage` (deterministic substring match against `expected_entities`),
`completeness_judge` (an LLM-as-judge), and `tool_call_efficiency`. Two of these are quality
signals — and **neither was ever shown the correct answer**:

- `entity_coverage` checks that a fixed list of positive keywords (site names, tenant, "Active")
  appears. It has no entry for "the answer must say 0%", so it cannot see a wrong number.
- `completeness_judge` was prompted with the question, the answer, and the **`expected_entities`
  list** — *not* the dataset's `reference_answer`. Its rubric asked "does this address all parts and
  provide specific values?" A confident "~7.7% utilization" is specific and on-topic, so it scored
  high. The judge's own recorded rationale: *"covers tenant assignments, device counts, and IP
  utilization percentages"* — it credited the presence of a number, never its truth.

The dataset *had* the correct answer all along (`reference_answer`: "IP allocation percentage: 0% at
all three"). It was stored as documentation and **consumed by no evaluator**. The ground truth sat
one field away from the judge and was never passed to it.

## The trap that exposed it

One cross-domain query — *"Compare infrastructure utilization across DM-Nashua, DM-Akron, and
DM-Scranton … and IP allocation percentages"* — turned out to be a near-perfect hallucination trap.
The demo database holds 180 IP addresses, but **all of them live in an unrelated `172.16.0.0/24`
range**; none are in the three sites' `10.112.x` prefixes. The correct per-site utilization is
therefore **0%**.

Models consistently *found* the global 180 IPs and **misattributed** them to the sites. Across runs
this produced a different fabricated figure almost every time — **7.7%, 17.6%, 100%, 23.2%,
"<2%"** — each delivered confidently, each scored as complete. The fabrication was stable; only the
invented number varied.

## The fix: a reference-grounded correctness judge

The change was small and surgical — one evaluator, fed the field that already existed:

```python
# tests/eval/evaluators.py — the axis that was missing
def correctness_judge(inputs, outputs, reference_outputs):
    """Fact-check the answer against the dataset's reference_answer,
    flagging CONTRADICTIONS (counts, percentages, IP/prefix allocations)."""
    reference = reference_outputs.get("reference_answer")   # the field nothing read before
    ...
```

Design choices that matter:

- **Judge contradictions, not completeness.** Extra correct detail does not lower the score; a
  specific value that *disagrees* with the reference scores low and is named in the rationale.
- **Primed on the failure mode.** The rubric explicitly calls out "reports allocated IPs / a
  non-zero utilization % when the reference says 0% → this is a fabrication, score ≤ 0.3."
- **`None`, not `0.0`, when there is no reference** — a missing ground truth is distinguishable from
  a wrong answer.

The evaluator was then run over the **already-stored** answers from the 9-model June sweep — no
agent re-runs, only judge calls — so the correction was measured against the exact answers the old
metric had scored.

## Result: 9 hallucinations, and a reordered leaderboard

Re-scoring surfaced **9 answers rated complete (≥0.7) that factually contradict the verified
reference**. A sample:

| Model | Query | old completeness | new correctness | Contradiction |
|---|---|---|---|---|
| deepseek-v4-flash | site-comparison | 0.9 | **0.0** | "~7.7% IP utilization, 180 IPs" vs 0% |
| qwen3.5-397b | site-comparison | 1.0 | **0.0** | "17.58% IP utilization" vs 0% |
| glm-5 | site-comparison | 0.85 | **0.0** | "17.6% utilization" vs 0% |
| minimax-m3 | multi-site-vlan | 1.0 | **0.0** | "1,586 interfaces use VLAN 100" vs none |
| nemotron-3-ultra | rack-inventory | 1.0 | **0.5** | invented a switch→core-switch link |

Per-model, completeness and correctness diverge sharply — and the ranking is **not the same
metric twice**:

| Model | completeness (old) | correctness (new) |
|---|---|---|
| deepseek-v4-**pro** | 0.717 | **0.833** (rose to #1) |
| glm-5 | 0.725 | 0.667 |
| deepseek-v4-**flash** | **0.800** (was #1) | 0.583 |
| minimax-m3 | 0.717 | 0.417 |

The June headline "flash ≥ pro" was a *completeness* artefact: **pro was more conservative on the
negative findings, which completeness penalised and correctness rewards.** The more trustworthy
model was being ranked below the more fluent one.

## Hardening the reference, and a variance caveat

Two follow-through findings kept the result honest:

- **The judge was too strict where the reference was ambiguous.** It penalised "4 devices per site"
  as contradicting the reference's "3 tenant-owned devices" — but 4 (3 tenant devices + 1
  tenant-less patch panel) is defensible. Clarifying the reference wording ("3 tenant / 4 physical;
  5 prefixes = /22 + /28 + three /24s") lifted both models (flash 0.667 → 0.75, pro 0.5 → 0.65)
  *by removing false contradictions* — while the genuine errors still scored low. A correctness
  judge is only as fair as the reference it is anchored to. This produced the corrected
  **`netbox-benchmark-v4`** dataset.
- **Single runs are noisy.** Correctness on 6 questions × 1 run swung ±0.15–0.25, and the
  flash-vs-pro ranking flipped between runs. The **per-question signal** (the site-comparison trap
  scoring low on every run, every model) is robust; a headline *aggregate* ranking needs ≥3 runs.
  The harness records the caveat rather than over-claiming.

## Why this matters

The programme's premise is that a confident wrong answer is worse than no answer. This phase shows
that the *evaluation* of such a system is subject to the same failure: **a completeness metric will
certify hallucination as success.** Grounding the judge in the reference answer — a one-field change
that had been latent in the dataset — turned "the model sounds thorough" into "the model is right,"
and changed who wins. Observability caught the regression at the trace level (see
[Observability & Monitoring](observability-and-monitoring.md)); this closes the loop by catching it
at the *score* level.

**See also**: [Multi-Model Evaluation](multi-model-evaluation.md) ·
[GraphQL Read Path](graphql-read-path.md) ·
[Observability & Monitoring](observability-and-monitoring.md) ·
[Research Methods → Benchmarking](../../methods/benchmarking.md) ·
[ADR-0033](../../adr/0033-reference-grounded-correctness-evaluator.md)
