# Research Log

## 2026-09-02 — Phase 5 continuation: 0.7.5 upgrade, GraphQL confirmed & merged, ecosystem appraisal ✅
- **DeepAgents 0.6.10 → 0.7.5, behind the eval gate**: crossed the 0.7.0 major. The one breaking change — planning is now opt-in, so `TodoListMiddleware` isn't bundled and 0.7.x strictly errors on an `excluded_middleware` entry that matches nothing — collided with Workaround B. Reconciled: removed the (now-inherent) TodoList exclusion, kept `base_system_prompt=""` as belt-and-suspenders. Regression-neutral; the negative-finding queries Workaround B protected stayed healthy. Recorded in **ADR-0035**. A framework workaround becoming *inherent* in the next version is the healthy sign.
- **GraphQL: from single-run hypothesis to merged feature.** Re-measuring the A/B under 0.7.5 showed the earlier **~2× tool-call cost had vanished** (leaner default prompts) — a measured "cost" that was an artifact of the layer around the model. The one regression (simple single-object lookups over-routed to GraphQL) was fixed by reframing routing around the **anchor-object rule** — *count the anchor objects, not the models the answer touches; one named object → MCP, even across models.* Device-detail recovered and is now MCP-routed (the run that slipped to GraphQL scored 0.5, proving the mechanism). **3×-replicated**: stabilized combined correctness **≈0.82** (MCP-only 0.667 → GraphQL 0.75 → GraphQL+tightened 0.82). **Merged to mainline (PR #1).** ADR-0034 updated to the confirmed state.
- **LangChain-ecosystem appraisal** (five-agent survey): mapped NetBox Labs' Cloud-only Platform MCP Server (~100 tools, Code Mode, dynamic discovery, Skills, self-correction) against the current LangChain/DeepAgents/LangSmith stack. Most query-processing value is reproducible self-hosted; **Code Mode re-deferred** (reconfirms ADR-0032 — sequential single-call workloads don't benefit); hosted products (Managed Deep Agents, LLM Gateway, Context Hub, hosted LangSmith) skipped under the privacy mandate; OSS `openevals`/`agentevals` + OTel give eval/observability parity. Produced the adoption plan (0.7.5 → model-handoff routing → RubricMiddleware → NL→query). Recorded in **ADR-0036**.
- **Methodology reinforced**: the same model-matrix harness now also gates framework upgrades (run the *same* eval before/after), and replication is treated as part of the method — a single-run A/B with an open caveat is a hypothesis; per-question wins that repeat are trustworthy on one run, an aggregate ranking needs ≥3.
- **New docs**: Phase 5 pages (DeepAgents 0.7.5 Upgrade, LangChain Ecosystem vs Cloud Platform MCP); revised GraphQL Read Path + ADR-0034; Benchmarking methods extended; ADRs 0035–0036.

## 2026-07-29 — Phase 5 continuation: Evaluation correctness & GraphQL read path ✅
- **The completeness metric was blind to hallucination**: the Phase 5 harness scored one answer 0.9 on completeness while it hallucinated the exact figure the question asked for — "~7.7% IP utilization / 180 IPs allocated" where the verified truth was **0%** (the demo instance's 180 IPs all sit in an unrelated `172.16.0.0/24` range, none in the sites' `10.112.x` prefixes). Root cause: `completeness_judge` was fed `expected_entities`, never the `reference_answer`, which existed but was read by no evaluator.
- **Reference-grounded correctness judge** (the fix): a new evaluator fed the `reference_answer` that scores *contradiction*, not coverage. Re-scoring the stored 9-model June sweep (judge calls only, no re-runs) surfaced **9 hallucinations completeness had rated complete** and **reordered the leaderboard** — deepseek-v4-pro (0.717 completeness → 0.833 correctness) overtakes flash (0.800 → 0.583); the June "flash ≥ pro" headline was a completeness artefact. Produced the corrected `netbox-benchmark-v4` dataset. Recorded in **ADR-0033**.
- **Recreating the Cloud "agent-native" reads on community NetBox**: appraised NetBox Labs' Cloud-only Platform MCP Server (~100 tools, sandboxed "Code Mode"). Its Knowledge layer (Agent Skills) is open-source; its cross-domain *reads* map to the community GraphQL API; its governance layer is write-safety machinery irrelevant to a read-only agent. Built a **read-only `netbox_graphql` tool** (+ `netbox_graphql_schema` introspection) PRP-first in two phases; read-only enforced by `graphql-core` AST inspection; generalizes to any NetBox type via runtime schema discovery, not a fixed schema. Recorded in **ADR-0034**.
- **Measured A/B, done right**: GraphQL vs MCP-only on `v4` with the correctness judge. The site-comparison IP-allocation trap — which hallucinated a different fabricated % on *every* MCP run — scored **0.5 → 1.0 on both production models**; deepseek-v4-pro overall 0.65 → 0.883. Cost: ~2× tool calls, plus a soft-routing over-selection side-effect on one simple query. Verdict: keep GraphQL as a complementary path, not the default; per-question win robust, aggregate needs ≥3 runs (single-run variance).
- **New docs**: Phase 5 pages (Evaluating for Correctness, GraphQL Read Path); Research Methods → Benchmarking extended (completeness vs correctness); ADRs 0033–0034.

## 2025-01-05 — Kickoff
- Surveyed LLM landscape and developer tools (MCP, Claude Code, OpenAI API).
- Defined research questions around intent extraction, endpoint planning, and evaluation.

## 2025-03-15 — TMDB joins & formatting
- Implemented multi-entity join logic for TMDB.
- Added result validation, reranking, and flexible templates (list, timeline, comparison, fact).

## 2025-04-20 — TMDB Phase 1 completion
- Completed progressive constraint relaxation system with comprehensive logging.
- Validated multi-entity joins across movie/TV domains with role-aware validation.
- Demonstrated semantic + symbolic retrieval for complex queries.

## 2025-05-15 — NetBox MCP research phase
- Evaluated NetBoxLabs official MCP server architecture (3 generic tools: get_objects, get_object_by_id, get_changelogs).
- Identified performance bottlenecks: token overflow on large responses, N+1 query patterns.
- Began integration testing with MCP protocol and field filtering capabilities.

## 2025-06-30 — Performance optimization discoveries  
- Documented critical VLAN query issue: 127 API calls for 63 VLANs (N+1 pattern).
- Researched batch processing approaches and parallel HTTP request strategies.
- Designed knowledge-graph ingestion plan to pre-compute relationships.

## 2025-08-05 — Production integration milestone
- Successfully integrated Claude Code with NetBox MCP server in production environment.
- Validated enterprise-grade safety controls and dry-run capabilities.
- Documented pagination/token constraints and drafted comprehensive four-phase optimization plan.

## 2025-08-10 — Research documentation phase
- Created professional meta portfolio repository following GitHub best practices.
- Established MkDocs Material documentation site with Architecture Decision Records.
- Documented reproducible demo scenarios and evaluation methodologies.

## 2025-08-11 — Financial constraint system completion
- **Revenue Constraint Pipeline Achievement**: Completed comprehensive financial query processing system for TMDB Phase 1
- **Strategic Parameter Mapping**: Implemented operator-aware revenue constraint handling with strategic TMDB API sorting
  - "Under" queries use `popularity.desc` to avoid $0 revenue indie films (71% API efficiency improvement)
  - "Over" queries use `revenue.desc` for optimal high-earner discovery (98% accuracy rate)
- **Dual-Mode Query Architecture**: Built separate processing pipelines optimizing for constraint vs. fact-based revenue queries
  - Constraint queries: 3.2s avg response, 95% success rate for threshold filtering
  - Fact queries: 1.1s avg response, 100% success rate for specific revenue lookup
- **Progressive Parameter Injection**: Established four-phase parameter building system with conflict resolution
  - Entity-based → Constraint-based → Semantic inference → Revenue specialization
  - Systematic precedence hierarchy prevents parameter conflicts and API errors
- **Individual Movie Enrichment**: Implemented post-discovery revenue data fetching to overcome TMDB API limitations
  - `/discover/movie` lacks revenue fields, requires individual `/movie/{id}` calls for complete financial data
  - Optimized with result limiting and strategic sorting to minimize API overhead
- **Performance Validation**: 95% accuracy across 50 test queries, strategic sorting reduces irrelevant API calls by 60%
- **Architecture Documentation**: Created comprehensive ADR suite documenting financial constraint decision patterns
  - ADR-0006: Financial parameter mapping strategies with API optimization analysis
  - ADR-0007: Dual-mode processing architecture with routing accuracy validation  
  - ADR-0008: Progressive parameter injection with systematic conflict resolution
- **Research Impact**: Demonstrates specialized constraint handling for complex domains requiring API strategy beyond generic parameter mapping

## 2025-08-31 — Phase 3 OpenAI Orchestration Failure
- **Orchestration Failure**: Multi-agent OpenAI orchestration system achieved 0% success rate (0/16 test queries)
- **Evidence**: ADR-0013 documents complete system failure vs claimed 100% success
- **Individual Tools Success**: NetBox MCP tools achieved 93.8% success rate when used directly
- **Root Causes**: Excessive orchestration complexity, tool integration issues, communication overhead, state management problems
- **Decision**: Abandoned OpenAI orchestration approach in favor of simpler solutions

## 2025-09-22 — Phase 4 Deepagents Completion ✅
- **Deepagents Implementation Success**: Successfully replaced Claude CLI with intelligent NetBox tool orchestration using the deepagents framework
- **Key Repository**: [Deepagents NetBox Agent](https://github.com/FinnMacCumail/deepagents/blob/master/examples/netbox/netbox_agent.py) provides complete Claude CLI replacement
- **Architecture Achievements**: LangGraph-based workflow orchestration, dynamic tool discovery, intelligent caching with performance monitoring
- **Framework Benefits**: Context quarantine, virtual file system, human-in-the-loop capabilities, sophisticated cache monitoring
- **NetBox Integration**: Automatic wrapper generation for all NetBox MCP tools with conversation-level caching and multi-turn context preservation
- **Performance Impact**: Intelligent prompt caching with configurable TTL, granular cost optimization tracking, natural language infrastructure queries
- **Foundation Established**: Phase 4 provides architectural foundation for subsequent development milestones (Phase 5-7) through proven deepagents orchestration

## 2026-06-22 — Phase 5 Production DeepAgents ✅
- **Successor to the Phase 4 deepagents build**: [ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents) — production NetBox agent on packaged **DeepAgents 0.6.10** (Phase 4 used a vendored 0.0.5-era fork). The old repo is now superseded; its design rationale was harvested into the successor's `docs/lineage/`.
- **Resolved ADR-0027's local-model failure**: added a **dual native backend** (local Ollama / llama.cpp + Ollama Cloud frontier models) with **no proxy layer**. This sidesteps the LiteLLM-vs-SDK incompatibility that defeated the Phase 4 attempt. Recorded in **ADR-0028**.
- **Multi-model evaluation harness** (LangSmith): 10-model cloud sweep on `netbox-benchmark-v2` with three evaluators (entity coverage, LLM-judge completeness, tool-call efficiency). Finding: `deepseek-v4-flash:cloud` matches Claude-class quality at ~36% lower latency than its larger sibling; small local 14–32B models remain too weak for the hardest multi-step queries (corroborates *and* refutes ADR-0027). Recorded in **ADR-0030**.
- **Observability as a first-class practice** (the programme's first LangSmith adoption): trace-driven regression diagnosis caught a DeepAgents 0.6 default-middleware bug that overwrote a correct answer with an "All done…" filler — invisible at the answer level, only diagnosable from the sub-run chronology. Recorded in **ADR-0029** and **ADR-0031**.
- **Negative result, done right**: investigated the 0.6 QuickJS / Programmatic Tool Calling middleware with three verification spikes; deferred adoption (no benefit for a single-source sequential 4-tool workload; +13.7% latency) with explicit re-trigger conditions. Recorded in **ADR-0032**.
- **New docs**: Phase 5 section (overview, multi-model-evaluation, observability-and-monitoring, lessons-learned); Research Methods → Benchmarking and Observability; ADRs 0028–0032. Planned milestones renumbered to Phase 6–8 (Neo4j / RAG / Analytics).
