# Phase 2 – NetBox MCP + Claude Code

## Goals
- Use MCP tools to interact with NetBox via Claude Code
- Handle pagination/limits; avoid token overflows
- Prototype knowledge-graph strategies to avoid N+1 calls

## Status
- Claude Code API integrated and MCP server running locally
- Next: research-dev UX (prompts, guardrails), performance, KG ingestion plan

## Demo Scenarios
- “List VLANs” with pagination summary + follow-ups
- “Show interfaces for device X” with summarization + page-through
- “Explain cable relationships for device Y” with structured output
