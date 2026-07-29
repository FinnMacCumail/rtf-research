# ADR-0034 — Read-Only GraphQL as a Complementary Read Path

## Status

**Accepted** — Adopted as a complementary path; not the default (July 2026)
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents) (`src/tools/netbox_graphql.py`, `src/skills/netbox-graphql/`)

## Context

NetBox Labs' commercial **Platform MCP Server** makes infrastructure "agent-native" (~100 tools, a
sandboxed "Code Mode", RBAC-scoped sessions) but is **NetBox Cloud only**. This project runs a
self-hosted community instance under a strict data-privacy mandate that rules out shipping topology
to a managed service. The community MCP server exposes four generic read tools and, critically,
**cannot express cross-model relationship filters** — the `netbox-mcp-filters` skill exists to work
around this by decomposing cross-domain questions into brittle multi-step sequences.

An appraisal of the Cloud product found its layers reproduce very unevenly on self-hosted NetBox:
the Knowledge layer (Agent Skills) is open-source and portable; the ~100-tool Access layer mostly
maps to the community REST/GraphQL API; the Governance layer is commercial and — being *write*-safety
machinery — irrelevant to a read-only query agent. NetBox's **GraphQL API** (read-only,
Strawberry-based since 4.3, present in the open-source edition) collapses "cross-domain reads" into a
single server-side join.

## Decision

**Add a read-only `netbox_graphql` tool (plus `netbox_graphql_schema` introspection) as a
complementary read path, taught by a routing skill; keep it strictly read-only and not the default.
Build it PRP-first in two phases split by validation method.**

- **Scope: read-only, no CRUD.** This is a hard constraint (privacy + safety), and it removes most of
  the Cloud surface (writes, bulk, branching, governance).
- **PRP 1 — mechanism (unit-tested).** Standalone tool; read-only enforced by `graphql-core` **AST
  inspection** (mutations/subscriptions rejected before any HTTP), never regex. Endpoint is read-only
  server-side, so the primary safety surface is **query cost** (depth/size/timeout). Bypasses the MCP
  filter-validator by design. 23 tests; live-verified.
- **PRP 2 — behaviour (A/B-tested).** A routing skill + one system-prompt line. The skill teaches
  **grammar + runtime introspection, not a fixed schema**, so it generalizes to any NetBox type
  (root = `snake_case(model)_list`; IDs bare; strings `{exact:}`/`{in_list:}`; nested filters; and
  `netbox_graphql_schema(<Type>)` for anything unfamiliar, including plugin objects).

## Evidence

**Generalization** was validated on an out-of-benchmark domain (circuits → provider → terminations,
absent from the dataset): the agent answered via a pure introspect-then-query path with zero MCP
fallback — confirming the skill teaches a method, not a memorised schema.

**A/B** (GraphQL-enabled vs MCP-only; same models; corrected `netbox-benchmark-v4`; reference-grounded
correctness judge from ADR-0033):

| Model | correctness (MCP → GraphQL) | tool calls |
|---|---|---|
| deepseek-v4-pro | 0.65 → **0.883** | 10.5 → 19.5 |
| deepseek-v4-flash | 0.75 → 0.75 | 15.7 → 27.0 |

The site-comparison IP-allocation query — which hallucinated a different fabricated utilization % on
every MCP-only run — scored **0.5 → 1.0 correctness on both models** with GraphQL: the server-side
join plus prefix-membership reasoning avoids the misattribution the MCP decomposition kept making.

## Consequences

### Positive
- Recreates the Cloud product's cross-domain reads **on-prem, read-only, and private**.
- A measured correctness win on exactly the query class that was hallucinating.
- Generalizes to any NetBox object via runtime introspection; does not overfit the benchmark.
- One high-value tool replaces the intent of "~100 tools" for the read use case.

### Negative / limitations
- **~2× tool calls** — GraphQL trades round-trips (and a schema-discovery tax) for a correct join.
- **Soft routing**: the model sometimes over-applies GraphQL to simple lookups (flash regressed a
  single-object query 1.0 → 0.5), cancelling its aggregate gain. Fixable by tightening the routing
  skill, not by dropping the tool.
- Aggregate figures need ≥3 runs per arm (single-run variance); the per-question win is robust.

## Relationship to ADR-0032

The Cloud "Code Mode" is the same sandboxed-code-orchestration pattern as the QuickJS/PTC middleware
deferred in ADR-0032 — and deferred for the same reason (a single-source sequential workload doesn't
benefit). GraphQL is the *right* lever for this workload: it does the join **server-side in one
request**, needing neither a sandbox nor a ≥10-tool surface. Where PTC was "not now," GraphQL is
"yes, complementary."

## References

- [Phase 5 → GraphQL Read Path](../phases/phase-5-production-deepagents/graphql-read-path.md)
- [Phase 5 → Evaluating for Correctness](../phases/phase-5-production-deepagents/evaluation-correctness.md)
- [ADR-0033 — Reference-Grounded Correctness Evaluator](0033-reference-grounded-correctness-evaluator.md)
- [ADR-0032 — QuickJS / PTC Deferral](0032-quickjs-ptc-deferral.md)
- Repository: `src/tools/netbox_graphql.py`, `src/skills/netbox-graphql/`, `PRPs/`,
  `docs/traces/2026-07-21_netbox-benchmark-v4_graphql.md`
