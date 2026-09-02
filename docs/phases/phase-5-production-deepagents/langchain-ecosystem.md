# Phase 5: Reproducing the Cloud "Platform MCP Server" on the Open-Source LangChain Stack

## Research Overview

**Research Question**: NetBox Labs sells a Cloud-only **Platform MCP Server** that makes
infrastructure "agent-native" — ~100 tools, a sandboxed **Code Mode**, dynamic model discovery,
tier-enforced tool registration, an Agent Skills knowledge layer, and structured self-correcting
errors. For a self-hosted, read-only, privacy-mandated build: **how much of that is reproducible on
the current (Aug 2026) LangChain / LangGraph / DeepAgents / LangSmith ecosystem — and what should
actually be adopted?**

**Method**: five parallel deep-research agents, each mapping one dimension (Code Mode, large tool
sets, routing/handoff, current DeepAgents/blog, NL→structured-query) against the Cloud product, all
primary-source-cited.

**Finding (short version)**: the query-processing value is **largely reproducible from open-source,
self-hostable pieces** — and the Cloud product's two genuinely novel ideas (portable Skills, Code
Mode) are the *easiest* to reproduce. What is not reproducible is the *managed convenience*, which is
orthogonal to answer quality. Every hosted/paid piece is skipped under the privacy mandate.

**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`docs/development/2026-08-10_langchain-ecosystem-vs-netbox-cloud-platform.md`)

## Capability-by-capability map

| Cloud capability | Closest self-hostable reproduction | Verdict |
|---|---|---|
| **Code Mode** (sandboxed executor, "75% fewer round-trips") | `langchain-quickjs` PTC middleware | **Skip** — see below |
| The round-trip problem Code Mode targets | **Read-only GraphQL tool** ([GraphQL Read Path](graphql-read-path.md)) | **Already shipped** |
| ~100 tools without context bloat | `LLMToolSelectorMiddleware` · DeepAgents subagent partitioning · `langgraph-bigtool` | Native; deferred until the tool surface grows |
| **Dynamic model discovery** | Generate tools from NetBox OpenAPI/GraphQL **at MCP-server startup** | Server-side (LangChain can't add tools mid-run) |
| Tier-enforced tool registration | Filter the tool list per role at agent-build time | Static-per-session |
| **Structured self-correcting errors** | `RubricMiddleware` + structured errors from our own MCP server | The OSS equivalent — high value |
| **Agent Skills knowledge layer** | DeepAgents Skills (open `agentskills.io` spec) | **Already in use** |
| **NL → correct structured query** | Curated GraphQL operations + schema-RAG + tool-arg validation | Highest correctness leverage |
| Model routing / handoff | DeepAgents subagent `task()` (the LangChain "SRE agent" template) | The blueprint for the next feature |

## The headline: Code Mode is the wrong lever for this workload

The Cloud product's "75% fewer tool round-trips" comes from its **~100-tool catalog + batch tasks** —
neither of which a read-only, single-source, dependency-chained NetBox workload has. This
independently **reconfirms the earlier QuickJS/PTC deferral**
([ADR-0032](../../adr/0032-quickjs-ptc-deferral.md)): Anthropic's own τ²-bench shows that sequential
single-call workflows gain nothing from programmatic tool calling and cost ~8% more — matching this
project's own spike (+13.7% latency, model never chose the code tool). The right lever for the
round-trip problem was the **read-only GraphQL tool** (a server-side join), not a sandbox.

## What's worth adopting (all open-source, all on-prem)

The research produced a prioritized plan, sequenced by what unblocks what:

1. **DeepAgents 0.7.5 upgrade** — tracks the subagent + `RubricMiddleware` APIs the rest needs.
   Done: [DeepAgents 0.7.5 Upgrade](0-7-5-upgrade.md) / [ADR-0035](../../adr/0035-deepagents-0.7.5-upgrade.md).
2. **Model-handoff routing** via DeepAgents subagents — the LangChain "autonomous SRE agent" (Aug 2026)
   is almost the exact architecture: tiered models by complexity, tool-scoped subagents. A fast local
   model fields simple queries; a heavier model + GraphQL subagent handles multi-hop.
3. **`RubricMiddleware`** — structured self-correction gated on project criteria (the OSS parallel to
   the Cloud's "self-correcting errors").
4. **Strengthen NL→query** — curated cross-domain GraphQL operations + tighter tool-arg schemas.

## What's skipped (hosted/paid — fails the privacy mandate)

Managed Deep Agents (US-cloud only), the LLM Gateway (no local-model provider), Context Hub (a git
repo serves the same purpose), and hosted LangSmith. Self-hosted eval/observability parity comes from
the MIT `openevals` / `agentevals` packages plus OpenTelemetry export — the project already runs its
model-matrix eval and reference-grounded correctness judge locally, so this is a natural fit.

## Takeaway

The "agent-native" bundle a vendor sells as a managed product decomposes, on inspection, into pieces
that are mostly open and self-hostable — with the genuinely novel agent-facing ideas (Skills, Code
Mode) being the *easiest* to reproduce and the *managed hosting* being the only thing you truly can't
rebuild. For a privacy-mandated build, that is the whole point: frontier-grade agent capability **and**
data that never leaves the network are not mutually exclusive — they are an engineering plan.

**See also**: [DeepAgents 0.7.5 Upgrade](0-7-5-upgrade.md) ·
[GraphQL Read Path](graphql-read-path.md) ·
[Lessons Learned](lessons-learned.md) ·
[ADR-0036](../../adr/0036-langchain-ecosystem-reproduces-cloud-platform-mcp.md) ·
[ADR-0032](../../adr/0032-quickjs-ptc-deferral.md)
