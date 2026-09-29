# RTF AI Research Portfolio (2025–2026)

## Research Overview: Systematic Anti-Hallucination in Domain-Specific LLM Applications

This repository documents RTF's comprehensive research program addressing the critical challenge of **LLM hallucination in domain-specific applications**. Through systematic development of constraint-based validation, direct tool protocols, orchestration failure analysis, and intelligent caching systems, this research demonstrates how to achieve reliable, factually accurate LLM responses in complex technical domains.

**Documentation site**: https://finnmaccumail.github.io/rtf-research/ · **RTF**: https://rtf.spc.org/

### Core Research Hypothesis
*Multi-stage validation combining symbolic constraints, structured tool protocols, and progressive verification can eliminate fabricated responses while maintaining natural language interaction quality.*

### Research Evolution Overview
The research progressed through five phases between January 2025 and September 2026, building sophisticated anti-hallucination mechanisms through iterative development and failure analysis:

- **Phase 1 – TMDB RAG API Routing + Constraint-Based Validation System**
  - **Repository**: https://github.com/FinnMacCumail/tmdbGPT
  - **Anti-Hallucination Focus**: RAG API endpoint routing, symbolic constraint trees, progressive relaxation, TMDB search API validation
  - **Key Innovation**: Multi-stage RAG endpoint routing + API validation pipeline preventing fabricated movie/TV data

- **Phase 2 – NetBox MCP Server (Shared Infrastructure)**
  - **Repository**: https://github.com/netboxlabs/netbox-mcp-server
  - **Purpose**: Official NetBoxLabs read-only MCP server providing foundational NetBox data access
  - **Core Tools**: 3 generic tools with field filtering (get_objects, get_object_by_id, get_changelogs)
  - **Key Innovation**: Simplified architecture reducing token overhead; shared infrastructure layer for Phase 4 frameworks

- **Phase 3 – OpenAI Multi-Agent Orchestration (Failed)**
  - **Repository**: No working implementation (0% success rate)
  - **Anti-Hallucination Focus**: Attempted multi-agent coordination with complex orchestration
  - **Failure Analysis**: Excessive complexity reduced rather than enhanced system reliability (see ADR-0013)

- **Phase 4 – Agent Framework Comparison: Deepagents vs Claude SDK**
  - **Repositories**:
    - Deepagents: https://github.com/FinnMacCumail/deepagents
    - Claude SDK: https://github.com/FinnMacCumail/claude-agentic-netbox
  - **Research Focus**: Empirical comparison of flexible vs production-ready agent frameworks
  - **Key Innovation**: Validated that framework choice is context-dependent; both approaches successfully build production agents with different trade-offs (flexibility vs convenience)

- **Phase 5 – Production DeepAgents: Multi-Model, Evaluation, Local Inference & Serving**
  - **Repository**: https://github.com/FinnMacCumail/ollamaDeepAgents (supersedes the Phase 4 Deepagents build)
  - **Research Focus**: Taking the Deepagents agent to DeepAgents 0.7.5 with a dual local/cloud model backend, a LangSmith model-matrix evaluation harness, trace-driven observability, a locally served 176B model, and a web chat over it built without changing what the harnesses measure
  - **Key Innovation**: Resolved ADR-0027's local-model failure via a native (no-proxy) backend; a frontier cloud model (`deepseek-v4-flash:cloud`) matches Claude-class quality at ~36% lower latency, while small local models remain insufficient for the hardest multi-step queries. A 176B open-weight model then ran locally on two used GPUs and scored **0.906** on the 90-question benchmark — indistinguishable from `deepseek-v4-flash` on that set (ADR-0038). A multi-turn harness showed the same 11 questions scoring **0.750 or 0.950 depending only on their order**, making conversation hygiene a correctness control that the web chat (ADR-0039) puts in the operator's hands


## Research Methodology: Multi-Protocol Anti-Hallucination System

### Problem Definition
Traditional LLM applications suffer from hallucination—generating plausible but factually incorrect responses, particularly problematic in technical domains where accuracy is critical. This research systematically addresses hallucination through structured validation approaches.

### Phase 1: TMDB RAG API Routing + Constraint-Based Validation Pipeline

**Query Processing Architecture:**
```
User Query → Parse/Extract → Semantic Endpoint Retrieval → Entity Resolution → Plan Assembly → Execution → Validation → Formatting
```

**7-Phase Execution Pipeline:**
- **Phase 1**: Natural language parsing with entity/intent extraction
- **Phase 2**: Semantic search across 54 TMDB endpoint descriptions  
- **Phase 3**: Entity resolution via TMDB Search API calls
- **Phase 4**: Multi-step plan assembly with dependency injection
- **Phase 5**: Constraint-aware execution with validation
- **Phase 6**: Post-validation with credit verification
- **Phase 7**: Response formatting with provenance tracking

**Intent Recognition & Classification:**
- **LLM + SpaCy Integration**: Hybrid approach using language model understanding with structured linguistic analysis
- **Query Type Classification**: Movie search, person lookup, relationship queries, temporal constraints
- **Ambiguity Detection**: Identifies unclear queries requiring user clarification to prevent assumption-based responses

**RAG API Endpoint Routing Process:**
- **ChromaDB Vector Database**: Semantic search across 54 TMDB API endpoint descriptions for optimal routing
- **Sentence Transformers**: Query-to-vector conversion using all-MiniLM-L6-v2 model enabling similarity matching between user queries and API capabilities
- **Endpoint-Based Routing**: Vector similarity scores determine which TMDB APIs to call (e.g., /discover vs /search vs /person)
- **Semantic API Matching**: Retrieved endpoint embeddings guide query routing to prevent incorrect API usage

**Entity Resolution & Validation:**
- **TMDB Search API Integration**: Actors, directors, movies, genres resolved through TMDB Search API calls
- **Multi-Stage Entity Resolution**: Search API calls resolve "Tom Hanks" → person_id, then cross-reference validation
- **API-Verified Entity Resolution**: All entities validated through TMDB database lookups with constraint building

**Weighted Endpoint Selection Algorithm:**
- **Decision Matrix**: Routes between `/discover/movie`, `/person`, `/search` based on:
  - Query complexity coefficient (0.0-1.0)
  - Entity confidence levels (weighted average)
  - Expected result quality metrics
  - API endpoint performance characteristics
  - Weighted Endpoint Selection: Static scoring system optimizes API
  routing using semantic similarity and rule-based parameter matching

**Progressive Constraint Relaxation:**
- **Constraint Hierarchy**: Primary (must match) → Secondary (should match) → Tertiary (nice to have)
- **Systematic Fallbacks**: Expands date ranges, relaxes genre constraints, broadens search scope  
- **Provenance Logging**: Complete audit trail of which constraints were modified and why
- **Fallback Sequence**: Constraint relaxation → semantic fallback → generic discovery with minimal constraints

**Advanced Implementation Features:**
- **Intent Correction Logic**: Automatic detection and correction of mismatched movie/TV classifications
- **Symbol-Free vs Constraint-Based Routing**: Dynamic routing strategy based on query complexity
- **Episode-Level Data Requirements**: Specialized handling for TV writers/directors requiring episode-specific validation
- **Multi-Layer Validation**: Symbolic filtering, role validation, and post-execution credit verification
- **Dynamic Plan Injection**: Real-time plan expansion based on resolved dependencies and constraint satisfaction

### Phase 2: NetBox MCP Server (Shared Infrastructure)

**NetBoxLabs MCP Server Architecture:**
- **Generic Tool Design**: 3 tools handle all NetBox object types (get_objects, get_object_by_id, get_changelogs)
- **Field Filtering**: Specify exact fields needed to minimize token usage
- **Read-Only Interface**: Safe, idempotent operations for all queries
- **Shared Foundation**: Infrastructure layer used by both Phase 4 agent frameworks

**Claude Code API Integration Patterns:**
- **Query-to-Tool Mapping**: Natural language intent parsed and matched to appropriate MCP tools
- **Parameter Validation Pipeline**: Input sanitization, type checking, constraint validation
- **Multi-Tool Orchestration**: Intelligent sequencing of tool calls for complex queries
- **Response Aggregation**: Structured data combination with conflict resolution

**Read-Only Tool Execution Safety:**
- **Idempotent Operations**: All read tools guaranteed to have no side effects
- **Dual-Tool Validation**: "info" + "list_all" tools provide cross-verification
- **Structured Response Enforcement**: MCP schema validation prevents fabricated data structures
- **Context-Aware Safety Levels**: Environment detection (demo/staging/production) with appropriate controls

**Anti-Hallucination MCP Mechanisms:**
- **Real-Time Data Binding**: Direct NetBox API integration eliminates data staleness
- **Atomic Validation**: Read-validate-confirm pattern for all infrastructure queries
- **Enterprise Safety Controls**: Mandatory confirmation for write operations, comprehensive audit logging
- **Automatic Rollback**: Complex operations with built-in failure recovery

### Phase 4: Agent Framework Comparison Study

**Research Question**: What are the trade-offs between flexible, open-source agent frameworks (Deepagents/LangChain) versus opinionated, production-ready SDKs (Claude Agent SDK)?

**Methodology**: Implement the same NetBox infrastructure agent using both frameworks to empirically compare architectural approaches.

#### Implementation A: Deepagents (LangChain)
**Repository**: https://github.com/FinnMacCumail/deepagents

**Framework Philosophy**: Open-source, maximum flexibility, DIY orchestration

**Architecture:**
```
User Query → Deep Agent → LangGraph Workflow → Tool Discovery → Dynamic Wrappers → NetBox MCP Tools → Cache Monitor → Response
```

**Key Features:**
- Planning & task decomposition
- Sub-agent delegation capabilities
- Filesystem as memory pattern
- Custom tool integration
- LangGraph-based orchestration
- Dynamic tool discovery

#### Implementation B: Claude Agent SDK
**Repository**: https://github.com/FinnMacCumail/claude-agentic-netbox

**Framework Philosophy**: Official Anthropic SDK, production-ready out-of-box, managed orchestration

**Architecture:**
```
User Query → Claude Agent → MCP Protocol → NetBox Tools → WebSocket Stream → Response
```

**Key Features:**
- Model Context Protocol (MCP) integration
- Built-in permission system
- WebSocket streaming responses
- Session lifecycle management
- Type-safe implementation (Pydantic)
- Hook-based extensibility
- Intelligent multi-model routing (December 2025)
- Explicit model selection with cost optimization

#### Key Findings

**Both Frameworks Succeed:**
- ✅ Natural language NetBox queries work effectively
- ✅ Multi-step tool coordination successful
- ✅ Context management strategies apply to both
- ✅ Production deployment viable for both

**Different Optimization Targets:**
- **Deepagents**: Flexibility, control, experimentation (46h initial dev, 1,200 LOC)
- **Claude SDK**: Speed, reliability, maintenance (2.5h initial dev, 400 LOC)

**Framework Choice is Context-Dependent:**
- No universal "best" framework
- Choice depends on project maturity, team capabilities, business constraints
- Both can coexist: Claude SDK for production, Deepagents for R&D

**Cache Performance Monitoring:**
- **Hit Rate Analytics**: Real-time tracking of cache effectiveness across different tool types
- **Cost Optimization Tracking**: Detailed analysis of cost savings through intelligent caching strategies
- **Performance Profiling**: Response time analysis and optimization recommendations
- **Utilization Metrics**: Tool usage patterns and optimization opportunities identification

**Anti-Hallucination Orchestration Mechanisms:**
- **Tool Validation**: Pre-execution validation ensuring tool compatibility and parameter accuracy
- **Response Verification**: Post-execution validation of tool responses for consistency and accuracy
- **Conversation Continuity**: Multi-turn context preservation without cross-contamination
- **Orchestration Transparency**: Complete audit trail of tool selection, execution, and result processing

### Phase 5: Production DeepAgents — Measuring, Then Serving, the Agent

**Research Question**: Can the Phase 4 deepagents build be taken to production quality — evaluated systematically across local and cloud models, run on a frontier-class open-weight model locally, and served to a person — without the proxy layer that broke SDK features in Phase 4?

**Methodology**: One repository ([ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents)), one fixed NetBox dataset, and a LangSmith evaluation harness as the gate for every framework, prompt, middleware and model change; each result recorded as an ADR with its limits, and corrected in place when a later run overturned it.

- **Native local and cloud backends** (ADR-0028): Ollama and llama.cpp reached directly, resolving the local-model failure of ADR-0027 that a LiteLLM proxy had caused.
- **Reference-grounded correctness** (ADR-0033): a completeness metric certified a fabricated "7.7% utilization" as 0.9 complete; a correctness evaluator scored it 0.0 and reordered the model leaderboard toward the more truthful model.
- **Read-only GraphQL read path** (ADR-0034): an on-prem reproduction of NetBox Cloud's agent-native cross-domain reads; combined correctness 0.667 → ≈0.82 across three replications, with routing keyed on the number of anchor objects.
- **A benchmark's discriminating power is a property of the models tested** (ADR-0037): three similar families landed inside a 2.7pp band on 90 questions and looked saturated; a fourth, more distant family produced the first significant differences. The set resolves ~9pp gaps and cannot resolve ~3pp ones.
- **A frontier-class model on two used GPUs** (ADR-0038): Qwen3.8-Flash-Next (176B MoE) on 2× RTX 2080 Ti scored **0.906** on the full 90-question set — 0.5pp below `deepseek-v4-flash`, inside the set's resolution. `--numa isolate` alone was worth +56% decode; the local constraint is cost, not capability.
- **Conversation order is a correctness variable**: the same 11 questions down one accumulating thread scored **0.750** site-first and **0.950** tenant-first, because one turn reused a wrong population inherited from an earlier one. *Context reuse is not the defect — reusing a wrong population is.*
- **Serving the agent without changing it** (ADR-0039): a FastAPI + Nuxt web chat with token-level streaming, tool activity, cancel and server-reported token accounting, over an agent whose `query()`, prompt, middleware and skills are byte-identical to the benchmarked commit. The first streaming client found a failure the non-streaming harnesses could never hit — a 120 s watchdog tripped by ~167 s of silent prefill — and measured the framework middleware chain at **0.27%** of wall time.

### Cross-Phase Validation Results

**Quantitative Anti-Hallucination Metrics:**
- **TMDB Phase**: 95% accuracy across 50 test queries with progressive constraint relaxation
- **NetBox Phase**: 100% tool accessibility with structured response validation
- **Orchestration Failure**: Phase 3 achieved 0% success rate demonstrating complexity risks
- **Deepagents Success**: Phase 4 achieved successful Claude CLI replacement with intelligent caching
- **Phase 5 Correctness**: a locally served 176B model scored 0.906 on 90 stratified NetBox questions against re-scored cloud figures of 0.911 (flash), 0.928 (kimi) and 0.944 (pro), with 0 errors and zero truncations
- **Phase 5 Conversation Order**: identical questions scored 0.750 or 0.950 depending only on order across an 11-turn thread — error propagation, not capability
- **Phase 5 Serving**: framework middleware measured at 0.27% of wall time; a 120 s streaming watchdog found only by the browser path, never by the harnesses
- **Enterprise Safety**: Zero fabricated infrastructure data across production environments

**Key Research Innovations:**
1. **RAG-Powered API Orchestration**: Novel use of semantic search for API endpoint selection rather than content retrieval
2. **Semantic Query Routing**: Vector similarity matching user queries to optimal TMDB API endpoints from 54 endpoint descriptions
3. **Progressive Relaxation with Provenance**: Systematic fallbacks maintaining accuracy audit trails
4. **MCP Protocol Safety Integration**: Structured tool protocols eliminating response fabrication
5. **Orchestration Failure Analysis**: Documented evidence that complexity can reduce system reliability
6. **Simplified LangGraph Architecture**: Successful orchestration through streamlined deepagents framework
7. **Intelligent Caching Integration**: Cache performance monitoring with cost optimization tracking
8. **Multi-Protocol API Intelligence**: Seamless integration spanning TMDB, NetBox MCP, and deepagents orchestration
9. **Intelligent Multi-Model Routing Discovery**: Documented Claude SDK's automatic model selection (70-80% cost savings)
10. **Framework Architecture Dependencies**: Established that SDK features require direct API access; proxy layers break managed benefits
11. **Reference-Grounded Correctness Evaluation**: A completeness metric certifies confident fabrications; grounding the judge in verified references catches them and reorders the leaderboard
12. **Testing a Benchmark Before Trusting It**: Saturation turned out to be a property of the models compared, not of the questions; the set's resolution (~9pp) is now stated with every result
13. **Frontier-Class Inference on Consumer Hardware**: A 176B open-weight model on two used GPUs matches a frontier cloud model on this benchmark; the binding constraint is cost and throughput, not capability
14. **Measurement Path Must Match Product Path**: A non-streaming harness cannot find a streaming client's failure; the first browser turn exposed a watchdog that 90 benchmark questions never could

## Technical Implementation Highlights

- **RAG API Orchestration**: ChromaDB vector database storing 54 TMDB endpoint descriptions with semantic routing using all-MiniLM-L6-v2
- **Static Scoring System**: Fixed scoring weights (cast: 0.4, director: 0.4) with boost factors for media type and parameters
- **TMDB Search API Entity Resolution**: Direct API calls for "Tom Hanks" → person_id resolution with fuzzy matching support
- **Constraint Tree Logic**: Symbolic AND/OR constraint satisfaction with set intersection for multi-entity queries
- **Multi-Step Execution Engine**: 7-phase pipeline with dynamic plan expansion and dependency injection
- **Comprehensive Validation**: Post-execution credit verification ensuring accurate role attribution
- **NetBoxLabs MCP Server Integration**: Official 3-tool generic interface with field filtering for token optimization
- **Enterprise Safety Architecture**: Dual-tool validation, atomic operations, and comprehensive audit logging
- **Deepagents LangGraph Integration**: Claude Sonnet-4 with sophisticated workflow orchestration replacing failed multi-agent systems
- **Dynamic Tool Wrapper Generation**: Automatic discovery and wrapping of NetBox MCP tools with enhanced error handling
- **Intelligent Prompt Caching**: Configurable TTL-based caching with granular performance metrics and cost optimization
- **Context Quarantine Architecture**: Sub-agent isolation preventing conversation pollution with virtual file system
- **Cache Performance Monitoring**: Real-time hit rate analytics, cost savings tracking, and utilization pattern analysis
- **Orchestration Failure Recovery**: Lessons learned from 0% success rate Phase 3 applied to simplified Phase 4 architecture
- **Dual Native Model Backend**: DeepAgents 0.7.5 with Ollama (local and cloud) and llama.cpp (OpenAI-compatible) selected by one environment variable, no proxy layer
- **Filter-Recovery Middleware**: NetBox MCP filter violations converted into structured `TOOL_VALIDATION_ERROR` / `TOOL_API_ERROR` messages the model can recover from, instead of opaque HTTP 400s
- **Read-Only GraphQL Path**: `graphql-core` AST inspection rejects mutations before any request; depth, size and timeout limits bound query cost
- **Local Frontier Inference Tuning**: `llama-server` with `--numa isolate -t 10`, `-c 131072`, `-b 2048 -ub 2048` — each flag measured, and the dead ends (n-gram speculation, KV offload) recorded
- **Web Serving Layer**: FastAPI + WebSocket back end and Nuxt 3 front end with per-call token accounting from the server's own `usage` and `timings`, a context gauge, cancel, and per-turn LangSmith trace links

## Implementation Repositories
- **Phase 1**: TMDB Chatbox – https://github.com/FinnMacCumail/tmdbGPT
- **Phase 2**: NetBox MCP Server – https://github.com/netboxlabs/netbox-mcp-server
- **Phase 3**: OpenAI Orchestration (Failed) – No working implementation (0% success rate)
- **Phase 4**: Deepagents Solution – https://github.com/FinnMacCumail/deepagents *(superseded by Phase 5)*
- **Phase 5**: Production DeepAgents – https://github.com/FinnMacCumail/ollamaDeepAgents

## Development Milestones

### Completed Phases

- **Phase 1 ✅**: TMDB Chatbox – Natural language movie/TV query system
- **Phase 2 ✅**: NetBox MCP Server – Official NetBoxLabs MCP server with 3 generic tools
- **Phase 3 ❌ FAILED**: OpenAI Orchestration – 0% success rate (see ADR-0013)
- **Phase 4 ✅ COMPLETED**: Agent Framework Comparison Study
  - **Achievement**: Empirical comparison of Deepagents vs Claude SDK frameworks
  - **Implementation A**: [Deepagents](https://github.com/FinnMacCumail/deepagents) - Flexible, research-oriented framework
  - **Implementation B**: [Claude SDK](https://github.com/FinnMacCumail/claude-agentic-netbox) - Production-ready, managed framework
  - **Key Finding**: Framework choice is context-dependent; both successfully build production agents
  - **Performance Results**: Intelligent prompt caching, cost optimization through cache monitoring, natural language NetBox queries
  - **Architecture**: Deepagents framework with automatic NetBox MCP tool wrapper generation and sophisticated cache performance tracking
  - **Model Selection Enhancement (December 2025)**: Discovered intelligent multi-model routing (70-80% cost savings), implemented explicit model selection, documented failed Ollama/LiteLLM integration attempt

- **Phase 5 ✅ COMPLETED (June–September 2026)**: Production DeepAgents – Multi-Model, Evaluation, Local Inference & Serving
  - **Repository**: [ollamaDeepAgents](https://github.com/FinnMacCumail/ollamaDeepAgents) (supersedes the Phase 4 Deepagents build)
  - **Achievement**: DeepAgents 0.7.5, dual local/cloud model backend (no proxy), LangSmith model-matrix and reference-grounded-correctness evaluation, trace-driven observability, a read-only GraphQL read path, a 176B model served locally, and a web chat over that local model with server-reported token accounting and per-turn trace links
  - **Key Finding**: Resolved ADR-0027's local-model failure; `deepseek-v4-flash:cloud` matches Claude-class quality at ~36% lower latency, small local models remain insufficient for the hardest queries, and a 176B open-weight model run locally is indistinguishable from flash on the 90-question set (ADR-0038)
  - **Decisions**: ADRs 0028 (native local+cloud) · 0029 (LangSmith) · 0030 (eval harness) · 0031 (0.6 upgrade) · 0032 (QuickJS/PTC deferral) · 0033 (correctness evaluator) · 0034 (GraphQL read path) · 0035 (0.7.5 upgrade) · 0036 (LangChain vs cloud MCP) · 0037 (v5 benchmark) · 0038 (local frontier model) · 0039 (web serving layer)

### Upcoming Milestones

**Phase 6: Neo4j Graph Intelligence**
- **Objective**: Add pre-computed relationship intelligence using Neo4j graph database for instant complex queries
- **Capabilities**: Graph-based relationship queries, real-time synchronization, hybrid routing intelligence
- **Impact**: Enable previously impossible relationship queries (20-50x faster complex analysis)

**Phase 7: RAG-Powered Semantic Intelligence**
- **Objective**: Add contextual understanding through operational documentation and institutional memory integration
- **Capabilities**: Semantic search across documentation, contextual recommendations, operational pattern recognition
- **Impact**: Transform system into organizational knowledge advisor with historical context

**Phase 8: Advanced Analytics Platform**
- **Objective**: Deploy graph algorithms and predictive analytics for intelligent infrastructure insights
- **Capabilities**: Network bottleneck identification, capacity planning, predictive maintenance, operational intelligence
- **Impact**: Predictive capabilities delivering 10-100x operational efficiency gains

👉 **Phase 5 Success**: The deepagents build reached production quality with a measured evaluation harness, a locally served frontier-class model and a browser front end, where Phase 3 orchestration had failed. The upcoming milestones (Phases 6–8) build on this foundation.

