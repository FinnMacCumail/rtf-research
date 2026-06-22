# ADR-0028 — Native Local and Cloud Models on DeepAgents

## Status

**Accepted** — Production implementation complete (June 2026)
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
**Supersedes**: [ADR-0027 — Intelligent Routing and Model Selection](0027-intelligent-routing-and-model-selection.md) *(on the question of local/multi-provider model support)*

## Context

ADR-0027 (Claude Agent SDK build) attempted to add local Ollama models via a LiteLLM proxy and
reverted after the proxy broke the SDK's native-protocol features (MCP integration, streaming,
prompt caching, intelligent routing) and a small local model (Qwen 2.5:14b) failed to synthesise
tool results into answers. Its conclusion was "Anthropic-only," but its *Future Considerations*
recommended: **"Use the Deepagents framework for local model integration (more flexible)"** in a
**separate application**.

Phase 5 (`ollamaDeepAgents`) is that application. The question this ADR settles: can local *and*
cloud models be used reliably on a DeepAgents NetBox agent, without the proxy-induced failures of
ADR-0027?

## Decision

Adopt a **dual native backend** with no proxy layer:

- `LLM_BACKEND=ollama` — local Ollama models **and** Ollama Cloud frontier models (the daemon
  transparently proxies `:cloud`-suffixed names to ollama.com; the application sees one backend).
- `LLM_BACKEND=llamacpp` — an OpenAI-compatible llama.cpp server for the fully-local path.

Model identity flows through config/params (`OLLAMA_MODEL`, or `--model`), so the same stack
(`langchain-ollama` → DeepAgents 0.6.10 → NetBox MCP) serves a 14B local model and a 1T-parameter
cloud MoE without code changes. There is **no LiteLLM / OpenAI-compatibility shim between the
framework and the provider** — the incompatibility class that defeated ADR-0027 does not arise,
because Ollama is a native LangChain backend.

## Why this works where ADR-0027 didn't

| ADR-0027 (Claude SDK + LiteLLM proxy) | ADR-0028 (DeepAgents + native Ollama) |
|---|---|
| Proxy sits between SDK and Anthropic; breaks MCP/streaming/caching | No proxy; Ollama is a first-class LangChain backend |
| SDK assumes Anthropic-native protocol | DeepAgents is provider-agnostic by design |
| Tool results not displayed (Qwen 2.5:14b) | Tool results integrated correctly by frontier models |
| Reverted after ~6 hours | Production default |

## Evidence

From the [model-matrix evaluation](../phases/phase-5-production-deepagents/multi-model-evaluation.md)
on `netbox-benchmark-v2`:

- **Frontier cloud matches Claude-class quality.** `deepseek-v4-flash:cloud` scores 0.95 entity /
  1.00 completeness at ~36% lower latency than its larger sibling `deepseek-v4-pro` — and is the
  production default.
- **Small local models remain insufficient.** `qwen2.5:32b` (~0.20 entity, malformed tool args)
  and `deepseek-r1:14b` (no tool-calling support) confirm ADR-0027's Qwen-class observation.

## Consequences

### Positive
- **Resolves ADR-0027's open question:** local+cloud model support is viable on DeepAgents.
- **No vendor lock-in:** swapping models/providers is a config change.
- **Privacy path preserved:** the llama.cpp backend keeps data on-box when required.

### Negative / limitations
- **Frontier-only for the hardest queries:** small local models can't yet match cloud frontier
  quality; the local-first privacy thesis is on hold pending affordable open-weights frontier
  hardware.
- **No Anthropic-style intelligent routing:** ADR-0027's automatic Haiku-for-tools cost
  optimisation is SDK-specific and not replicated here; model selection is explicit per run.

## Relationship to ADR-0027

This ADR **supersedes ADR-0027 on the local/multi-provider question** and **confirms its core
lesson from the other side**: adapter/proxy layers between an agent framework and its model
provider break provider-native features. The resolution is not "avoid local models" but "use a
framework with a native backend for each provider." ADR-0027 remains accepted for the Claude SDK
build's Anthropic-only decision.

## References

- [Phase 5 Overview](../phases/phase-5-production-deepagents/overview.md)
- [Multi-Model Evaluation](../phases/phase-5-production-deepagents/multi-model-evaluation.md)
- [ADR-0027](0027-intelligent-routing-and-model-selection.md)
- Repository: [ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
