# ADR-0041 — Reject `__in`: A Filter That Answers 200 Instead of 400

## Status

**Accepted (October 2026).** Corrects a validator decision made earlier in Phase 5 and the
skill text that went with it. NetBox 4.3 does not reject the `<field>__in` lookup; it drops the
filter and returns the **unfiltered** set with HTTP 200. The agent's filter validator now
rejects `__in` before the request leaves the process, even though the MCP server's own
whitelist and tool description still list it, and the skill names the bare-key list value as
the only multi-value syntax.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`8af278e`, merged `b2a02ac`; decision note with the live verification in
`docs/development/2026-10-06_in-lookup-silently-ignored.md`; harness re-run in
`docs/traces/2026-10-06_netbox-session_in-lookup-fix.md`)
**Bears on**: [ADR-0033](0033-reference-grounded-correctness-evaluator.md) (a check that
inspects only the status code is the completeness metric of filters: it certifies a wrong
result as fine), [ADR-0039](0039-web-serving-layer-serialised-turns-server-token-counts.md) and
[ADR-0040](0040-durable-web-chat-memory-sqlite-checkpoints-cancel-rollback.md) (the long,
persisted browser thread that made the cost visible), [ADR-0038](0038-local-frontier-model-viable-cost-not-capability.md)
(the cost is paid in prefill seconds)

## Context

The NetBox MCP filter grammar is the load-bearing domain fact of this programme: the server
accepts bare fields, a short whitelist of lookup suffixes, and no relationship traversal. The
agent mirrors that whitelist in a local `FilterValidator` so the model gets a structured,
recoverable error instead of an opaque 400.

Earlier in Phase 5 that validator kept a *blacklist* that rejected `__in`, `__regex`, `__gt`,
`__gte`, `__lt`, `__lte` and `__iregex`. A trace (`019e63c0`) showed it contradicting the skill
and costing a recovery cycle per attempt; the fix replaced the blacklist with a copy of the MCP
server's `VALID_SUFFIXES`, which lists `in`, and the skill was rewritten to call `__in` the
*canonical* way to fetch several ids, because the server's tool description shows
`{'id__in': [1,2,3]}` as a valid example. The comparison and regex suffixes had indeed been
mis-blacklisted. `in` had not. Nobody checked the result set; only the status code.

Two places in the same repository had it right all along and were not consulted: the system
prompt line *"NEVER use Django ORM lookups (e.g., `__icontains`, `__in`, `__startswith`)"*, and the
filter-recovery middleware, whose `__in` branch already said "pass a Python list as the value
of the bare key".

**How it surfaced.** On the first long browser thread after ADR-0040 made threads durable
(`3528fa85`), turn 3 — *"Across all Halvorsen Logistics PDUs, how many of the available power
outlets are actually in use?"* — spent **283 s in one model call**. The call was slow because the
tool result before it was **52,721 characters**: `netbox_get_objects("dcim.poweroutlet",
filters={"device_id__in": [...12 PDU ids...]})` had returned the first 200 power outlets of
*every* tenant. The model noticed the tenant mismatch, fell back to 12 per-device calls, and
answered correctly (17 of 96); the cost was the problem, not the answer. The September run of
the same question ladder never hit this because the model happened to write
`{"device_id": [...]}` there. Same question, two syntaxes, one of them a silent no-op.

## Decision

1. **`in` is removed from the validator's `VALID_SUFFIXES`.** The validator now deliberately
   diverges from the MCP server's list, and the comment records the live evidence rather than
   the earlier trace. `device_id__in` becomes a `TOOL_VALIDATION_ERROR` before any request is
   made; the recovery middleware's existing list-form strategy then applies.
2. **The rejection message names the fix.** `suggest_alternative("device_id__in")` returns
   *"`__in` is silently ignored by NetBox (it returns the UNFILTERED set, not an error). To match
   several values, pass a list as the value of the bare key: `{'device_id': [v1, v2, ...]}`"*.
3. **The skill has one multi-value form.** The "batching multiple ids" section shows only
   `{"id": [1, 2, 3]}`, carries the verification table below, says never `__in`, and states that
   the upstream tool description's example is wrong. The suffix table drops the `in` row.
4. **The local copy of the MCP server is corrected** (tool description shows the list form as
   valid and `id__in` as invalid; `in` removed from its whitelist; its test updated). The diff is
   kept beside the decision note for an upstream report to `netboxlabs/netbox-mcp-server`. The
   tool description matters because the model reads it on every call.
5. **The gate for this change is the session harness, not the cloud matrix.** The project's
   convention after a skill or validator change is a `deepseek-v4-flash:cloud` matrix re-run;
   that needs cloud quota and hours and was not done. The 11-turn local session, which contains
   the exact question that failed, was run instead and is recorded below.

## Evidence

**Live verification** (NetBox 4.3.3, read-only token, through the MCP tools; `__in` with the
same values on the same object type as the bare-key list):

| Object | `__in` filter | Returned | Bare-key list | Returned (correct) |
|---|---|---|---|---|
| `dcim.poweroutlet` | `device_id__in=[149,150]` | **200** (page 1 of all tenants) | `device_id=[149,150]` | **16** |
| `dcim.device` | `id__in=[...]` | 141 (all devices) | `id=[...]` | 2 |
| `dcim.device` | `site_id__in=[25]` | 141 | `site_id=25` | 30 |
| `dcim.device` | `name__in=[...]` | 141 | `name=[...]` | 2 |
| `ipam.vlan` | `vid__in=[100,200]` | 94 (all VLANs) | `vid=[100,200]` | 26 |

No field class honours `__in`: not primary keys, not relational ids, not strings or integers.
NetBox's filtersets implement multi-value with `MultiValue*Filter` on the bare name, which the
REST API reads as repeated query parameters (`?device_id=149&device_id=150`); the MCP client
produces exactly that from a list value. Django REST Framework ignores unknown query
parameters, hence 200 rather than 400.

**Session harness re-run**, same 11 questions in the same order as the 2026-09-24 `kvvram`
baseline (the comparable one; the `reorder` run used a different order), same model and server
flags:

| | 2026-10-06 (after) | 2026-09-24 (baseline) |
|---|---|---|
| wall, 11 turns | **1,312 s** | 1,552 s |
| tool calls | **20** | 39 |
| the outlets question (turn 5) | **177 s, 1 tool call** | 195 s, 9 tool calls |
| the same question in the browser thread, 2026-10-05 | 283 s model call + 12 follow-up calls | — |
| `__in` attempts / validation errors / API errors | 0 / 0 / 0 | — |
| context, decode | 9,353 → 55,426 tokens; 14.9 → 11.1 t/s | 9,280 → 46,376; 14.8 → 12.6 t/s |
| scored correctness, turns 1–10 | 1, 1, 1, 0, 0.5, 0.5, 1, 0, 1, 1 | 1, 1, 1, 0, 0.5, 0.5, 1, 0.5, 1, 1 |

The model used the list form four times unprompted (`device_id: [131..136]` twice,
`device_id: [119..130]`, `site_id: [21, 22, 23, 24]`), each returning the correctly filtered set.
The outlets question is now a single list-form `dcim.poweroutlet` query with
`fields=[id, name, device, cable]`, and its per-PDU breakdown (8 / 8 / 1 in use, nine PDUs
empty) matches the reference.

**What did not change, and why.** The scores are the same as the baseline. Turns 4, 5, 6 and
8 lose points because the model scoped "Halvorsen Logistics PDUs" to **site HVL-SEA-DC1 (6
PDUs, 48 outlets)** instead of the **tenant (12 PDUs, 96 outlets)**: "6 PDUs", "17 of 48",
"all 6 PDUs uncabled". That is the site-first contamination the session harness documented on
2026-09-25, inherited from the three site-anchored opening questions, and the `reorder` run
which asks the PDU question first scores it 1.0 with all 12 PDUs. The one movement (turn 8,
0.5 → 0.0) is the judge scoring the same kind of answer — substantively right, wrong
population — differently on two days. This change fixes the cost of a wrong filter syntax; it
has nothing to say about a wrong population.

## Consequences

### Positive
- A silent wrong result becomes a structured error the model recovers from in one step. On the
  ladder that contains the failing question: **half the tool calls, 15% less wall time, same
  correctness.**
- The failure mode that produced the 283 s call cannot recur silently: it is caught before the
  request is made.
- The repository's three statements about `__in` (prompt, middleware, validator and skill) agree
  for the first time.
- A measurement principle is now on record beside ADR-0033's: **a validator that mirrors the
  server's whitelist and checks only the status code cannot see a filter the server ignores.**
  Verify filters by the result set, not by the absence of an error.

### Negative / limitations
- **The validator is now stricter than the server it mirrors**, and that divergence has to be
  maintained by hand; the repo's "keep in sync" rule became "the server's list minus `in`".
- **Verified on NetBox 4.3.3 only.** A later NetBox could add `__in` lookups; the verification
  should be re-run on upgrade.
- **The cloud-matrix gate was not re-run.** The single-question numbers (ADR-0037, ADR-0038)
  have not been re-baselined against the stricter validator. The session harness is one run.
- **The upstream MCP server still documents `id__in` as valid** until the report is filed; any
  client that trusts the tool description will repeat the mistake.
- The earlier decision was reasonable on its evidence — it read the server's code — and was
  wrong for a year. Nothing in the harness could have caught it: a 200 with the wrong rows
  scores as a slow turn, not a failure.

## Relationship to ADR-0033 and ADR-0039

ADR-0033 showed that a completeness metric certifies a fabricated answer as complete and that
only a reference-grounded check sees the error. This is the same shape one layer down: a
status-code check certified an unfiltered result as a successful query for a year, and only a
result-set check saw it. ADR-0039's streaming client is what made the cost legible: in the CLI
or the harness, a 283 s call is a slow turn; in the browser it is a 283 s silence with a
52,721-character tool result sitting above it.

## References

- [Phase 5 → Web Chat — Serving the Local Agent](../phases/phase-5-production-deepagents/web-chat.md)
- [Phase 5 → Local Frontier Inference — the session harness](../phases/phase-5-production-deepagents/local-frontier-inference.md#the-session-harness-one-wrong-scope-four-wrong-answers)
- [Phase 2 → MCP Integration](../phases/phase-2-netbox/mcp-integration.md) (its two-step example
  corrected on the same day)
- [ADR-0033 — Reference-Grounded Correctness Evaluator](0033-reference-grounded-correctness-evaluator.md)
- Repository: `src/tools/netbox_tools.py` (`VALID_SUFFIXES`, `FilterValidator.suggest_alternative`),
  `src/skills/netbox-mcp-filters/SKILL.md`, `src/middleware/filter_recovery.py` (the `__in`
  branch that was right all along), `docs/development/2026-10-06_netbox-mcp-server-in-suffix.patch`
- Upstream: `netboxlabs/netbox-mcp-server` 1.0.0, `src/netbox_mcp_server/server.py`
  (`VALID_SUFFIXES` and the `netbox_get_objects` description); NetBox `netbox/utilities/filters.py`
  (`MultiValue*Filter`)
