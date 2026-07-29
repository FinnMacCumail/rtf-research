# Phase 5: A Read-Only GraphQL Path — Recreating Cloud "Agent-Native" Reads on Community NetBox

## Research Overview

**Research Question**: NetBox Labs' commercial **Platform MCP Server** makes infrastructure
"agent-native" — ~100 tools, a sandboxed "Code Mode", and RBAC-scoped sessions — but it is **NetBox
Cloud only**. On a self-hosted community instance (and under a strict data-privacy mandate that
rules out shipping topology to a managed service), how much of that capability can a DeepAgents build
reproduce — and does it actually help?

**Scope decision**: **read-only**. No CRUD, ever. This removes most of the commercial surface
(writes, bulk ops, branching, the governance layer — all of which exist to make *writes* safe) and
leaves the question that matters for a query agent: can the agent do **cross-domain / nested reads**
— the one thing the community MCP server structurally cannot?

**Finding (short version)**: Yes, with one added tool and a routing skill — and it is a **measured
correctness win** on exactly the queries that were hallucinating, at a tool-call cost.

**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`src/tools/netbox_graphql.py`, `src/skills/netbox-graphql/`)

## What the Cloud product is, and what's actually reproducible

The Platform MCP Server has three layers. Their reproducibility on self-hosted NetBox differs
sharply:

| Layer | Cloud product | Self-hostable? |
|---|---|---|
| **Knowledge** — "Agent Skills" | Open-source Markdown skills (Apache-2.0, `netboxlabs/skills`) | ✅ Fully — portable to any agent |
| **Access** — ~100 tools + Code Mode | Managed server; Code Mode = sandboxed Python executor | ◑ The *reads* map to community REST/GraphQL; Code Mode is the QuickJS/PTC pattern already investigated ([ADR-0032](../../adr/0032-quickjs-ptc-deferral.md)) |
| **Governance** — RBAC sessions, Branching, Validation | Cloud / commercial products | ✗ Mostly not (and, being write-safety machinery, **irrelevant to a read-only agent**) |

The two genuinely novel agent-facing ideas (portable Skills, and Code Mode) are the *easiest* to
reproduce; the hard part is tool *breadth*. For read-only cross-domain queries, the community
**GraphQL API** (read-only, Strawberry-based since NetBox 4.3, present in the open-source edition)
collapses the whole "~100 tools" question into **one** high-value tool — a server-side join in a
single request.

## Built PRP-first, in two phases

The work used the project's **Project Requirements Package (PRP)** workflow — a curated
requirements-plus-context spec generated against the live codebase, reviewed, then executed — split
deliberately by *validation method*:

- **PRP 1 — the mechanism (deterministic, unit-tested).** A standalone, read-only `netbox_graphql`
  tool + a `netbox_graphql_schema` introspection tool. Read-only is enforced by **AST inspection**
  (`graphql-core` parses the document; mutations/subscriptions rejected before any HTTP call) — never
  regex. Because the endpoint is read-only server-side anyway, the *primary* safety surface is
  **query cost** (depth / size / timeout limits). The tools are standalone, bypassing the MCP
  filter-validator by design. 23 unit tests; live-verified against the instance.
- **PRP 2 — the behaviour (empirical, A/B-tested).** A routing skill teaching *when* to prefer
  GraphQL, plus a one-line system-prompt change permitting it. Validated by an A/B evaluation, not
  unit tests.

Splitting on validation method kept a clean seam: PRP 1 proves the tool works; PRP 2 measures
whether the agent uses it well.

## The generalization problem (and how introspection solves it)

A real risk: a skill written around the benchmark's six queries would only handle *those* object
types. NetBox has 100+ types plus plugins — enumeration is impossible and would overfit.

The resolution is to teach **grammar + runtime discovery, not a schema**:

- **Universal grammar** (identical for every type): root fields are `snake_case(model)_list`; types
  are `<Model>Type`; IDs are bare (`{id: 6}`); strings use lookup objects
  (`{name: {exact: …}}` / `{in_list: […]}`); nested relationship filters work.
- **`netbox_graphql_schema(<Type>)` is the generalization engine** — for any unfamiliar type the
  model introspects the *live* schema (including plugins) first, then writes the query. This mirrors
  how the existing `netbox-mcp-filters` skill teaches filter *grammar* rather than enumerating
  objects.

This was validated out-of-band: on an **out-of-benchmark** query (circuits → provider →
terminations — a domain absent from the dataset), the agent answered via a *pure* introspect-then-
query path with zero MCP fallback. The skill generalizes; it does not overfit.

## The A/B result

GraphQL-enabled vs MCP-only, same models, same corrected `netbox-benchmark-v4`, same
reference-grounded correctness judge (see [Evaluating for Correctness](evaluation-correctness.md)):

| Model | correctness (MCP → GraphQL) | tool calls (MCP → GraphQL) |
|---|---|---|
| deepseek-v4-pro | 0.65 → **0.883** (+0.23) | 10.5 → 19.5 |
| deepseek-v4-flash | 0.75 → 0.75 (flat) | 15.7 → 27.0 |

The signal is per-question, and it lands on the trap:

- **The site-comparison IP-allocation query — which hallucinated a different fabricated utilization %
  on every MCP-only run — scored 0.5 → 1.0 correctness on BOTH models with GraphQL.** The
  server-side join plus prefix-membership reasoning avoids the "180 global IPs misattributed
  per-site" error that the MCP decomposition kept making. This reproduces, in the harness, a win
  first seen in an interactive trace.
- **Cost: ~2× tool calls.** GraphQL trades round-trips for a correct join (including a schema-
  discovery tax).
- **Over-routing side-effect**: flash applied GraphQL to a *simple* single-object lookup that MCP
  handled fine and got a field wrong (1.0 → 0.5), cancelling its site-comparison gain and leaving
  its aggregate flat. Routing is *soft* guidance, not a hard switch.

## Verdict

Keep GraphQL as a **complementary read path** for cross-domain / aggregation queries; keep simple
single-object lookups on the MCP tools; do **not** make GraphQL the default. The costs (tool calls,
occasional over-routing) are addressable by tightening the routing skill, not by dropping the tool —
and the correctness gain lands precisely on the query class that most needed it. As with the QuickJS
deferral and the local-model finding, the deliverable is the **measured** verdict, not adoption for
its own sake. A publishable aggregate needs ≥3 runs per arm (single-run variance caveat from the
[correctness work](evaluation-correctness.md) applies); the per-question correctness win is robust.

The strategic point: this capability is reproducible **on-prem, read-only, and private** — the
Cloud product's cross-domain reads without shipping infrastructure data to a managed service, which
the project's privacy mandate forbids.

**See also**: [Evaluating for Correctness](evaluation-correctness.md) ·
[Multi-Model Evaluation](multi-model-evaluation.md) ·
[Lessons Learned](lessons-learned.md) ·
[Research Methods → Benchmarking](../../methods/benchmarking.md) ·
[ADR-0034](../../adr/0034-read-only-graphql-complementary-read-path.md) ·
[ADR-0032](../../adr/0032-quickjs-ptc-deferral.md)
