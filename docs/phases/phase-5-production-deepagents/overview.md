# Phase 5: Production DeepAgents — Multi-Model & Observability

## Research Overview

**Research Question**: Can a production DeepAgents infrastructure agent run on a matrix of
**local and cloud** models — with systematic, automated evaluation and observability — and in
doing so, resolve the local-model failure documented in Phase 4 (ADR-0027)?

**Methodology**: Take the Phase 4 deepagents NetBox agent forward to packaged **DeepAgents
0.6.10**, replace the Anthropic-only path with a **dual Ollama / llama.cpp backend** (local
models *and* Ollama Cloud frontier models), and stand up a **LangSmith model-matrix evaluation
harness** that scores a fixed benchmark across many models automatically.

**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)

## Motivation — closing the loop on Phase 4

Phase 4 built the same NetBox agent twice (deepagents vs Claude Agent SDK) and concluded that
framework choice is context-dependent. A follow-on experiment (**ADR-0027**) then tried to add
local Ollama models to the **Claude SDK** build via a LiteLLM proxy — and failed:

- Qwen 2.5:14b executed MCP tools but did not synthesise the results into responses.
- Routing Anthropic through LiteLLM broke the SDK's native-protocol features (MCP, streaming,
  prompt caching, intelligent routing).
- The attempt was reverted after ~6 hours; the build stayed Anthropic-only.

Critically, ADR-0027's own *Future Considerations* pointed the way out:

> *"Alternative for Local Models: Consider a separate application for local model
> experimentation. Use the Deepagents framework for local model integration (more flexible).
> Don't compromise the production Claude SDK application for research use cases."*

Phase 5 is that separate application. It takes the **deepagents** side — not the SDK side — and
integrates local and cloud models through Ollama's **native** backend, with **no proxy layer**.
This sidesteps the exact incompatibility that sank the ADR-0027 attempt.

## Research Hypothesis

**Hypothesis**: The local-model failure in ADR-0027 was an artefact of the *proxy-vs-SDK*
architecture mismatch, **not** an inherent limitation of local/open models for tool-using
infrastructure agents. On a flexible framework (DeepAgents) with a native model backend, both
local and cloud open models should be usable — and a frontier cloud model should be competitive
with Claude on answer quality.

**Validation** (see [Multi-Model Evaluation](multi-model-evaluation.md)): partially confirmed
with an important nuance. Frontier *cloud* open models work excellently — `deepseek-v4-flash:cloud`
**matches Claude-class quality at ~36% lower latency than its own larger sibling**. But small
*local* models (Qwen 2.5:32b, DeepSeek-R1:14b) remain too weak for the hardest multi-step
queries — which **corroborates** ADR-0027's Qwen-14b finding while **refuting** its broader
"local models don't work" conclusion. The bottleneck is model scale/capability, not the
framework or the backend.

## Implementation Approach

### From vendored 0.0.5 to packaged 0.6.10

The Phase 4 deepagents repo vendored an early (0.0.5-era) fork of the framework. Phase 5 moves
to the **packaged DeepAgents 0.6.10** release, which required handling real framework-evolution
maintenance (documented in [Lessons Learned](lessons-learned.md)): a documentation bug worked
around in 0.5.x was fixed upstream (workaround removed), while 0.6's new default prompt/middleware
behaviour introduced a quality regression that needed a new `HarnessProfile`-based workaround.

### Dual backend + native model selection

```python
# One .env switch selects the backend; model identity flows through config/params.
LLM_BACKEND=ollama            # ollama | llamacpp
OLLAMA_MODEL=deepseek-v4-flash:cloud   # local OR :cloud frontier — no proxy
```

The Ollama daemon transparently proxies `:cloud`-suffixed names to Ollama Cloud, so the same
stack (`langchain-ollama` → DeepAgents → NetBox MCP) serves local 14B models and 1T-parameter
frontier MoEs without code changes. There is **no LiteLLM/OpenAI-compatibility shim** — the
incompatibility class that defeated ADR-0027 does not arise.

### Domain reliability carried forward

The NetBox-specific robustness from the ancestor build is preserved and hardened: the 4 generic
MCP tools, field filtering, the "no subagents" finding, and a custom **filter-recovery
middleware** that converts NetBox filter violations into structured, recoverable
`TOOL_VALIDATION_ERROR` / `TOOL_API_ERROR` messages rather than opaque HTTP 400s.

## What Phase 5 contributes

1. **A reproducible multi-model evaluation harness** (LangSmith) — the first systematic
   cross-model benchmarking in this research programme.
   See [Multi-Model Evaluation](multi-model-evaluation.md).
2. **Observability as a first-class practice** — automated scoring and trace-driven regression
   diagnosis, filling the programme's prior LangSmith/observability gap.
   See [Observability & Monitoring](observability-and-monitoring.md).
3. **A resolution of ADR-0027** — local+cloud models *do* work on DeepAgents via a native
   backend; the prior failure was architecture-specific. Recorded in ADR-0028.
4. **A framework-maintenance case study** — the DeepAgents 0.6 upgrade, its regressions, and a
   deferred-feature investigation (QuickJS/PTC). See [Lessons Learned](lessons-learned.md).

## Relationship to the other repositories

| Repo | Role |
|---|---|
| [deepagents](https://github.com/FinnMacCumail/deepagents) | Phase 4 ancestor (vendored 0.0.5, Anthropic-cloud-only). Now **superseded**; its design rationale was harvested into the successor's `docs/lineage/`. |
| [ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents) | **This phase.** Production DeepAgents 0.6.10 build with dual-backend + eval harness. |
| [claude-agentic-netbox](https://github.com/FinnMacCumail/claude-agentic-netbox) | Phase 4 Claude SDK build; subject of ADR-0027's local-model attempt. |

**See also**: [Multi-Model Evaluation](multi-model-evaluation.md) ·
[Observability & Monitoring](observability-and-monitoring.md) ·
[Lessons Learned](lessons-learned.md) · [ADR-0028](../../adr/0028-native-local-and-cloud-models-on-deepagents.md)
