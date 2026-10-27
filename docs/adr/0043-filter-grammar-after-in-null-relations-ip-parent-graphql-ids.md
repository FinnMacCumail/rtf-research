# ADR-0043 — Filter Grammar After `__in`: Null Relations, IP `parent`, GraphQL IDs, Change-Log Verbs

## Status

**Accepted (October 2026).** Extends ADR-0041 from one silently-ignored filter to the class.
Mining the two 90-question v5 runs on the local server found 41 of 439 tool calls (9%, about
23 minutes of a 5.3 h run) spent on filters NetBox silently ignores, inverts or rejects, and
on GraphQL grammar the agent's own skill was teaching wrongly. Each suspect form was verified
live against NetBox 4.3.3; the agent's validator, its GraphQL tool and both skills now reject
or avoid them with the working form in the message. A full v5 run on the change removed every
targeted REST mistake and every GraphQL grammar rejection, raised correctness and completeness,
and left wall time unchanged — because the model, given a corrected GraphQL skill, used GraphQL
twice as often.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`8a73ccf` rules, skills, tests; `ed8adf8` full run; `e06a8c8` follow-ups; decision note with the
live verification in `docs/development/2026-10-18_filter-naming-after-in.md`; traces
`docs/traces/2026-10-19_netbox-benchmark-v5_filter-naming-full.md` and
`docs/traces/2026-10-18_netbox-session_filter-naming.*`; server-docstring patch
`docs/development/2026-10-19_netbox-mcp-server-changelog-action.patch`)
**Bears on**: [ADR-0041](0041-in-lookup-silently-ignored-reject-in-validator.md) (the first
instance of the class and the method — verify by result set, not status code),
[ADR-0042](0042-expert-offloading-not-the-lever-pinned-weights-rejected-gpu-expert-cache.md)
(established that turn time is now call count, not the server), [ADR-0034](0034-read-only-graphql-complementary-read-path.md)
(the GraphQL skill whose grammar section was wrong), [ADR-0033](0033-reference-grounded-correctness-evaluator.md)

## Context

ADR-0042 left the local agent with a server that prefills 2.2× faster and a turn whose cost is
dominated by how many model calls the agent makes and how large the tool results are. ADR-0041
had shown that one wrong filter form costs not a 400 but a silent unfiltered result, and that
removing it halves the calls on the question that hits it. The obvious next question was whether
`__in` was alone.

The two full v5 runs (2026-09-23 and 2026-10-17, 497 and 439 tool calls) are the evidence base.
Every `netbox_get_objects` call that errored, returned nothing and was retried with different
keys, or returned the whole table with a filter present, and every `netbox_graphql` call that
returned `errors`, was classified; every suspect filter was then sent to NetBox directly with the
read-only token.

## Decision

1. **"No `<relation>` assigned" is `{"<relation>_id": null}` only.** *(Corrected by
   [ADR-0045](0045-schema-driven-filter-validation-null-wire-tool-results-twice.md): right about
   NetBox, wrong about the wire — `requests` drops a `None` value, so the JSON null never reached
   NetBox and the "correct uses" counted below were unfiltered; the wrapper now sends the string
   `"null"`.)* The validator rejects
   `<relation>__empty` (silently ignored) and `<relation>__n: null` (returns the complement),
   and its suggestion for `__null` / `__isnull` names the `_id: null` form. A `RELATION_FIELDS`
   set plus any `*_id` key decides what counts as a relation; `__empty` on a string field
   (`serial__empty`) stays valid.
2. **On `ipam.ipaddress`, `within` / `within_include` / `without` / `contains` are rejected**
   (the first silently returns every address, the rest 400); the address-in-prefix filter is
   `parent`. `vrf` on ipam types is rejected (it wants the route distinguisher); `vrf_id` is the
   key. The validator is now object-type aware for this.
3. **The GraphQL tool pre-checks grammar before any HTTP request**: the argument is `filters:`
   (not `filter:`, not a bare `cluster:`); `limit:`/`offset:` are `pagination: {...}`; an ID key
   takes one bare id — a list or a lookup object is rejected at any nesting depth; `<x>_id: null`
   is rejected as silently ignored; at most 10 aliases. The GraphQL skill's statement that
   `{id: [6, 7, 8]}` is valid is reversed: on this instance it fails with *"ID cannot represent a
   non-string and non-integer value"*, and six of October's twenty GraphQL failures were the
   model following that line.
4. **Change-log `action` is `create` / `update` / `delete`.** The upstream MCP server's tool
   description says `created, updated, deleted`; those are 400s. Rejected by the validator as a
   filter key or a direct argument; the local server copy's docstring is corrected and the diff
   kept beside the `__in` patch for one upstream report.
5. **Two things stay guidance, not rules.** A tenant name is never a `circuits.provider` name
   (a tenant's circuits are `circuits.circuit` by `tenant_id`), and an exact-name miss should be
   followed by one search, not a `name__ic` retry and then an unfiltered fetch. Both are
   legitimate calls, so the validator cannot reject them; the skill and prompt say so.
6. **Gate.** The session harness (ADR-0041's ladder) for regressions, then the full v5 harness
   for the measurement — the first time a filter change has been gated on the full set.

## Evidence

**Live verification** (NetBox 4.3.3, 2026-10-18; `dcim.device` has 141 objects, 15 without a
tenant; `ipam.ipaddress` has 616):

| filter | HTTP | count | verdict |
|---|---|---|---|
| `dcim/devices/?tenant_id=null` | 200 | **15** | the only working form |
| `?tenant__empty=true`, `?tenant__isnull=true` | 200 | 141 | **silently ignored** |
| `?tenant__n=null` | 200 | 126 | **the complement** |
| `ipam/ip-addresses/?parent=10.0.0.0/8` | 200 | 436 | correct |
| `?within=10.0.0.0/8` | 200 | 616 | **silently ignored** |
| `?without=...` | 400 | — | not a filter |
| `?vrf=hvl-guest` | 400 | — | wants the RD; `vrf_id` works |
| `core/object-changes/?action=create` | 200 | 3,158 | correct |
| `?action=created` | 400 | — | tool description's spelling |
| GraphQL `device_list(filters: {id: [146, 147]})` | — | error | "ID cannot represent a non-string and non-integer value" |
| GraphQL `filters: {tenant_id: null}` | — | 141 rows | **silently ignored** |
| GraphQL `filters: {name: {in_list: [...]}}`, `pagination: {limit, offset}` | — | ok | the working forms |

**Where the calls went** (every tool call in the 17 October run, before the change):

| category | calls | questions |
|---|---|---|
| GraphQL rejected by NetBox + the introspections around them | 20 + 21 | 9 |
| "no X assigned" as `__empty` / `__isnull` / `__null` / `__n: null` | 9 | 5 |
| tenant looked up as a provider | 7 | 5 |
| ip-address `within` / `without` / `vrf=` | 3 | 2 |
| slug-versus-name 400s | 2 | 2 |

**The full v5 run on the change** (`...-v5-filter-naming-full-8b847369`, same server, harness,
evaluators and judge as 17 October):

| | 2026-10-17 | 2026-10-18 |
|---|---|---|
| wall · agent time · median | 5.30 h · 14,396 s · 120 s | 5.26 h · 14,248 s · 113 s |
| questions faster than the previous run | — | 48 of 90 |
| correctness · completeness | 0.922 · 0.944 | **0.928 · 0.978** |
| tool calls per question | 4.88 | 4.99 |
| "no X assigned" fumble reaching NetBox · `_id: null` used | 4 · 1 | **0 · 13** |
| ip-address `within`/`vrf` reaching NetBox · filter-value 400s | 2 · 2 | **0 · 0** |
| tenant as provider | 7 | 2 |
| GraphQL queries · questions using GraphQL | 31 · 9 | 53 · 18 |
| GraphQL accepted · caught before HTTP · rejected by NetBox | 15 · 0 · 16 | **30** · 7 · 16 |
| ... rejections that are grammar · that are field names | 8 · 2 | **0** · 14 |
| change-log `action=created` 400s | 0 | 3 (now fixed) |

**Session harness** (same ladder as 2026-10-06): 0 validation errors, 9.5 of 10 against 7.0,
1,444 s against 1,312 s on 36 against 20 calls — turn 6 re-queried twelve times where October
answered from context, the planning variance the harness documents.

## Consequences

### Positive
- **Every targeted REST mistake is gone from a full run**, and the silent ones — the whole
  table returned for "devices with no tenant", every address returned for "addresses in this
  prefix" — can no longer become wrong answers. That is the ADR-0041 property extended to the
  class.
- **GraphQL grammar errors no longer reach NetBox.** The seven that the model still wrote were
  turned into one-step corrections with the fix named.
- **Correctness and completeness rose** (0.922 → 0.928, 0.944 → 0.978) with no loss elsewhere.
- **The method is now repeatable**: dump both runs' tool calls, classify, verify live, fix in
  validator + skill + prompt, gate on the session ladder, measure on the full set.

### Negative / limitations
- **No wall-time saving.** The projected 20–30 calls per run were freed and immediately spent:
  the corrected GraphQL skill reads as usable, so the model routed twice as many questions
  through GraphQL, each costing a schema call plus a query, and the residual GraphQL rejections
  are now field-name mistakes (`Cannot query field`, `not defined by type`) that a grammar check
  cannot know about. Validator rules buy correctness and recovery quality, not seconds.
- **The guidance-only items are only reduced.** The tenant-as-provider detour fell from 7 calls
  to 2.
- **The follow-ups are unit-tested but not re-gated** (change-log verbs, nested GraphQL ids):
  five calls per run is below what one run can resolve.
- **Every "silently ignored" row is a NetBox 4.3.3 fact** and must be re-verified on upgrade,
  as must the 10-alias cap and the `pagination` grammar.
- **The upstream MCP server now has two documentation defects on file here** (`id__in`,
  `action=created`) and no report filed yet.

## Relationship to ADR-0041 and ADR-0042

ADR-0041 found the first silently-ignored filter by accident, in a browser session, and fixed
it. This ADR applies its lesson — check the result set, not the status code — deliberately,
across every tool call of two full runs, and finds the rest. ADR-0042 is why it mattered: with
the server-side levers spent, the only place turn time can still move is the number of calls
the model makes, and a wasted call is the cheapest kind to remove. The full-run result then
sharpens ADR-0042's conclusion: even removing the wasted calls did not move wall time, because
the model spent the freed budget on a tool it now trusted more. Turn time is a planning
property before it is a server or a validator property.

## References

- [Phase 5 → GraphQL Read Path — grammar re-verified](../phases/phase-5-production-deepagents/graphql-read-path.md#grammar-re-verified-october-2026)
- [Phase 2 → MCP Integration](../phases/phase-2-netbox/mcp-integration.md) (examples corrected)
- [ADR-0041 — Reject `__in`](0041-in-lookup-silently-ignored-reject-in-validator.md) ·
  [ADR-0042 — Expert Offloading Is Not the Lever](0042-expert-offloading-not-the-lever-pinned-weights-rejected-gpu-expert-cache.md)
- Repository: `src/tools/netbox_tools.py` (`RELATION_FIELDS`, `CHANGELOG_ACTION_FIX`,
  object-type-aware `validate_filter`), `src/tools/netbox_graphql.py` (`_validate_filter_grammar`,
  `_check_filter_object`), `src/middleware/filter_recovery.py`, `src/skills/netbox-graphql/SKILL.md`,
  `src/skills/netbox-mcp-filters/SKILL.md`, `tests/test_filters.py`, `tests/test_netbox_graphql.py`
- Upstream: `netboxlabs/netbox-mcp-server` 1.0.0 `server.py` (`netbox_get_changelogs` docstring);
  NetBox 4.3.3 `ipam/filtersets.py` (`parent` on IPAddressFilterSet), `core/filtersets.py`
  (`ObjectChangeFilterSet.action` choices), strawberry GraphQL schema (`DeviceFilter.id: ID`)
