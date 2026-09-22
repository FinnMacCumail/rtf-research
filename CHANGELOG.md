# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Phase 5 correction: `--no-kv-offload` was a false choice — removing it is ~2x on every axis - Sep 2026**: The flag was justified below as "4× context for ~37% of decode speed (11.2 → 7.0 tok/s)". That trade-off is **withdrawn**: the 128k window never needed the offload ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **Why it fits**: this is a HYBRID model — only **12 of 48 layers are full attention** (indices 3, 7, 11 … 47). The other 36 are Gated DeltaNet, whose recurrent state is sized by *sequences*, not tokens, so it does not grow with context. Those 12 use **2 KV heads** (GQA) at 256 key/value length → **24 KiB per token**, so `-c 131072` is **3.00 GiB** of attention KV plus ~0.46 GiB recurrent. Confirmed by VRAM subtraction: resident **14.7 → 17.4 GiB**, ~3.1 GiB for both caches, ~3.3 GiB headroom on 21 GiB, no OOM. Geometry read from the GGUF; `kv_unified=true` means the 4 slots share one pool rather than each claiming `-c`
  - **Measured on three NetBox questions in one accumulating thread**: prefill **89.86 → 126.42 tok/s** (1.41×, matched ~8.7k prompts); decode **5.1–7.5 → ~10.8 tok/s** (~2×); Q2 wall **171.2 → 91.6 s** (1.87×); Q3 wall **665.3 → 329.4 s** (2.02×). Correctness unchanged, zero truncations
  - **Q1 shows a 1.24× regression and is excluded**: that turn paid a full 8,743-token prefill on a cold cache after restart — 81.8 s in a single call — against a baseline whose server was already warm. Comparing small prefills against large ones also produced a false "prefill got slower" reading mid-analysis; the matched-prompt pair is the only fair comparison
  - **The flag was CORRECT WHEN SET.** The failure it addressed was `-c 32768` losing 3 of 12 questions, one silently. Raising the window fixed that, and nobody re-checked whether the offload still earned its cost. *A workaround outlives the problem it was built for unless something forces the re-check*
  - **What is NOT withdrawn**: the **≥128k** requirement still stands — only the mechanism changed. And every runtime figure recorded below (775 s/question, ~19.4 h, ~15.8 h, 82 → 155 min) was measured *with KV offloaded*; they are now **lower bounds on speed**. The 12-question benchmark has not been re-run, so no corrected extrapolation is quoted rather than a modelled one
  - **Docs**: ADR-0038's Correction block and Phase 5 → Local Frontier Inference both carry the withdrawal struck in place with the original reasoning preserved; `scripts/serve_qwen4exp.sh` drops the flag and records why (`36c8486`)
- **Phase 5: the 3.8x prefill gain is 1.23x end-to-end — measured, and my projection was 20% optimistic - Sep 2026**: A second full 12-question run with `-b 2048 -ub 2048` replaced the modelled estimate recorded below ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **155.1 → 126.3 min (1.23×)** at *identical* correctness — **11/12 both runs, zero context failures both runs** — and 77 tool calls against 88. The flag is a free speed-up; it changes throughput, not behaviour
  - **The projection said ~105 min / ~12.6 h and was optimistic by ~20%.** It applied the isolated 3.8× prefill gain to *all* 187,272 prompt tokens, when **~85% were already served from llama-server's prefix cache**. Only the genuinely-new remainder and each question's cold first turn are accelerated. Measured full-v5 figures are now **19.4 h → ~15.8 h**
  - **Decode is now the bottleneck and this flag does not touch it**: roughly two thirds of the remaining 126 min is the model writing answers at ~4–5 tok/s. That is the limit no currently-available setting moves
  - **Baseline restated at ~19.4 h** (was ~18.6 h): same run, exact recomputation from per-question times rather than a pre-rounded rate. Not a superseded claim — sharper arithmetic
  - Two questions swapped which one scored wrong (point-to-point circuits ok→wrong, changelog-records wrong→ok) with totals unchanged at 11/1. That is n=1-per-question variance, and the changelog-records item has now flipped in *both* comparisons — an unstable question, not a regression
- **Phase 5: local inference tuning — one flag worth 3.8x prefill, one that actively hurts - Sep 2026**: Four configurations measured on a 9,093-token agent-shaped prompt, one server at a time, baseline re-measured in the same session ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **`-b 2048 -ub 2048` is worth 3.8x on prefill** (34.0 → 129.7 tok/s) and **2.8x on wall clock** (318.7 → 113.7 s). llama.cpp only copies CPU-resident expert weights to the GPU once a microbatch is large enough to amortise the PCIe transfer; the default `-ub 512` never reaches that threshold, so every microbatch pays full freight. With 112 GB of experts in system RAM this single threshold dominates prefill — and prefill is where an agent lives: **187,272 prompt tokens against 21,825 generated** across 12 questions
  - **`--spec-type ngram-simple` is actively harmful**: **163 drafts proposed, ZERO accepted**, decode 5.0 → 3.8 tok/s (−24%) because the model pays verification for drafts it always rejects. Combined with `-ub 2048` it degrades *both* axes (prefill 129.7 → 101.4, decode 5.9 → 3.7). It needs an exactly-repeating 12-token run to draft the next 48, and a table of *distinct* device names has almost none — a 0% acceptance rate is mechanical, not a tuning failure. Recorded in the serve script header so the next reader does not repeat the experiment
  - **Two negatives worth as much as the win.** Prompt caching was *already* doing its job — median LCP similarity **0.956**, 75 of 76 requests warm, only 2,432 of a mean 16,405-token context actually prefilled (**~85% from cache**); the residual is genuinely new tool-result text. And **MoE CPU decode achieves only ~19% of memory bandwidth** (19–20 GB/s against a 101.5 GB/s STREAM triad) with no remedy available: `GGML_NUMA_STRATEGY_MIRROR` exists in ggml's enum *and nowhere else in the source*, while KTransformers needs Ampere+ and AMX this host lacks, and vLLM/SGLang cannot host 112 GB on 21 GiB of VRAM
  - **Modelled** effect on the 12-question run: prefill 68 → 18 min, total ~155 → ~105 min, full v5 18.6 h → **~12.6 h**. Flagged as *modelled, not measured* — the projection accounts for ~100% of observed wall time, which is too neat to be a validated model
  - **Docs**: ADR-0038 gains the prefill lever under *Positive* and three new *Negative* entries; Phase 5 → Local Frontier Inference gains a Tuning section with the four-config table
- **Phase 5 correction: the local constraint was the CONTEXT WINDOW, and 32k understated it - Sep 2026**: A 12-question stratified run (4 per tier, 6/6 across both data islands, full evaluator set) superseded the n=1-per-tier proof recorded below ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **8/12 → 11/12 correct**, and **3 → 0 questions lost to context**, by changing one flag: `-c 131072 --no-kv-offload`. At `-c 32768` the run suffered 2 hard overflows and 1 *silent* truncation; at 128k, none, with peak context reaching **44,295** tokens
  - All three context-lost questions recovered **and scored correct** — point-to-point circuits (15 calls), branch-site firewalls (13 calls), changelog deletions (6 calls); a fourth flipped wrong → correct. The single regression (a changelog count, correct → wrong on *fewer* calls, 4 → 2) is the model under-searching at n=1, not a context effect
  - **The fix uses the resource the box has in surplus**: `--no-kv-offload` moves the KV cache into system RAM — **376 GB against 21 GB of VRAM** — buying 4× context for ~37% of decode speed (11.2 → 7.0 tok/s). It loaded in **20 seconds**
  - **Cost went UP because the fix works**: 60 → 88 tool calls, 82 → 155 min (**1.9×**), and the full-v5 extrapolation from ~10.3 h to **~18.6 h**. A runtime measured on a configuration that silently drops a quarter of its questions is not a runtime worth quoting
  - **A process error recorded, not hidden**: before testing the window, a per-tool-result size cap was built into the MCP wrapper. It cost two ~87-minute runs and was reverted. First it was **inert** — `langchain_mcp_adapters` returns `(content, artifact)` tuples and the guard measured only `str`/`list`, so it skipped every MCP result (3 oversized results, 0 firings); the unit test fed it a bare string, validating an assumption about the adapter rather than its contract. Then it was **mis-sized** — 60,000 chars ≈ 63% of the working budget, still allowing a truncation while turning a question the baseline answered in **3 calls / 141 s** into a **65-minute retry loop**. *Test the cheap hypothesis first*: raising the window took 20 seconds to verify and made the cap unnecessary
  - **Docs**: ADR-0038 gains a Correction block with the `≥32k` limitation struck in place; Phase 5 → Local Frontier Inference gains the 12-question comparison and the detour written up as a process lesson
- **Phase 5: a frontier-class model runs locally — the constraint is cost, not capability - Sep 2026**: Ran **Qwen3.8-Flash-Next** (`qwen4_exp`; 125B MoE with ~6B active per token plus a **51B n-gram lookup table** — a table, not a network — ~176B total) against the real NetBox agent on 2× RTX 2080 Ti (21.1 GiB VRAM) + 376 GB RAM ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **3/3 correct** on one v5 question per tier, including a two-hop power trace that named the right host and PDU outlet and correctly classified every decoy the reference lists. Every *previous* local candidate had failed the **simple** tier, so this reverses the premise that local models lack the capability for this agent
  - **Cost asymmetry is the finding, not correctness.** All three scored 1.0 entity coverage; simple and medium took **2 tool calls each**, advanced took **20** (167% over budget, 16.6 min). A 10× cost jump precisely at the tier where help is most wanted — direct local evidence for ADR-0037's conclusion that *efficiency, not correctness, is the usable routing signal*
  - **`--numa isolate -t 10` was worth more than a 2362-commit llama.cpp upgrade**: +47% CPU-only and **+56%** hybrid decode (13.71 → 21.38 tok/s, sub-1% run-to-run deviation), lifting effective bandwidth ~19 → ~28 GB/s. llama.cpp defaults `--numa` to *disabled*, so threads straddle both NUMA nodes. The rebuild itself measured flat-to-slightly-worse — its value was disproving a stale-binary hypothesis, not raising the number
  - **Measured**: decode 11.2 tok/s; prefill 64.4 tok/s at 1.5k and ~48 tok/s on ~9k agent prompts. Full v5 extrapolates to **~12 h per run** (~36 h at the 3× replication standard). **≥32k context required** — the advanced item runs to ~28k
  - **Two things recorded because they were wrong**: the first advanced failure was an *operator* context limit (`18181 tokens exceeds 16384`), not a model failure; and the predicted tool-call fragility never appeared across 17 turns — but at the reported ~0.7%/turn rate, 24 tool calls yields **~0.17 expected failures**, so the earlier claim that it *disqualified* the model is **withdrawn, not refuted**
  - **Docs**: ADR-0038 added; Phase 5 → Local Frontier Inference added. **n=1 per tier — an existence proof, not a score**
- **Phase 5 correction: a fourth model overturned the v5 "ceiling" - Sep 2026**: The negative result recorded below was drawn from three closely-matched models and is **wrong as stated** ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **qwen3.5:397b-cloud**, a family distant from both deepseek and kimi, produced the **first statistically significant results the benchmark has generated**: pro − qwen **+0.117 [+0.038, +0.196]** and kimi − qwen **+0.094 [+0.016, +0.173]**, both significant. qwen scored 0.817 overall (simple 0.900 / medium 0.883 / advanced 0.667) against flash 0.906, kimi 0.911, pro 0.933
  - **Saturation is a property of the models tested, not of the questions.** One additional family moved it from **69/90 (77%) to 59/90 (66%)** and the items defeating everything from 3 to 2. Ten questions that carried no information across three similar models became discriminating immediately — `vcpu-aggregation`, `ip-mask-mismatch`, `orphaned-cable`, `prefixes-without-vlan`, `tenant-group-size`, `vms-without-primary-ip` among them
  - **Corrected claim**: the set resolves capability gaps of roughly **9pp and above** on 90 paired items, and cannot resolve the ~3pp separating flash, pro and kimi. That is a sample-size limit behaving correctly — not a ceiling
  - **Process errors recorded, not hidden**: generalising "cannot discriminate" from n=3 models of comparable strength, and predicting that further runs would be *"uninformative by construction"* — a forecast the very next run falsified. ADR-0037 preserves the original reasoning and corrects it in place
  - **Three reference-wording defects** found and fixed separately (`deletion-cascade`, `tenantless-instance-wide`, `cross-estate-power-compare`), each of which had penalised a *correct* answer. Re-scoring the stored runs raised the other three models (flash 0.911, kimi 0.928, pro 0.944), which **widens** qwen's gap rather than explaining it
  - **Docs**: ADR-0037 revised with a visible Correction block; Phase 5 → Stratified Benchmark v5 updated; Research Methods → Benchmarking rewritten on saturation
- **Phase 5 continuation: stratified 90-question benchmark (v5) & the ceiling result - Sep 2026** *(superseded by the correction above)*: Scaled the evaluation dataset from 6 to 90 questions to test the premise behind model-handoff routing — that query difficulty predicts which model should handle a query ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **90 questions stratified on four axes** (tier × answer type × domain × data island), with **difficulty defined by retrieval mechanism, not topic** (simple = one object/one filter; medium = one join or aggregation; advanced = ≥2 hops or absence reasoning). Every tenant appears in every tier (19/19/19 and 11/11/11) so tenant name cannot proxy for difficulty
  - **Data enrichment**: an *additive* tenant seeded beside the demo data (69 devices, 42 prefixes, 31 VLANs, 14 circuits, 28 VMs, plus deliberate defects), leaving the original instance untouched as a second island; ground truth computed live against a pinned snapshot through the agent's own **read-only** token
  - **Negative result, recorded as such (ADR-0037)**: across three model families — deepseek-v4-flash 0.906, kimi-k2.6 0.911, deepseek-v4-pro 0.933 — **every paired CI includes zero**. The diagnostic statistic is saturation: **69 of 90 items are solved by all three models and only 3 defeat all three**. The set measures difficulty and efficiency; it does **not** discriminate capability among competent models, and further model runs against it are uninformative by construction
  - **Efficiency is the signal that survives the ceiling**: pro 4.93 tool calls vs flash 6.27 (76/90 vs 66/90 within per-tier budget) — for routing, cost per query discriminates where correctness cannot
  - **Authoring-rule discovery**: an audit found **four of the six v4 examples carried entities their own reference answer could not match** (one self-scored 0.500), so a *perfect* answer could not score 1.0 — some earlier entity-coverage figures were depressed by an authoring bug, not model behaviour. Also: absence items cannot carry entities at all (no stable phrasing for "none"), and `entity_coverage` is **confounded with verbosity** (the most correct model scored lowest while writing 71% the length)
  - **Harness fixes**: the dataset loader was *create-only*, so editing the source had no effect once the dataset existed → true add/update/delete sync; the results reader could return before the primary correctness score landed; per-tier tool-call budgets calibrated from measured percentiles
  - **Docs**: Phase 5 page (Stratified Benchmark v5 & the Ceiling); Research Methods → Benchmarking extended (scale, stratification, saturation, authoring rules, coverage-vs-verbosity); ADR-0037
- **Phase 5 continuation: 0.7.5 upgrade, GraphQL confirmed & merged, ecosystem appraisal - Aug/Sep 2026**: Carried the GraphQL read path from single-run hypothesis to merged mainline feature ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **DeepAgents 0.6.10 → 0.7.5 upgrade** (eval-gated): crossed the 0.7.0 major; the one breaking change (planning now opt-in → strict `excluded_middleware` error) forced a **Workaround B reconciliation** — the TodoList exclusion is now inherent, so the workaround shrank to one line. Regression-neutral; no correctness regression vs the 0.6.10 baseline (ADR-0035)
  - **GraphQL A/B re-measured under 0.7.5**: the earlier ~2× tool-call cost **vanished** (leaner 0.7 prompts) — the GraphQL win became cost-neutral (ADR-0034 updated)
  - **Over-routing fixed via the anchor-object rule**: simple single-object lookups were mis-routed to GraphQL ("count the anchor objects, not the models the answer touches — one named object → MCP, even across models"); device-detail recovered and is now MCP-routed (trajectory-verified)
  - **3× replication**: stabilized combined correctness ≈0.82 (MCP-only 0.667 → GraphQL 0.75 → GraphQL+tightened ≈0.82); cross-domain queries route to GraphQL every run
  - **Merged to mainline (PR #1)** — read-only GraphQL tool + routing skill now on `master`
  - **LangChain-ecosystem appraisal**: mapped the Cloud-only NetBox Platform MCP Server (~100 tools, Code Mode, dynamic discovery) against the current LangChain/DeepAgents/LangSmith stack — most query-processing value is reproducible self-hosted; Code Mode re-deferred (reconfirms ADR-0032); hosted products skipped under the privacy mandate (ADR-0036)
  - **Docs**: Phase 5 pages (DeepAgents 0.7.5 Upgrade, LangChain Ecosystem vs Cloud Platform MCP); GraphQL Read Path + ADR-0034 revised to the confirmed state; Research Methods → Benchmarking extended (framework-upgrade gating + replication methodology); ADRs 0035–0036
- **Phase 5 continuation: Evaluation Correctness & GraphQL Read Path - July 2026**: Hardening the evaluation harness and closing the cross-domain read gap ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **Reference-grounded correctness evaluator**: the completeness judge never read the `reference_answer`, so a confident hallucination ("~7.7% IP utilization" where the truth was 0%) scored 0.9. A new `correctness_judge` fact-checks against the reference and flags contradictions; re-scoring the 9-model June sweep caught **9 hallucinations completeness had rated complete** and reordered the leaderboard (deepseek-v4-pro, penalised by completeness on negative findings, is the more *correct* model) — corrected `netbox-benchmark-v4` dataset (ADR-0033)
  - **Read-only GraphQL complementary read path**: appraised NetBox Labs' Cloud-only "agent-native" Platform MCP Server (~100 tools, Code Mode) and reproduced its cross-domain *reads* on community NetBox — one read-only `netbox_graphql` tool + `netbox_graphql_schema` introspection, built PRP-first in two phases (mechanism, then routing+eval). Read-only enforced by `graphql-core` AST inspection; generalizes to any NetBox type via runtime schema discovery, not a fixed schema (ADR-0034)
  - **Measured A/B**: GraphQL vs MCP-only on `v4` — the site-comparison IP-allocation hallucination went **0.5 → 1.0 correctness on both production models**; deepseek-v4-pro overall 0.65 → 0.883; cost ~2× tool calls, with a soft-routing over-selection side-effect on one simple query. Verdict: complementary path, not the default
  - **Docs**: Phase 5 pages (Evaluating for Correctness, GraphQL Read Path); Research Methods → Benchmarking extended with the completeness-vs-correctness distinction; ADRs 0033–0034
- **Phase 5: Production DeepAgents - June 2026**: Successor to the Phase 4 deepagents build, on packaged DeepAgents 0.6.10 ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents))
  - **Dual native backend**: local Ollama / llama.cpp + Ollama Cloud frontier models, no proxy layer — resolves ADR-0027's local-model failure (ADR-0028)
  - **Model-matrix evaluation harness** (LangSmith): 10-model cloud sweep on `netbox-benchmark-v2`; `deepseek-v4-flash:cloud` matches Claude-class quality at ~36% lower latency; small local models remain too weak for the hardest queries (ADR-0030)
  - **Observability**: first LangSmith adoption in the programme; trace-driven diagnosis caught a 0.6 default-middleware answer-overwrite regression invisible at the answer level (ADR-0029, ADR-0031)
  - **QuickJS / PTC investigation**: deferred with explicit re-trigger conditions after three verification spikes (no benefit on a single-source sequential workload) (ADR-0032)
  - **Docs**: Phase 5 section (overview, multi-model-evaluation, observability-and-monitoring, lessons-learned); Research Methods → Benchmarking + Observability; ADRs 0028–0032
  - **Old repo superseded**: [deepagents](https://github.com/FinnMacCumail/deepagents) marked superseded; design rationale harvested into the successor's `docs/lineage/`
  - **Navigation**: added the 11 previously-unlinked ADRs (0003–0012, 0027) to the docs nav; renumbered planned milestones to Phase 6–8
- **Phase 4 Model Selection Enhancement - December 2025**: Intelligent routing and explicit model control for Claude SDK implementation
  - **Discovery**: Documented Claude SDK's intelligent multi-model routing (Haiku for tools, Sonnet/Opus for responses)
  - **Cost Optimization**: 70-80% cost reduction through automatic model selection when `model=None`
  - **User Control**: Implemented explicit model selection (Auto, Haiku 4.5, Sonnet 4.5, Opus 4) with WebSocket-based switching
  - **Frontend**: `ModelSelector.vue` component with modal interface, current model indicator, and localStorage persistence
  - **API**: `/models` endpoint for model discovery, WebSocket protocol for runtime model changes
  - **Failed Multi-Provider Attempt**: Documented Ollama/LiteLLM integration attempt and reversion
    - Problems: Tool results not displayed (Qwen 2.5:14b), SDK feature loss (MCP, streaming, caching, permissions)
    - Architecture complexity: Dual-path system with separate agents, health checks, Docker Compose orchestration
    - Framework incompatibility: SDK assumes direct Anthropic API; proxy translation breaks native protocol
  - **Research Contribution**: Documents that SDK features depend on architecture assumptions; proxy layers break managed SDK benefits
  - **Repository**: https://github.com/FinnMacCumail/claude-agentic-netbox
- **Research Documentation Accuracy Update - August 2025**: Comprehensive correction of research findings following systematic evaluation
  - Documentation cleanup to accurately reflect Phase 3 OpenAI orchestration failure (0% success rate)
  - Corrected ADRs and research findings to represent actual implementation results vs theoretical claims
  - Preserved accurate record of 93.8% individual NetBox MCP tool success rate for reliable baseline
- Professional meta portfolio documentation structure
- Enhanced research methodology documentation  
- Reproducible demo scenarios and evaluation approaches
- Updated repository URLs and professional presentation elements
- Advanced diagnostic methodologies for complex pipeline debugging
- Execution tracing patterns using monkey patching for root cause analysis
- **Financial Constraint System**: Comprehensive revenue-based query processing for TMDB Phase 1
  - Strategic parameter mapping for revenue thresholds with operator-aware sorting strategies
  - Dual-mode query processing (constraint-based vs fact-based revenue queries) 
  - Progressive parameter injection pipeline with financial constraint specialization
  - Individual movie detail fetching for complete revenue data before threshold filtering
  - 95% accuracy rate for revenue constraint queries, 100% success for revenue fact queries
- **Timeline Query Processing System**: Complete timeline query functionality for person-based temporal queries
  - Multi-layer pipeline debugging methodology for systematic root cause identification
  - Endpoint-aware constraint validation with selective bypass logic for person credit summaries
  - Temporal sorting preservation through constraint filtering with response format coordination
  - Comprehensive test coverage for single-person, multi-constraint, and edge case scenarios
  - 100% success rate for timeline queries: "First movies by Steven Spielberg" returns 332 chronologically sorted entries
- **Intent-Aware Sorting System**: Intelligent query analysis with automatic sort parameter injection
  - Hierarchical intent detection for temporal, quality, and popularity sorting preferences
  - Comprehensive keyword coverage for natural language intent understanding
  - Pipeline integration with automatic parameter override for detected user intents
  - Support for temporal queries ("Latest A24 movies"), quality queries ("Best rated horror films"), and contextual defaults
  - Seamless coordination with timeline query processing and constraint validation systems
- **Phase 3 OpenAI Orchestration FAILED**: Complete multi-agent orchestration system failure with 0% success rate
  - **Failure Evidence**: Comprehensive testing revealed 0% success rate (0/16 test queries) despite extensive development effort
  - **Technical Approach Attempted**: 5-agent system (Conversation Manager, Intent Recognition, Response Generation, Task Planning, Tool Coordination) with LangGraph StateGraph implementation
  - **Root Causes**: Excessive orchestration complexity, tool integration failures, communication overhead, state management problems
  - **Individual Tools Performance**: NetBox MCP tools achieved 93.8% success rate when used directly (15/16 queries)
  - **Lessons Learned**: Complex orchestration can reduce rather than enhance system reliability; working individual tools more valuable than failed coordination
  - **Documented Evidence**: ADR-0013 Multi-Agent Orchestration Architecture documents complete system failure analysis
  - **Git Milestone Tags**: `phase3-week1-4-complete`, `phase3-week5-8-complete` (marking failed attempts)
- **Phase 4 Deepagents Solution SUCCESSFUL**: Intelligent NetBox tool orchestration achieving goals Phase 3 failed to deliver
  - **Success Metrics**: Successfully replaced Claude CLI with working orchestration system
  - **Deepagents Framework**: LangGraph-based workflow orchestration with context quarantine and virtual file system
  - **Dynamic Tool Discovery**: Automatic wrapper generation for all NetBox MCP tools
  - **Intelligent Caching**: Sophisticated prompt caching with configurable TTL and performance monitoring
  - **Cache Performance Tracking**: Granular insights into caching effectiveness and cost optimization
  - **Natural Language Interface**: Conversational NetBox queries replacing CLI commands
  - **Repository**: https://github.com/FinnMacCumail/deepagents
- **Repository Reorganization - September 2025**: Major phase renumbering and documentation restructuring
  - **Phase Renumbering**: Converted from "future development" to sequential milestone tracking
    - Phase 1A (Deepagents) → Phase 4 (completed milestone)
    - Future Phase 2 (Neo4j) → Phase 5 (upcoming milestone)
    - Future Phase 3 (RAG) → Phase 6 (upcoming milestone)
    - Future Phase 4 (Analytics) → Phase 7 (upcoming milestone)
  - **Directory Restructuring**:
    - Moved `/docs/future-development/` content to `/docs/milestones/` and `/docs/phases/phase-4-deepagents/`
    - Renamed `/docs/phases/phase-3-openai/` to `/docs/phases/phase-3-openai-failed/` to indicate failure
    - Updated all documentation references to reflect new phase numbering
  - **Professional Documentation**: Converted from theoretical "future development" to accurate milestone tracking with clear success/failure indicators

### Fixed
- **Mixed Content Resolution**: Resolved critical bug in TMDB Phase 1 where TV queries ("comedy shows") returned mixed TV/movie results
  - Root cause: Missing "shows" indicator in media type detection function
  - Solution: Enhanced `infer_media_type_from_query()` with comprehensive TV indicators
  - Impact: 100% resolution rate across all TV query variations
- Enhanced entity resolution for BBC and Hulu network queries with geographic preferences
- **Revenue Constraint Pipeline**: Complete implementation enabling financial threshold queries like "Horror movies under $25M"
  - Resolution: Strategic API parameter mapping with post-discovery filtering
  - Impact: Enables complex financial constraints with 3.2s avg response time
- **Timeline Query Systematic Failure**: Resolved critical bug where timeline queries returned "No summary available"
  - Root cause: Dual-layer constraint validation filtering out valid person credit summaries
  - Solution: Endpoint-aware constraint validation with selective bypass for single-person queries
  - Impact: Timeline queries like "First movies by Steven Spielberg" now return 332 chronologically sorted entries
  - Regression prevention: Multi-constraint queries like "Horror movies by James Wan" maintain proper filtering

### Architecture Decision Records
- **ADR-0003**: Diagnostic-First Debugging Methodology - systematic execution tracing for complex pipeline failures
- **ADR-0004**: Media Type Detection Enhancement Strategy - comprehensive natural language indicator coverage  
- **ADR-0005**: Entity Resolution Cache Override Pattern - geographic preference handling for ambiguous entities
- **ADR-0006**: Financial Constraint Parameter Mapping Strategy - strategic revenue threshold processing with API optimization
- **ADR-0007**: Dual-Mode Financial Query Processing - constraint-based vs fact-based revenue query routing
- **ADR-0008**: Progressive Parameter Injection Pipeline - systematic multi-phase parameter building with conflict resolution
- **ADR-0009**: Endpoint-Aware Constraint Validation - selective bypass logic for person credit summaries while preserving multi-constraint validation
- **ADR-0010**: Multi-Layer Pipeline Debugging Methodology - progressive isolation techniques for complex pipeline failure analysis
- **ADR-0011**: Temporal Sorting and Constraint Interaction Pattern - chronological ordering preservation through constraint validation layers
- **ADR-0012**: Intent-Aware Sorting Strategy - hierarchical intent detection with automatic sort parameter injection for temporal and quality queries
- **ADR-0013**: Multi-Agent Orchestration Architecture ⚠️ FAILED - Documents complete system failure analysis and lessons learned
- **ADR-0014**: OpenAI Model Selection Strategy - GPT-4o for conversation management, GPT-4o-mini for specialized agents (superseded by failure)
- **ADR-0015**: CLI Testing Infrastructure Design - Interactive testing approach (revealed orchestration failures)
- **ADR-0016**: Agent Communication Protocol - Correlation ID system design (non-functional due to orchestration failure)
- **ADR-0017**: Session Management Strategy - Conversation state tracking approach (superseded by deepagents solution)
- **ADR-0018**: LangGraph StateGraph Architecture ⚠️ FAILED - 5-node workflow system failure documentation
- **ADR-0019**: Limitation Handling Strategy - Progressive disclosure approach (superseded by deepagents framework)
- **ADR-0020**: Intelligent Caching Redis Strategy - Tool-specific TTL configuration (concepts applied in deepagents solution)
- **ADR-0021**: Phase 4 Framework Comparison Study - Deepagents vs Claude SDK comparative implementation
- **ADR-0022**: Project Requirements Package Framework (Deepagents) - pyproject.toml architecture for LangChain-based implementation
- **ADR-0026**: Claude SDK Project Requirements Package - pyproject.toml architecture for managed SDK implementation
- **ADR-0027**: Intelligent Routing and Model Selection Strategy - Documents intelligent routing discovery, model selection implementation, and Ollama/LiteLLM reversion

## [1.0.0] - 2025-08-10

### Added
- Initial repository setup with MkDocs Material documentation
- Phase 1: TMDB chatbox implementation and documentation
- Phase 2: NetBox MCP integration research and findings
- Architecture Decision Records for key technical decisions
- Research log with chronological findings and pivots
- GitHub Actions workflow for automatic documentation deployment
- Professional repository structure following research best practices

### Research Milestones
- **Phase 1 ✅ Complete**: TMDB chatbox with semantic+symbolic retrieval
- **Phase 2 ✅ Complete**: NetBoxLabs official MCP server with 3 generic tools and field filtering
- **Phase 3 ❌ Failed**: OpenAI orchestration attempt with 0% success rate - documented failure analysis
- **Phase 4 ✅ Complete**: Agent framework comparison study (Deepagents vs Claude SDK)
- **Performance Analysis**: Identified N+1 query optimization opportunities (VLAN queries: 127 calls → optimized batching)
- **Architectural Foundation**: Established reliable foundation for upcoming milestones (Phase 5-7)

### Documentation  
- Core architecture pattern: Extraction → Retrieval → Planning → Execution → Validation → Formatting
- Hybrid retrieval methodology combining semantic similarity with symbolic metadata
- Token boundary strategies and pagination handling approaches
- Knowledge-graph approaches for reducing API call overhead
