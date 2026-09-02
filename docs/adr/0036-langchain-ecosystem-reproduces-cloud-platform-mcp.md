# ADR-0036 — The LangChain Ecosystem Reproduces the Cloud Platform MCP (Self-Hosted)

## Status

**Accepted** — Research + adoption plan (September 2026)
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents) (`docs/development/2026-08-10_langchain-ecosystem-vs-netbox-cloud-platform.md`)

## Context

NetBox Labs' Cloud-only **Platform MCP Server** advertises an "agent-native" bundle: ~100 tools,
sandboxed **Code Mode**, dynamic model discovery, tier-enforced tool registration, an Agent Skills
knowledge layer, and structured self-correcting errors. The question for a self-hosted, read-only,
privacy-mandated build: **how much of that is reproducible on the current (Aug 2026)
LangChain / LangGraph / DeepAgents / LangSmith ecosystem?** A five-agent survey mapped each Cloud
capability to its closest open-source, self-hostable equivalent.

## Decision

**Reproduce the Cloud Platform MCP's *query-processing* value from open-source, self-hostable
pieces; skip every hosted/paid convenience under the privacy mandate.**

Capability-by-capability verdict:

| Cloud capability | Self-hostable reproduction | Verdict |
|---|---|---|
| **Code Mode** (sandboxed executor) | `langchain-quickjs` PTC | **Skip** — reconfirms [ADR-0032](0032-quickjs-ptc-deferral.md); Anthropic's own τ²-bench shows sequential single-call workloads gain nothing (~8% more cost). The GraphQL tool is the right lever for round-trips. |
| The round-trip problem Code Mode targets | **Read-only GraphQL tool** ([ADR-0034](0034-read-only-graphql-complementary-read-path.md)) | **Already shipped** — server-side join |
| ~100 tools without context bloat | `LLMToolSelectorMiddleware` · DeepAgents subagent partitioning · `langgraph-bigtool` | Available, native — deferred until tool surface grows |
| Dynamic model discovery | Generate tools from NetBox OpenAPI/GraphQL **at MCP-server startup** | Server-side (LangChain can't add tools mid-run, #33808) |
| Structured self-correcting errors | **`RubricMiddleware`** + structured errors emitted by our MCP server | High value — the OSS equivalent |
| Agent Skills knowledge layer | **DeepAgents Skills** (open `agentskills.io` spec) | Already in use |
| NL → correct structured query | Curated GraphQL operations + schema-RAG + tool-arg validation | Highest correctness leverage |
| Model routing / handoff | **DeepAgents subagent `task()`** (the Aug-2026 LangChain "SRE agent" template) | The blueprint for the next feature |

**Skip (hosted/paid, fail the privacy mandate):** Managed Deep Agents (US-cloud only), LLM Gateway
(no local provider), Context Hub (use own git repo), hosted LangSmith. Self-hosted eval/observability
parity comes from the MIT `openevals` / `agentevals` packages + OpenTelemetry export.

## Consequences

### Positive
- The Cloud product's genuinely novel agent-facing ideas — portable **Skills** and **Code Mode** — are
  the *easiest* to reproduce (Skills already in use; Code Mode is the already-investigated PTC pattern).
  The hard part is tool *breadth*, which the read-only workload does not yet need.
- Produced a concrete, prioritized adoption plan: (1) the deepagents 0.7.5 upgrade ([ADR-0035]),
  (2) model-handoff routing via subagents, (3) `RubricMiddleware`, (4) strengthen NL→query.
- Everything on the plan is OSS and on-prem — consistent with the strict data-privacy mandate.

### Negative / limitations
- The Cloud product's *managed* convenience (hosting, tiered RBAC sessions) is genuinely not
  reproducible self-hosted — but it is orthogonal to query-processing quality.
- LangChain cannot add/remove tools after agent creation (#33808), so "dynamic discovery" must be
  done server-side at startup, not continuously as the Cloud product does it.

## Relationship to prior ADRs

Reconfirms **ADR-0032** (Code Mode / PTC deferral) from the ecosystem-wide vantage; triggered
**ADR-0035** (the 0.7.5 upgrade); and frames **ADR-0034** (GraphQL) as the correct lever for the
round-trip problem Code Mode claims to solve.

## References

- [Phase 5 → LangChain Ecosystem vs Cloud Platform MCP](../phases/phase-5-production-deepagents/langchain-ecosystem.md)
- [ADR-0032 — QuickJS / PTC Deferral](0032-quickjs-ptc-deferral.md)
- [ADR-0034 — Read-Only GraphQL Complementary Read Path](0034-read-only-graphql-complementary-read-path.md)
- [ADR-0035 — DeepAgents 0.7.5 Upgrade](0035-deepagents-0.7.5-upgrade.md)
- Repository: `docs/development/2026-08-10_langchain-ecosystem-vs-netbox-cloud-platform.md`
