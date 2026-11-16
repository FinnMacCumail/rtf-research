# ADR-0047 — The Result Envelope: a Header on Every List Result, and a Probe That Names the Filter NetBox Dropped

## Status

**Accepted (November 2026).** Every list result from the NetBox MCP tools (`netbox_get_objects`,
`netbox_get_changelogs`) now opens with a one-line header stating the filters actually sent,
NetBox's `count`, the rows returned and whether the result is complete; an `INCOMPLETE:` line
gives the exact next offset when it is not; and when a filtered call returns exactly the
endpoint's unfiltered total, the wrapper re-sends each filter key with an impossible value and
reports which keys NetBox ignored (`WARNING`) or that all of them are honoured and the
population really is the whole table (`NOTE`). First of the three levers from the 27 October
review of NetBox Labs' Platform MCP server (envelope, composite reads, upstream reconciliation).
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`a966bd5` envelope, merge `b33a8ce`; `8d69ea1` full-run record; decision note
`docs/development/2026-11-09_result-envelope.md`; ladders
`docs/traces/2026-11-09_netbox-session_result-envelope.*` and `…_result-envelope-probe.*`;
full run `docs/traces/2026-11-10_netbox-benchmark-v5_envelope-full.md`; LangSmith
`…-v5-envelope-full-ced8482f` against `…-v5-strata-full-00b39a92`)
**Bears on**: [ADR-0045](0045-schema-driven-filter-validation-null-wire-tool-results-twice.md)
(the schema layer this closes the gap behind, and the JSON-null bug that is this failure's
canonical instance), [ADR-0041](0041-in-lookup-silently-ignored-reject-in-validator.md) (the
method: verify by result set, not status code — now applied at run time to every result),
[ADR-0043](0043-filter-grammar-after-in-null-relations-ip-parent-graphql-ids.md),
[ADR-0046](0046-strata-inference-server-same-weights-faster-agent.md) (the server both
compared runs used)

## Context

NetBox refuses a filter in two ways and neither of them is an error. A key the filterset does
not know is dropped and the request answers 200 with the unfiltered set; ADR-0045's schema
layer now rejects those names before they leave the process. A *valid* key whose value form
NetBox does not honour is dropped exactly the same way, and nothing downstream can see it: the
key passes the schema check, the HTTP status is 200, the body is well-formed, the rows are real.
The JSON-null bug of ADR-0045 is the canonical case — `{"tenant_id": null}` reached NetBox as
no filter and returned 141 devices to a tenant-scoped question through two full benchmark runs,
and the model, the judge and the author all took it for the answer. The same shape recurs with
boolean spellings (`has_primary_ip=no` is dropped; `false` works), with custom-field filters the
index does not carry, and with any key a NetBox upgrade stops honouring before the index is
rebuilt.

The October skill already told the model to compare `count` against `next` and never to report
a partial list as the total. The replay of the 18 and 19 October runs still found 25 silently
unfiltered calls among 533 scoped ones. The instruction was present; the information was not.
The model had no way of knowing that the device table holds 141 rows, so "141 matched" and
"the filter was dropped" were indistinguishable to it.

The only signal that survives a dropped value is arithmetic: a scoped call whose `count` equals
the endpoint's unfiltered total almost certainly matched nothing. "Almost" is the design
problem: on small endpoints a honoured filter frequently matches everything (all 8 outlet
templates belong to one device type; one seeding user made all 3,543 change-log entries in
this instance). A detector that warns on equal counts alone sends the model to re-query results
that were right.

## Decision

1. **A header on every paginated list result, as the first content block.** Line one is
   `NETBOX_RESULT scope=<filters sent, minus view keys> matched_count=<count>
   returned_count=<len(results)> complete=<next is null>`. `scope` is the filters dict after
   ADR-0045's normalisation with `limit`, `offset`, `fields`, `ordering`, `q`, `brief`,
   `export`, `format` removed: a key missing from it was never applied. The adapter's blocks
   and the structured artifact follow unchanged. About 90 characters per honoured call.
2. **`INCOMPLETE: <returned> of <count> returned. Fetch the rest with offset=<offset+returned>
   (limit=100), or raise limit; do not report a partial list as the total.`** whenever `next`
   is set — the pagination rule the skill states at length, as a field with the next call in it.
3. **The detector: equal counts trigger a probe, never a warning.** The unfiltered total comes
   from one `GET <endpoint>?limit=1&fields=id` (75 ms live), fetched on the first scoped call
   to that endpoint and cached ten minutes. When a scoped call's `count` is non-zero and at
   least that total, each scoped key is re-sent *alone* with an impossible value — `999999999`
   for numeric keys, `zz-no-such-value-zz` for strings, kept as a list when the model sent a
   list — which is precisely how the ADR-0045 research ground-truthed its replay. A honoured
   key answers 400 ("Select a valid choice") or fewer than the total; an ignored key answers the
   unchanged total. One GET per key, one retry, verdict cached per (endpoint, key) ten minutes.
4. **Three wordings, by verdict.** Ignored keys → `WARNING: filter(s) <keys> had NO EFFECT:
   NetBox ignores them on <object type> (a probe with an impossible value still returned all
   <total>). These rows are the UNFILTERED table, not the answer. Re-check the key names and
   value forms in the netbox-mcp-filters skill and re-query.` All honoured → `NOTE: the filters
   are honoured and happen to match every <type> (<count> of <total>); this result is usable as
   the answer.` Probe failed twice → the older "no effect or coincide" wording, which asks the
   model to check the form itself.
5. **Fail-open at every step.** No `count` in the payload, an object type the index does not
   map, a failed total fetch: no warning, counts still shown. Tools without NetBox's paginated
   shape (`netbox_search_objects`, `netbox_get_object_by_id`, GraphQL) and the
   `TOOL_VALIDATION_ERROR` / `TOOL_API_ERROR` paths are untouched.
6. **The skill and the system prompt tell the model to read the header first** — what each
   field means, that `WARNING` means re-query and `NOTE` means use the result — and nothing
   else about the detector. Guidance explains the header; it does not substitute for it.

## Evidence

**Replay before shipping**, over the recorded benchmark tool calls and the 349 live per-key
verdicts from the ADR-0045 research. Rule: non-empty scope and `count` equal to the
endpoint's unfiltered total.

| run | scoped calls | flagged | ignored key | dropped null | honoured filter flagged |
|---|---|---|---|---|---|
| 2026-10-18 (counts from recorded output) | 260 | 14 | 10 | 4 | **0** |
| 2026-10-19 (counts from recorded output) | 273 | 11 | 5 | 6 | **0** |
| all four runs, simulated from live verdicts | 1,073 | 53 | 34 | 14 | 0 (7 coincidences) |

Every flagged call on the two fully recorded runs was a key NetBox ignores (`vlan_group_id`,
`circuit_type_id`, `cable_id`, `contact_role_id`, `assigned_object_*`, `device_id` on
cable-terminations, `user_id` / `user` on change-logs) or a `<relation>_id: null` dropped on
the wire. The seven coincidences across all runs are two shapes — `user_id=9` on change-logs
(3,543 of 3,543) and `object_id=10` on contact-assignments (3 of 3) — and the probe resolves
both to NOTE.

**Live through the real MCP server and NetBox 4.3.3**: `{has_primary_ip: "no"}` → 141 / 100 /
`complete=false`, INCOMPLETE offset=100, WARNING naming `has_primary_ip`; `{device_type_id: 8}`
on outlet templates → 8 / 8, NOTE (probe answers 400); a 25-device `device_id` list on outlets
→ 200 / 1, INCOMPLETE, NOTE (list probe answers 400); `{tenant_id: null}` → sent as `"null"`,
15 / 15, no warning; `{devicerole_id: 5}` → `TOOL_VALIDATION_ERROR` as before, no header.

**Session ladder, twice**, same 11 questions down one thread as every previous ladder:

| ladder | server | secs | tool calls | correct of 10 |
|---|---|---|---|---|
| 2026-10-20 (ADR-0045, tool-result fix) | llama-server | 840 | 17 | 10 |
| 2026-11-09 envelope, count-only rule | Strata | 827 | 39 | **7** |
| 2026-11-09 envelope, probe rule | Strata | 837 | 24 | 10 |

The count-only ladder is the reason for decision 3. Its two warnings were both coincidences
(the 8 outlet templates; the 200 outlets of 25 PDUs), and they landed on a plan that had
already gone wrong on turn 4 — the model resolved "Halvorsen PDU" through the device type
instead of the tenant — so two ambiguous warnings on an already-wrong scope cost three
questions. With the probe, the same questions scored 10 of 10 at the October time; zero
whole-table hits, zero validator rejections.

**Full v5, 90 questions, the envelope the only change** against the 8 November Strata run
(ADR-0046): same server, dataset, evaluators and `gpt-oss:20b` judge.

| | 2026-11-08, no envelope | 2026-11-10, envelope |
|---|---|---|
| correctness · completeness · entity coverage | 0.939 · 0.972 · 0.908 | **0.961** · 0.972 · 0.923 |
| per tier (simple / medium / advanced) | 0.983 / 0.917 / 0.917 | 0.983 / **0.967** / 0.933 |
| agent time, sum of 90 root runs | 8,603 s | **8,060 s** |
| model calls · tool calls | 412 · 400 | **384 · 387** |
| prompt tokens · completion tokens | 5,619,813 · 121,635 | **5,098,446** · 113,167 |
| errors | 0 | 0 |

Five questions changed score: four up — including the two medium null-relation questions,
"hypervisor hosts with no VMs" 0 → 1 and "devices not at Dunder-Mifflin sites" 0.5 → 1 — and
one down, where both runs answered "no, 0 VMs on all 23 clusters" and only the judge differed.

**What the detector actually did in that run.** 304 headers; 42 `INCOMPLETE:` lines; 2
whole-table hits, both resolved by the probe to NOTE, and they were exactly the two
coincidences the replay had predicted; **0 WARNINGs** — the model sent no silently ignored
key in those 90 questions — and 0 ambiguous headers. The headers themselves added 32,061
characters of tool output across the run, on the order of 8,000 tokens, re-read on later
turns of the same question. The run consumed 521,000 fewer prompt tokens.

## Consequences

### Positive
- **The failure the schema layer cannot see is now visible at run time, by name.** A valid key
  with a dropped value produces a WARNING that names the key and says the rows are the
  unfiltered table. The replay shows 25 such calls on the two recorded October runs, every one
  of which had reached the model as a successful result.
- **The header pays for itself in tokens.** About 90 characters per result, and 9 % fewer
  prompt tokens on the full run, because the model stopped re-querying to learn counts it can
  now read. A skill could not have done this: a skill tells the model what to look for; it
  cannot tell it the endpoint's total, and the model doing the probe itself would cost two
  tool round trips and two context re-reads per suspicious result against two cheap HTTP
  requests outside its context.
- **No false alarms where the count-only rule had them.** On the ladder, on the live checks and
  on 90 benchmark questions, every equal-count hit was resolved by the probe to the right
  verdict.
- **Fewer model calls at higher correctness** (412 → 384; 0.939 → 0.961), the first change
  since the `__in` rejection to move both in the right direction on the full set.

### Negative / limitations
- **The WARNING path is proven by replay and live checks, not by the benchmark.** The model did
  not reproduce the dropped-value mistake in the 10 November run, so what it does on seeing a
  WARNING mid-plan is measured only on the live checks and the (negative) count-only ladder.
- **The correctness gain is at the edge of judge noise** (about ±0.02 on 90 questions). The
  token and call reductions are the firmer signal.
- **A probe blind spot**: a key whose honoured-ness depends on the value form
  (`has_primary_ip=true` works, `=no` is dropped) is probed with a string, so a `true` that
  coincidentally matched everything would be reported as ignored. No recorded call has that
  shape.
- **Not seen**: a filter that is honoured but wrong (right key, wrong id), and a dropped key
  beside a honoured one (`prefix` ignored next to `vrf_id` honoured returned 436 of 616, not
  the total). The schema layer and the value rules remain the first line.
- **Totals are cached ten minutes**; an instance edited during a run can drift by a few objects
  and the detector then stays silent. The static benchmark instance does not drift.
- **The remaining two levers are unbuilt**: composite reads for the request sequences the
  ladder repeats (tenant power estate, tenant inventory, change-log statistics) and
  reconciliation of the local MCP server with upstream 1.2.2.

## Relationship to ADR-0041 and ADR-0045

ADR-0041 established the method — verify by result set, not status code — on one filter, by
hand, after the fact. ADR-0045 applied it by construction to filter *names*, from the
instance's own schema, before the call is sent. This ADR applies it to every list result after
the call returns, where the thing being checked is the one the schema cannot carry: whether
NetBox honoured the *value*. The instrument is the same one the ADR-0045 research used to
ground-truth its replay, an impossible value sent against the unfiltered total, moved from the
researcher's notebook into the tool wrapper. The three together — reject the unknown name,
normalise the value form, verify the result against the population — are the full answer to
a server that says 200 when it means "I ignored you", and the last of them is the only one
that can catch what the next NetBox upgrade changes.

## References

- [Phase 2 → MCP Integration — the tool wrapper](../phases/phase-2-netbox/mcp-integration.md)
- [Phase 5 → Overview](../phases/phase-5-production-deepagents/overview.md)
- [ADR-0045](0045-schema-driven-filter-validation-null-wire-tool-results-twice.md) ·
  [ADR-0041](0041-in-lookup-silently-ignored-reject-in-validator.md) ·
  [ADR-0043](0043-filter-grammar-after-in-null-relations-ip-parent-graphql-ids.md) ·
  [ADR-0046](0046-strata-inference-server-same-weights-faster-agent.md)
- Repository: `src/tools/netbox_tools.py` (`UnfilteredTotals`, `list_endpoint`, `scope_filters`,
  `list_payload`, `result_header`, `with_header`, `NetBoxToolWrapper._envelope`);
  `tests/test_filters.py::TestResultEnvelope`; `src/skills/netbox-mcp-filters/SKILL.md`
  ("Read the result header first"); `src/agents/netbox_agent.py`
