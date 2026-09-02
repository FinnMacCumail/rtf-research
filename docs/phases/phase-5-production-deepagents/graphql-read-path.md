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

**Finding (short version)**: Yes, with one added tool and a routing skill — a **measured,
3×-replicated correctness win** on exactly the queries that were hallucinating, at **no tool-call
cost** once the framework and routing were tuned. Now **merged to mainline** (PR #1).

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
reference-grounded correctness judge (see [Evaluating for Correctness](evaluation-correctness.md)).
The result evolved across three measurement rounds — each round removed a caveat from the last, and
this arc is itself the finding:

| Round | Combined correctness | Tool-call cost | Over-routing | Confidence |
|---|---|---|---|---|
| **1. First A/B (deepagents 0.6.10, 1 run)** | 0.667 → 0.75 | **~2× more** (flash 15.7 → 27.0) | present (flash device-detail 1.0 → 0.0) | single run |
| **2. Re-measured on 0.7.5 (1 run)** | 0.667 → 0.75 | **vanished** — cost-neutral / cheaper (flash 18.5 → 12.8) | still present | single run |
| **3. + routing tightening, 3× replicated** | **→ ≈0.82** | cost-neutral | **fixed** (device-detail flash 3/3 = 1.0, pro 2/3) | **3 runs** |

Three things happened between the first A/B and the shipped result:

- **The cost objection dissolved on the framework upgrade.** Under [deepagents 0.7.5](0-7-5-upgrade.md)
  its leaner default prompts made the agent far more efficient with the GraphQL path — the ~2×
  tool-call penalty seen on 0.6.10 did not recur (it became cost-neutral, sometimes cheaper). The
  "trades round-trips for correctness" tradeoff was a 0.6.10 artifact, not intrinsic to GraphQL.
- **The one regression was diagnosed and fixed.** GraphQL over-routed *simple single-object*
  lookups (e.g. "show device X's location, IPs, tenant") — a query that *looks* cross-model but is
  anchored on one object. Root cause: the routing guidance keyed on "nested/related data → GraphQL,"
  and one object's site+IPs+tenant *is* nested data, so the model followed the miscalibrated rule
  into the wrong tool. The fix reframed all routing guidance around the **anchor-object rule**:
  *count the anchor objects, not the models the answer touches — one named object → MCP, even across
  models.* Trajectory-verified: device-detail now routes to `netbox_get_objects`, and the two MCP-only
  runs scored 1.0 while the one run that slipped to GraphQL scored 0.5 (MCP→right, GraphQL→wrong on a
  single object — the mechanism proven, not inferred).
- **The win was replicated 3×.** The per-question signal that always held: the site-comparison
  IP-allocation trap — which hallucinated a *different fabricated utilization %* on every MCP-only run
  (7.7 / 17.6 / 100 / 23.2%) — scored 0.5 → 1.0 correctness on both models with GraphQL, because the
  server-side join plus prefix-membership reasoning avoids the "180 global IPs misattributed per-site"
  error. Across 3× replication the cross-domain queries kept routing to GraphQL every run; the
  stabilized combined correctness settled at **≈0.82** (MCP-only 0.667 → GraphQL 0.75 →
  GraphQL + tightened ≈0.82 — the best configuration measured).

## Verdict

GraphQL is a **complementary read path** for cross-domain / aggregation queries — a **confirmed
correctness win at no tool-call cost** under 0.7.5, with simple single-object lookups kept on the MCP
tools by the anchor-object routing rule. **Merged to mainline (PR #1).** As with the QuickJS deferral
and the local-model finding, the deliverable is the **measured** verdict — and here the three-round
arc (cost objection dissolved on upgrade → regression diagnosed and fixed → win replicated) is the
lesson: a single-run A/B with an open caveat is a *hypothesis*, not a result. Two honest residuals,
neither a routing bug: pro's device-detail routing is 2/3 deterministic (the mechanical
`LLMToolSelectorMiddleware` gate is the escalation if airtight routing is ever required — not needed
at 2/3), and `tenant-site-summary` is a persistent model-accuracy weak spot unrelated to GraphQL.

The strategic point: this capability is reproducible **on-prem, read-only, and private** — the
Cloud product's cross-domain reads without shipping infrastructure data to a managed service, which
the project's privacy mandate forbids.

**See also**: [Evaluating for Correctness](evaluation-correctness.md) ·
[DeepAgents 0.7.5 Upgrade](0-7-5-upgrade.md) ·
[LangChain Ecosystem vs Cloud Platform MCP](langchain-ecosystem.md) ·
[Multi-Model Evaluation](multi-model-evaluation.md) ·
[Lessons Learned](lessons-learned.md) ·
[Research Methods → Benchmarking](../../methods/benchmarking.md) ·
[ADR-0034](../../adr/0034-read-only-graphql-complementary-read-path.md) ·
[ADR-0035](../../adr/0035-deepagents-0.7.5-upgrade.md) ·
[ADR-0036](../../adr/0036-langchain-ecosystem-reproduces-cloud-platform-mcp.md) ·
[ADR-0032](../../adr/0032-quickjs-ptc-deferral.md)
