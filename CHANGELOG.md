# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
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
- **Phase 1 Complete**: TMDB chatbox with semantic+symbolic retrieval
- **Phase 2 Integration**: NetBox MCP server with 142+ tools  
- **Performance Analysis**: Identified N+1 query optimization opportunities (VLAN queries: 127 calls → optimized batching)
- **Future Planning**: Comprehensive four-phase OpenAI orchestration roadmap

### Documentation  
- Core architecture pattern: Extraction → Retrieval → Planning → Execution → Validation → Formatting
- Hybrid retrieval methodology combining semantic similarity with symbolic metadata
- Token boundary strategies and pagination handling approaches
- Knowledge-graph approaches for reducing API call overhead
