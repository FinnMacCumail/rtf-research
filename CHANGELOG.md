# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
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
