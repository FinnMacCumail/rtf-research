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

### Fixed
- **Mixed Content Resolution**: Resolved critical bug in TMDB Phase 1 where TV queries ("comedy shows") returned mixed TV/movie results
  - Root cause: Missing "shows" indicator in media type detection function
  - Solution: Enhanced `infer_media_type_from_query()` with comprehensive TV indicators
  - Impact: 100% resolution rate across all TV query variations
- Enhanced entity resolution for BBC and Hulu network queries with geographic preferences

### Architecture Decision Records
- **ADR-0003**: Diagnostic-First Debugging Methodology - systematic execution tracing for complex pipeline failures
- **ADR-0004**: Media Type Detection Enhancement Strategy - comprehensive natural language indicator coverage  
- **ADR-0005**: Entity Resolution Cache Override Pattern - geographic preference handling for ambiguous entities

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
