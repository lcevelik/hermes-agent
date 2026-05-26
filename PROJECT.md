# PROJECT.md — Hermes Agent

> Self-maintained by: lcevelik
> Upstream: https://github.com/NousResearch/hermes-agent.git
> Fork: https://github.com/lcevelik/hermes-agent.git
> Version: 0.14.0 | License: MIT | Language: Python 3.11+

---

## Goals

- Maintain a personal fork of Hermes Agent for customization and contribution
- Track upstream NousResearch changes and selectively merge
- Identify and implement improvements to architecture, security, and performance
- Build expertise in the codebase for potential upstream contributions

## In Progress

- [x] Initial repo setup: lcevelik/hermes-agent created with full git history
- [x] Remote configuration: origin → lcevelik, upstream → NousResearch
- [x] Deep code analysis and PROJECT.md creation

## To Do

### Architecture Improvements
- [ ] Extract `run_agent.py` (4,246 LOC) into smaller modules — already partially done (conversation_loop.py extracted at 4,191 LOC), but the main class still has ~60 constructor params
- [ ] Reduce `cli.py` (14,781 LOC) — largest single file, should be split into sub-modules
- [ ] Introduce dependency injection / config object pattern for AIAgent constructor (currently ~60 positional params)
- [ ] Add async support to the core agent loop — currently entirely synchronous, blocks on tool calls
- [ ] Standardize the plugin interface — currently plugins use various patterns (some are directories, some single files, no formal ABC/protocol)

### Security
- [ ] Audit SSRF protection (url_safety.py) — DNS rebinding TOCTOU gap is documented but not mitigated; consider connection-level validation
- [ ] Add rate limiting on gateway API server endpoint (gateway/platforms/api_server.py, 3,524 LOC)
- [ ] Review MCP OAuth token storage — tokens stored in plaintext JSON files under ~/.hermes/
- [ ] Add Content Security Policy headers to web dashboard
- [ ] Audit credential_pool.py for credential leakage in logs (AUTH_TYPE constants are masked but log messages may leak)
- [ ] Review subagent auto-approve path — delegate_tool.py allows auto-approve for cron/batch, which could be exploited if cron jobs are user-controlled

### Testing
- [ ] Add integration tests for MCP tool (tools/mcp_tool.py, 3,584 LOC) — currently only unit tests exist
- [ ] Add load/stress tests for gateway platform adapters under concurrent messages
- [ ] Improve test coverage for error_classifier.py failover paths
- [ ] Add end-to-end tests for the full agent loop with mock LLM responses
- [ ] Test credential pool rotation under concurrent access

### Performance
- [ ] Profile cold-start time — lazy OpenAI SDK import already in place (auxiliary_client.py), but other heavy imports remain
- [ ] Optimize FTS5 session search (hermes_state.py, 3,273 LOC) — consider adding trigram indexes for fuzzy matching
- [ ] Cache tool schema generation — currently regenerated every conversation turn
- [ ] Implement connection pooling for MCP server connections (currently one connection per server per session)
- [ ] Consider using `uvloop` for the MCP background event loop

### New Features
- [ ] Add streaming support to all platform adapters (currently partial)
- [ ] Implement skill versioning and dependency management
- [ ] Add web-based skill editor with live preview
- [ ] Implement conversation branching (fork from any point in history)
- [ ] Add multi-agent collaboration protocols beyond delegate/spawn pattern

## Done

- [x] Repo created at https://github.com/lcevelik/hermes-agent (2026-05-26)
- [x] Full git history pushed (9,230 commits)
- [x] Remotes configured: origin=lcevelik, upstream=NousResearch
- [x] Deep codebase analysis completed

## Blocked

- (none currently)

## Releases

| Version | Date | Notes |
|---------|------|-------|
| 0.14.0 | Current | Latest upstream from NousResearch |

## Notes

### Codebase Statistics

| Metric | Value |
|--------|-------|
| Total Python files | 1,813 |
| Total Python LOC | ~876,839 |
| Test files | 1,192 |
| Git commits | 9,230 |
| Built-in skills | 25 |
| Optional skills | 18 |
| Gateway platforms | 15+ (Telegram, Discord, Slack, WhatsApp, Signal, Matrix, etc.) |
| Terminal backends | 7 (local, Docker, SSH, Singularity, Modal, Daytona, Vercel) |

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Entry Points                           │
│  cli.py (14,781 LOC)  │  gateway/run.py (18,204 LOC)       │
│  hermes CLI           │  Messaging platforms                │
└───────────┬───────────┴──────────────┬──────────────────────┘
            │                          │
            ▼                          ▼
┌─────────────────────────────────────────────────────────────┐
│                   run_agent.py (4,246 LOC)                   │
│              AIAgent class — core conversation loop          │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  agent/conversation_loop.py (4,191 LOC)              │    │
│  │  run_conversation() — model call, tool dispatch,     │    │
│  │  retries, fallbacks, compression, post-turn hooks    │    │
│  └─────────────────────────────────────────────────────┘    │
│  ┌─────────────────┐  ┌──────────────────┐                   │
│  │ agent/auxiliary  │  │ agent/credential │                   │
│  │ _client.py      │  │ _pool.py         │                   │
│  │ (5,289 LOC)     │  │ (1,955 LOC)      │                   │
│  │ LLM fallback    │  │ Multi-cred pool  │                   │
│  │ chain router    │  │ with failover    │                   │
│  └─────────────────┘  └──────────────────┘                   │
└────────────────────────────┬────────────────────────────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
   ┌──────────────┐  ┌─────────────┐  ┌─────────────────┐
   │ model_tools  │  │  MCP Tools  │  │  Skills System  │
   │ .py (923 LOC)│  │ (3,584 LOC) │  │ (1,567 LOC)     │
   │ Tool orch.   │  │ MCP client  │  │ Auto-created    │
   │ + discovery  │  │ integration │  │ from experience │
   └──────┬───────┘  └─────────────┘  └─────────────────┘
          │
          ▼
   ┌─────────────────────────────────────────────────┐
   │            tools/ (60,617 LOC total)              │
   │  terminal (2,379) │ browser (3,796) │ delegate   │
   │  file_ops (1,910) │ web (1,561)     │ mcp (3,584)│
   │  voice (1,129)    │ vision (1,421)  │ kanban etc │
   └─────────────────────────────────────────────────┘
```

### Key Design Patterns

1. **Tool Registry** (tools/registry.py): Central registration system. Each tool file calls `registry.register()` at module level. `model_tools.py` queries registry instead of maintaining parallel data structures.

2. **Credential Pool** (agent/credential_pool.py): Multi-credential failover with strategies (fill_first, round_robin, random, least_used). Supports OAuth refresh, cooldown timers for exhausted credentials, and per-provider state persistence.

3. **Auxiliary Client Router** (agent/auxiliary_client.py): 7-level fallback chain for side tasks (compression, search, vision). Auto-detects best available provider. Handles 402 credit exhaustion by falling through to next provider.

4. **Error Classifier** (agent/error_classifier.py): Structured taxonomy of API errors → recovery strategies (retry, rotate credential, fallback, compress context, abort). Replaces scattered inline string matching.

5. **Gateway Platform Adapters** (gateway/platforms/base.py): Abstract base class. Each platform (Telegram, Discord, Slack, etc.) implements required methods. 15+ adapters, with Telegram being the largest (5,656 LOC).

6. **Prompt Caching** (agent/prompt_caching.py): Anthropic-specific cache control with `system_and_3` layout — caches system prompt + last 3 messages for ~75% input token cost reduction.

7. **MCP Integration** (tools/mcp_tool.py): Full Model Context Protocol client with stdio, HTTP/StreamableHTTP, and SSE transports. Thread-safe with dedicated background event loop. Supports server-initiated sampling.

8. **Security Layers**:
   - `tools/url_safety.py` — SSRF protection with cloud metadata IP blocklist
   - `tools/path_security.py` — Directory traversal prevention
   - `agent/file_safety.py` — Write-deny for SSH keys, .env, shell configs
   - `tools/tirith_security.py` — Pre-exec command scanning (homograph URLs, terminal injection)

### Dependency Strategy

All core dependencies are exact-pinned (==X.Y.Z) — no ranges. Rationale: supply-chain attack prevention (informed by the Mini Shai-Hulud worm incident). Provider-specific deps are in optional extras, lazy-installed via `tools/lazy_deps.py`.

### Potential Issues Found

1. **Large single files**: cli.py (14,781 LOC) and gateway/run.py (18,204 LOC) are too large for effective code review and testing
2. **Constructor complexity**: AIAgent.__init__ takes ~60 parameters — fragile, hard to test, easy to get wrong
3. **Synchronous core loop**: The entire agent loop is synchronous, blocking on every tool call. The MCP layer works around this with a dedicated async event loop, but the main loop doesn't benefit from async
4. **1,145 TODO/FIXME/HACK comments**: Significant technical debt markers across the codebase
5. **No formal plugin protocol**: Plugins use various patterns — some are packages with plugin.py, some are single files, some register at import time. No Plugin ABC or Protocol class
6. **Credential storage**: OAuth tokens and API keys stored as plaintext JSON in ~/.hermes/auth.json and ~/.hermes/.env. Consider OS keyring integration (partially implemented for CLI auth but not for all providers)
