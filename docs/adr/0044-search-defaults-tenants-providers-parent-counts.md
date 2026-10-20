# ADR-0044 — A Search That Cannot See Tenants: Search Defaults, Name Resolution, Parent Counts

## Status

**Accepted (October 2026).** A single web-chat turn — correct answer, no errors, 377 seconds
and nine tool calls for a question NetBox answers in one 2.7 KB call — exposed the largest
wasted-call category still in the local agent after ADR-0043: the MCP server's global search
could not see tenants, providers or manufacturers, so an organisation name returned eight
empty lists and the model guessed object classes. The local server now searches those types by
default, the filters skill resolves such names in one call and answers per-parent count
questions from the parent's count field, and the full v5 harness on the change is the fastest
of the four full runs on every aggregate.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`9699526` skill, prompt, server patch; `15a6902` full run; decision note
`docs/development/2026-10-19_search-cannot-see-tenants.md`; traces
`docs/traces/2026-10-19_netbox-session_search-types.*` and
`docs/traces/2026-10-20_netbox-benchmark-v5_search-types-full.md`; local MCP server commit
`d7809d9`, diff in `docs/development/2026-10-19_netbox-mcp-server-search-default-types.patch`)
**Bears on**: [ADR-0043](0043-filter-grammar-after-in-null-relations-ip-parent-graphql-ids.md)
(whose projected call saving this change delivered), [ADR-0041](0041-in-lookup-silently-ignored-reject-in-validator.md)
(the method), [ADR-0042](0042-expert-offloading-not-the-lever-pinned-weights-rejected-gpu-expert-cache.md)
(turn time is call count), [ADR-0039](0039-web-serving-layer-serialised-turns-server-token-counts.md)
(the per-call usage ledger that made the turn legible)

## Context

ADR-0043 removed every filter form NetBox silently ignores and found that wall time did not
move, because the model spent the freed calls on GraphQL. Its classification of wasted calls
counted errors, empty `get_objects` results and oversized results. It did not count empty
*searches*, because an empty search is a successful call.

On 2026-10-19 the web chat (thread `70a4c089…`) was asked *"How many power outlets does each
Halvorsen Logistics PDU provide, and how many PDUs are there?"* The web layer's usage ledger
(ADR-0039) split the 377 s turn exactly: 48 s cold prefix; ~100 s and six tool calls finding
out what "Halvorsen Logistics" is — `netbox_search_objects` twice, eight empty lists each time,
then `dcim.manufacturer`, `dcim.devicetype`, every manufacturer, a search for "PDU", and on the
seventh call `tenancy.tenant` — then 227 s fetching all 68 tenant devices (73 KB) and all 96
outlets (46 KB), two prompts of 32k and 49k tokens. The answer was right.

The MCP server's `DEFAULT_SEARCH_TYPES` was devices, sites, IP addresses, interfaces, racks,
VLANs, circuits and VMs. An organisation name matches none of them. Counted properly, the two
full v5 runs had **26 of 35 and 18 of 24 searches returning nothing**, across 16 and 14
questions, almost all "Halvorsen" or "Halvorsen Logistics" — more wasted calls than the
GraphQL grammar errors ADR-0043 fixed.

## Decision

1. **The local MCP server searches `tenancy.tenant`, `circuits.provider` and
   `dcim.manufacturer` by default** (appended to `DEFAULT_SEARCH_TYPES`; the server's own test
   updated and one added; diff kept beside the `__in` and change-log patches for one upstream
   report). Verified through the agent's wrapped tool: a default search for "Halvorsen
   Logistics" returns the tenant (id 14) first.
2. **The filters skill resolves an organisation name in one call** — `tenancy.tenant` (or
   `circuits.provider`, `dcim.manufacturer`) by `name__ic`, then `tenant_id` — and states that
   an empty global search is not evidence of absence and must not be followed by guessing
   another object class.
3. **"How many X does each Y have" is answered from the parent's `*_count` field with a role
   filter, never from the child list.** The PDU question is the worked example: one
   `dcim.device` call with `tenant_id`, `role=pdu` and `power_outlet_count` returns 12 rows in
   2.7 KB (verified live) against the 120 KB and four minutes of prefill the turn spent.
4. **System prompt**: one line for each. Both are guidance; neither can be a validator rule,
   because a search or a device list is a legitimate call.
5. **Gate**: the session ladder (all Halvorsen questions, so for once the ladder exercises the
   change), then the full v5 harness.

## Evidence

**The turn** (web chat, 2026-10-19, LangSmith `7743a8f5…`): 377 s, 8 model calls totalling
364 s, 9 tool calls totalling 15 s, 141k prompt tokens. Prefill 223 tok/s cold, decode 14.6.

**Session ladder**, same 11 questions and order as 6 and 18 October:

| | 2026-10-06 | 2026-10-18 | 2026-10-19 |
|---|---|---|---|
| total · tool calls · correct of 10 | 1,312 s · 20 · 7.0 | 1,444 s · 36 · 9.5 | **1,248 s · 16 · 9.5** |
| turn 4, outlets per PDU | 131 s · 7 | 240 s · 7 | 221 s · **4**, tenant resolved by the first search |
| turn 6, PDUs with an outlet in use | 27 s · 0 | 117 s · 12 | 22 s · 0 |

**Full v5 harness** (`…-v5-search-types-full-ba60270e`, same server, harness, evaluators and
judge as the two October runs):

| | 2026-09-23 | 2026-10-17 server | 2026-10-18 filters | **2026-10-20 search** |
|---|---|---|---|---|
| wall, first question to last | 6.78 h | 5.30 h | 5.26 h | **5.14 h** |
| agent time, sum of 90 | 19,345 s | 14,396 s | 14,248 s | **13,782 s** |
| mean · median per question | 215 · 153 s | 160 · 120 s | 158 · 113 s | **153 · 97 s** |
| simple · medium · advanced mean | 109 · 206 · 331 | 97 · 168 · 216 | 91 · 173 · 211 | **77** · 163 · 220 |
| tool calls per question | 5.52 | 4.88 | 4.99 | **4.49** |
| correctness · completeness | 0.906 · 0.944 | 0.922 · 0.944 | 0.928 · 0.978 | 0.917 · 0.972 |
| errors | 0 | 0 | 0 | 0 |

**Where the calls went**, 18 against 20 October: tool calls 449 → **404**; searches returning
nothing 18 → **6**, of which with default types 12 → **0**; searches that found something 6 →
11; tenant guessed as provider/manufacturer/device type 4 → 1; GraphQL queries ·
introspections 53 · 30 → **17 · 6**, rejected by NetBox 25 → **4**; `power_outlet_count` used
0 → 2; tool results over 20k characters 59 → 58 (4.9M characters either way).

## Consequences

### Positive
- **The fastest of the four full runs on every aggregate** — wall, sum, mean, median, simple
  tier, calls per question — at correctness and completeness inside the judge's day-to-day
  spread. Across the sequence: agent time 19,345 → 14,396 → 14,248 → 13,782 s; calls per
  question 5.52 → 4.88 → 4.99 → 4.49. The server bought the first 26%; the two guidance
  changes together another 4% of time and 8% of calls.
- **The targeted pattern is gone from a full run**: no default-type search returned empty.
- **GraphQL use fell back on its own** (53 → 17 queries) once the REST path resolved the name
  directly — the expansion ADR-0043 triggered reversed without any routing change, and with it
  most of the field-name rejections ADR-0043 could not reach.
- **The classification method gains a category.** Empty searches are successful calls and must
  be counted explicitly; the earlier audit missed the largest bucket by looking only at errors.

### Negative / limitations
- **Tool-result size did not move** — 58 results over 20k characters per run, as before. The
  per-parent-count guidance was followed twice; the whole-list habit (fetch all tenant devices,
  then the children) is the next lever and is guidance-only.
- **One run each way.** Correctness 0.917 against 0.928 the run before is five questions down
  and nine up — judge noise, not a signal, but not evidence of improvement either.
- **The server change is local**, like the two before it; the upstream server still ships the
  eight-type default. Three local commits are now waiting on one report.
- **A new slug trap appeared** in the run (`role=hypervisor`; the slug is `hypervisor-host`),
  handled by the existing 400 recovery.

## Relationship to ADR-0043

ADR-0043 projected a 20–30 call saving from removing wasted filter forms and measured none,
because the model spent the freed budget on GraphQL. This change is what that projection was
actually describing: once the name the question starts with resolves in one call, the agent
makes fewer calls *and* stops reaching for the more expensive tool. The lesson about where turn
time lives is unchanged — it is the model's plan — but this is the first guidance change that
moved the plan in the measured direction, and it did so by fixing the first call of the plan
rather than the later ones.

## References

- [Phase 2 → MCP Integration — Global search](../phases/phase-2-netbox/mcp-integration.md#4-global-search-search_objects)
- [ADR-0043](0043-filter-grammar-after-in-null-relations-ip-parent-graphql-ids.md) ·
  [ADR-0041](0041-in-lookup-silently-ignored-reject-in-validator.md) ·
  [ADR-0039](0039-web-serving-layer-serialised-turns-server-token-counts.md)
- Repository: `src/skills/netbox-mcp-filters/SKILL.md` (*RESOLVING A TENANT, PROVIDER OR
  MANUFACTURER NAME*, *PER-PARENT COUNTS*), `src/agents/netbox_agent.py`;
  `netbox-mcp-server/src/netbox_mcp_server/server.py` (`DEFAULT_SEARCH_TYPES`, local commit `d7809d9`)
- Upstream: `netboxlabs/netbox-mcp-server` 1.0.0 (`DEFAULT_SEARCH_TYPES`); NetBox 4.3.3 device
  serializer (`power_outlet_count` and the other `*_count` fields)
