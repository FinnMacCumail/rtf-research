# ADR-0045 — Schema-Driven Filter Validation, and Two Bugs the Benchmarks Never Saw

## Status

**Accepted (October 2026).** Replaces the hand-written list of silently-ignored filter forms
(ADR-0041, ADR-0043) with layers that check filter names and enum values against the NetBox
instance's own OpenAPI document, and GraphQL documents against its introspected schema, with
the hand rules retained for the value semantics the schema cannot express. The research that
decided this also found two defects that had been invisible to every benchmark: JSON `null`
filter values never reached NetBox, and every MCP tool result reached the model twice.
**Repository**: [https://github.com/FinnMacCumail/ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)
(`ef320bf` schema layers, GraphQL validate, null wire fix; `1587629` tool-result fix; research
report `docs/development/2026-10-20_openapi-schema-driven-filter-validation.md` with four notes
and the prototype under `2026-10-20_openapi-validation-notes/`; implementation note with every
gate `docs/development/2026-10-20_schema-validation-implemented.md`; traces
`docs/traces/2026-10-20_netbox-session_schema-validation.*` and `…_tuple-fix.*`)
**Bears on**: [ADR-0043](0043-filter-grammar-after-in-null-relations-ip-parent-graphql-ids.md)
(whose `_id: null` decision was right about NetBox and wrong about the wire),
[ADR-0041](0041-in-lookup-silently-ignored-reject-in-validator.md) (the method, now applied by
construction), [ADR-0044](0044-search-defaults-tenants-providers-parent-counts.md),
[ADR-0039](0039-web-serving-layer-serialised-turns-server-token-counts.md) (the per-call token
ledger that exposed the doubled payload), [ADR-0034](0034-read-only-graphql-complementary-read-path.md)

## Context

Three days of hand-written rules had produced five families of rejected forms and still
missed the one that cost a retest 138 seconds: `device_role_id`, a NetBox 3 name that 4.x
silently drops. The question was whether NetBox's OpenAPI document — 9.3 MB, 292 query
parameters on the device list alone — could be the authority for "does this filter exist
here" instead of a list someone remembers.

Four researchers answered it against the local 4.3.3 instance: an audit of the schema against
every known form, a prototype validator replayed over 1,237 recorded benchmark calls with 349
live ground-truth checks, a survey of prior art, and a study of GraphQL introspection. The
prior-art finding that frames everything: django-filter's DRF backend validates only *known*
filters, DRF ignores unknown query parameters, NetBox has no strict mode and none is proposed
(the open performance issue #23397 explicitly keeps ignore-unknown behaviour). The only client
that validates names against the schema is pynetbox, `strict_filters` since 7.6.0.

## Decision

1. **REST filter names are checked per endpoint against a vendored index of `/api/schema/`**
   (`scripts/build_netbox_filter_index.py` → `src/tools/netbox_filter_index.json.gz`, 145
   endpoints, 11,520 parameters, 919 with enums, 126 of 126 MCP object types resolved, 27.6 KB,
   1 ms load), stamped with the NetBox version and compared with `/api/status/` at agent
   start. Enum values are checked against the schema's enum tables. `fields`, `brief`, `q`,
   `limit`, `offset`, `ordering`, `export`, `format` and `cf_*` are exempt. No index, unknown
   object type or endpoint → fail-open to the hand rules. No HTTP per call.
2. **The hand rules stay, and run first.** They own value semantics the schema cannot carry:
   `__n` with null returns the complement, `<relation>_id` null form, `vrf` wants a route
   distinguisher, slug values, the change-log verbs. The schema layer owns names.
3. **Suggestions in a fixed order**: alias table, then the longest real parameter the wrong
   key ends with (`device_role_id` → `role_id`, `circuit_type_id` → `type_id`, `vlan_group_id`
   → `group_id`), then `_id` → bare key, then difflib with a guard that never offers `__n` for
   an `__in`/`__empty` input (difflib's own top hit for `site_id__in` is `site_id__n`), then the
   endpoint's relation filters.
4. **GraphQL documents are validated with `graphql.validate`** against the schema the module
   already introspects and caches, plus a custom rule for `null` on any ID-typed input field;
   the hand-written grammar rules are only the fallback when introspection is unavailable.
   graphql-core's messages are byte-identical to NetBox's (its Strawberry layer *is*
   graphql-core), carry "Did you mean", and catch the field-name errors no grammar rule can.
5. **Two bug fixes shipped first**, because they change answers regardless of the rest:
   - *JSON `null` on the wire.* `requests` drops parameters whose value is `None`, and the MCP
     client passes the filters dict straight through, so `{"tenant_id": None}` reached NetBox
     as no filter and returned every device. The wrapper normalises `None` → `"null"` and
     booleans → `"true"`/`"false"`.
   - *Tool results twice.* The wrapper rebuilt each MCP tool without
     `response_format="content_and_artifact"`; the adapter returns `(content, artifact)`, so
     LangChain stringified the pair into the tool message: the JSON once backslash-escaped
     inside a text block, then again as `structured_content`. The wrapper now declares the
     format and returns a pair on every path.
6. **Whitelists corrected**: `regex`/`iregex` removed (silently ignored on 4.3.3; the lookup
   arrives in 4.3.5, netbox#19903); GraphQL `ordering` removed (no `_list` field takes it).

## Evidence

**Replay of 1,237 recorded calls** (four v5 runs; `scripts/replay_filter_validator.py`):

| | count |
|---|---|
| filter keys the index does not know | 54 uses, 27 distinct, on 52 calls |
| … verified silently ignored live | **54 of 54** |
| false positives (live-honoured keys rejected) | **0** |
| change-log `action` 400s caught by the enum layer | 6 of 6 |
| hand rules alone would catch | 25 |
| schema layers alone would catch | 58 |
| combined | 61, 4.9% of calls |

The 27 names are mostly real filters with a model prefix (`device_role_id`, `circuit_type_id`,
`cluster_type_id`, `vlan_group_id`, `contact_role_id`, `cable_id`, `parent_prefix`,
`assigned_object_*`), the `__in` family, `tenant__empty`/`__null`, ip-address `prefix`/`without`,
and three filters that exist on no endpoint. The schema lists no `__in`, no `__isnull`, and
`__empty` only on scalar fields, so the whole ADR-0041 suffix family is caught by name alone.

**What the schema cannot see**, all value-level: `tenant__n=null` (complement), `vrf=<name>`
(400; the schema does say "VRF (RD)"), `site=DM-Scranton` (400; slug), `has_primary_ip=1`
(ignored; only `true`/`false` work). It also mis-types 109 `*_id` parameters as strings and lists
`created__empty` as a date-time filter that returns a 400, so types and nullability are never
enforced.

**The null wire bug, measured.** Through the real MCP tool: `{"tenant_id": None}` → 141 of 141
devices; `"null"` → 15. The 13 and 16 "correct null-form uses" ADR-0043's full run had counted
were unfiltered queries. After the fix: 15.

**The doubled payload, measured.** On the 20 October full run, 361 of 404 tool messages had the
stringified-tuple shape: 4.85M characters in total of which the data was 1.99M — about 2.4× the
necessary tool text prefilled on every call since the wrapper was written. The 68-device query
that reached the model as 63,717 characters reaches it as 27,076 after the fix.

**GraphQL.** Introspection 1.15 MB in 0.6 s, schema build 0.19 s, 1.4 ms per validation; 25
probe cases produce messages identical to NetBox's live responses; 14 of the 16 residual
GraphQL rejections in the 18 October run were name errors `validate` catches locally. Not
caught, and kept as guards: the 10-alias cap (a runtime setting), mutations (the schema has no
mutation type, so `validate` passes them), and `tenant_id: null` (324 of 325 ID inputs are
nullable; hence the custom rule).

**Session ladder**, same 11 questions as every previous run:

| | 2026-10-19 | 2026-10-20 schema layers | 2026-10-20 + tool-result fix |
|---|---|---|---|
| total · calls · correct of 10 | 1,248 s · 16 · 9.5 | 971 s · 15 · **10.0** | **839 s · 17 · 10.0** |
| context at end of session | 90,106 | 70,321 | **40,911** |
| outlets-in-use turn | 168 s | 55 s | 131 s (a larger fetch this run) |
| validator rejections fired | 0 | 0 | 0 |

The last column is the number that compounds: 42% less context after eleven turns of the same
conversation, because the same results now arrive once and unescaped.

## Consequences

### Positive
- **"Does this name exist here" is now answered by the instance, not by memory.** The 27
  wrong names the replay found had produced more silently wrong results in the two latest
  full runs than every suffix mistake combined, and none of them was on the hand list.
- **Zero false positives on 1,237 real calls** and no rejection on two ladders, with the hand
  rules unchanged in effect.
- **Two bugs that no benchmark could have surfaced are fixed.** A 200 with the whole table and
  a correct answer prefilled from twice the text both score as success. The benchmark's own
  ground truth was partly wrong: the null-form calls it counted as correct were unfiltered.
- **The first ladder with a perfect score, then the fastest ladder** (839 s against 1,312 s on
  6 October) at the same correctness.

### Negative / limitations
- **The index is a 4.3.3 artefact.** Lookup sets change between releases (regex at 4.3.5);
  the version warning catches a stale index but someone must re-run the script.
- **Step 6 of the plan — a full v5 run with everything on — has not been run.** The replay
  guarantees the catches; it does not measure whether the model recovers in one turn from the
  new messages. One retest trace shows two rejections each recovered on the next call; one
  trace is not a measurement.
- **Suggestions are right for 20 of 27 recorded wrong names** by the prototype's own
  expectation and arguably better on the other 7; the model's reaction to a "no such filter;
  its relation filters are …" message is unmeasured.
- **Custom-field filters are invisible to the schema** (instance-time) and are exempt rather
  than validated.
- **Three upstream defects are documented and not yet filed** (`None` dropped, `action`
  docstring, search defaults), alongside the earlier `__in` one that upstream fixed
  independently in 1.2.0.

## Relationship to ADR-0041 and ADR-0043

ADR-0041 established the method — verify by result set, not status code — on one filter.
ADR-0043 applied it by hand to four more families and projected a saving that did not arrive.
This ADR changes the authority for one question, name existence, from a list to the running
instance's own document, and finds in the process that two of the project's measurements had
been wrong in the same way the filters were: a successful-looking result that was not what it
claimed. The null-form "wins" ADR-0043 reported were not wins; the tool text every run had
prefilled was twice what it needed to be. A validator that rejects loudly, a result count
checked against the endpoint total, and a token ledger that shows what the model actually read
are the three instruments that found all of this; none of them is a benchmark score.

## References

- [Phase 5 → GraphQL Read Path — validated against the schema](../phases/phase-5-production-deepagents/graphql-read-path.md#validated-against-the-schema-october-2026)
- [Phase 2 → MCP Integration — the tool wrapper](../phases/phase-2-netbox/mcp-integration.md)
- [ADR-0043](0043-filter-grammar-after-in-null-relations-ip-parent-graphql-ids.md) ·
  [ADR-0041](0041-in-lookup-silently-ignored-reject-in-validator.md) ·
  [ADR-0044](0044-search-defaults-tenants-providers-parent-counts.md)
- Repository: `src/tools/netbox_tools.py` (`normalise_filter_values`, `validate_against_schema`,
  `_schema_suggestion`, `index_version_warning`, `response_format="content_and_artifact"`),
  `src/tools/netbox_graphql.py` (`NoNullIdFilterRule`, `_validate_against_schema`, `_ensure_schema`),
  `scripts/build_netbox_filter_index.py`, `scripts/replay_filter_validator.py`
- Prior art: pynetbox `strict_filters` (7.6.0, PR #697); netbox-community/netbox #23397
  (keeps ignore-unknown behaviour); netbox #19903 (regex lookups, 4.3.5); drf-spectacular 0.28
  django-filter extension; graphql-core 3.2 `validate`
