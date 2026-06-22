# ADR-0032 — QuickJS / Programmatic Tool Calling: Deferral

## Status

**Accepted (Deferred)** — Investigated and not adopted (June 2026)
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents) (`tests/spike/`)

## Context

DeepAgents 0.6 introduced a QuickJS-based **Code Interpreter / Programmatic Tool Calling (PTC)**
middleware: the agent can write JavaScript that composes tool calls inside a sandbox
(`tools.netboxGetObjects(...)`), instead of round-tripping every intermediate result through the
model. It was an attractive lever for the slow multi-call NetBox queries (the VLAN-deployment class).

## Decision

**Investigate with verification spikes; do not adopt now; record explicit re-trigger conditions.**

Three spikes were run against the real NetBox MCP agent on `deepseek-v4-flash:cloud`:

| Spike | Question | Result |
|---|---|---|
| 1 | Do MCP tools bridge into the JS sandbox? | ✅ Yes — auto-exposed as `tools.netboxGetObjects(...)`, real round-trip, one outer tool call. |
| 2 | Does the existing filter-recovery middleware still observe errors made *inside* the sandbox? | ✅ Yes — it wraps the tool call itself (an unexpected positive; no skill rewrite needed). |
| 3 | Does it reduce wall time on the heavy VLAN query? | ❌ **No.** The model never chose to invoke the `eval` tool on this workload, and merely having it available cost **+13.7%** latency from prompt bloat. |

## Rationale

PTC's value comes from **fan-out / parallelism across many tools, or large-result filtering** —
none of which a **single-source, sequential, 4-tool** NetBox agent has. The dependency-chained
nature of the queries ("resolve tenant → its sites → VLANs → prefixes") means there is nothing to
parallelise, so the model correctly declines to use the interpreter. This independently reproduces
the vendor's own published finding that *sequential single-call workflows do not benefit* from PTC.

It also corroborates the **no-subagents** finding inherited from the ancestor build: the same
workload shape that doesn't benefit from subagent delegation doesn't benefit from code-orchestrated
tool calls, for the same reason.

## Re-trigger conditions (when to revisit)

Adopt PTC only when the agent's workload acquires one of:

- **≥ ~10 tools** (the threshold where token savings from PTC begin to appear),
- **≥ 2 independent data sources queryable in parallel** (e.g. NetBox + Grafana + a CMDB — the
  canonical `Promise.all` fan-out case),
- **cross-source joins composed in code**, or
- **large result sets needing pre-filtering** before the model sees them.

Spikes 1 and 2 (mechanism + error recovery) do not need re-running; only a wall-time re-measurement
would. A future multi-source phase is the natural place to revisit.

## Consequences

### Positive
- A clean, evidence-backed "not now, and exactly when to revisit" — avoids adopting a feature that
  would have *increased* latency on the current workload.
- Confirmed the filter-recovery middleware survives the PTC path (useful if adopted later).

### Negative
- The slow multi-call query class is not accelerated by this lever; other approaches (e.g. an
  MCP-server-side composite query, or teaching parallel native tool calls) are the alternatives.

## References

- [Phase 5 → Lessons Learned](../phases/phase-5-production-deepagents/lessons-learned.md)
- [ADR-0031 — DeepAgents 0.6 Upgrade](0031-deepagents-0.6-upgrade-and-harnessprofile-workaround.md)
- Repository: `tests/spike/` (the three spike scripts) and
  `docs/development/2026-06-03_quickjs-code-interpreter-research.md`
