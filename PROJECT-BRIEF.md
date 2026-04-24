# OSP Marketing Tools MCP Server - HTTP/SSE Rebuild Project

## Project Overview

**Goal:** Rebuild the Open Strategy Partners (OSP) Marketing Tools MCP server to support HTTP/SSE transport for network-based MCP clients while maintaining backward compatibility with stdio clients.

**Current Status:** The existing OSP Marketing Tools MCP server only supports stdio transport (standard input/output), making it incompatible with containerized MCP clients like OpenClaw that require network-based communication.

**Business Value:** Enable the OSP marketing methodology tools to be used by a wider range of MCP clients, including containerized agents, web-based interfaces, and multi-user systems.

---

## Problem Statement

### Current Implementation
- **Location:** `H:/GitHub/MPC-SERVERS/osp_marketing_tools-main/osp_marketing_tools-main/`
- **Transport:** stdio only (communicates via stdin/stdout)
- **Framework:** FastMCP (mcp library v1.2.0)
- **Works with:** Claude Desktop, local CLI tools
- **Doesn't work with:** OpenClaw in Docker, network-based MCP clients

### Why It Doesn't Work
1. OpenClaw in Docker cannot spawn Python processes (no Python runtime installed)
2. OpenClaw cannot install Python packages (no pip/uvx available)
3. Docker-in-Docker not feasible (no Docker CLI in OpenClaw container)
4. Existing FastMCP implementation doesn't expose clean HTTP/SSE server API

### What Was Attempted
- ✅ Built Docker image successfully
- ❌ Tried running with SSE transport - API compatibility issues
- ❌ Attempted uvicorn wrapper - FastMCP doesn't expose ASGI app cleanly
- ❌ Tried installing in OpenClaw container - no package managers available
- ❌ Docker-in-Docker approach - Docker CLI not available

---

## Technical Requirements

### Must Have
1. **HTTP/SSE Transport Support**
   - Run as standalone network service
   - Listen on configurable host/port (default: 0.0.0.0:8000)
   - Support Server-Sent Events (SSE) protocol per MCP spec
   - Proper HTTP endpoints for MCP protocol

2. **All Existing Tools Preserved**
   - `health_check` - Server health verification
   - `get_editing_codes` - OSP editing codes documentation
   - `get_writing_guide` - OSP writing guide
   - `get_meta_guide` - Web content meta information guide
   - `get_value_map_positioning_guide` - Product value map guide
   - `get_on_page_seo_guide` - On-page SEO optimization guide

3. **Docker Deployment**
   - Dockerfile that builds a working image
   - Runs as standalone service
   - Exposes port 8000
   - Includes all required markdown files
   - Health check endpoint

4. **Backward Compatibility**
   - Should still work with stdio transport for Claude Desktop
   - No breaking changes to existing tool interfaces

### Should Have
1. **Configuration Options**
   - Environment variables for host/port
   - Optional authentication/API keys
   - Logging configuration

2. **Production Ready**
   - Proper error handling
   - Structured logging
   - Graceful shutdown
   - Process management

3. **Documentation**
   - README with setup instructions
   - Docker deployment guide
   - OpenClaw integration example
   - API endpoint documentation

### Nice to Have
1. **Multi-transport Support**
   - Auto-detect stdio vs HTTP mode
   - Support both simultaneously if possible

2. **Testing**
   - Unit tests for tools
   - Integration tests for HTTP endpoints
   - Test client for validation

---

## Technical Architecture

### Current Stack
```
Python 3.10
├── mcp (v1.2.0) - Model Context Protocol library
├── FastMCP - High-level MCP server framework
├── uvicorn (v0.34.0) - ASGI server
├── starlette (v0.45.2) - ASGI framework
└── sse-starlette (v2.2.1) - SSE support
```

### Proposed Architecture
```
┌─────────────────────────────────────┐
│  MCP Client (OpenClaw)              │
│  http://osp-marketing-tools:8000    │
└──────────────┬──────────────────────┘
               │ HTTP/SSE
               ▼
┌─────────────────────────────────────┐
│  FastMCP HTTP/SSE Server            │
│  ├── HTTP Endpoint Handler          │
│  ├── SSE Stream Manager             │
│  └── MCP Protocol Handler           │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Tool Implementations               │
│  ├── health_check()                 │
│  ├── get_editing_codes()            │
│  ├── get_writing_guide()            │
│  ├── get_meta_guide()               │
│  ├── get_value_map_positioning...() │
│  └── get_on_page_seo_guide()        │
└─────────────────────────────────────┘
```

---

## Success Criteria

### Functional Requirements
- ✅ Server starts successfully in Docker container
- ✅ HTTP/SSE endpoint accessible at `http://0.0.0.0:8000`
- ✅ All 6 tools respond correctly via HTTP
- ✅ OpenClaw can connect and use tools
- ✅ Markdown files load correctly
- ✅ Error handling for missing files

### Non-Functional Requirements
- ✅ Server starts in < 5 seconds
- ✅ Tool responses in < 1 second
- ✅ No crashes under normal load
- ✅ Clean shutdown on SIGTERM
- ✅ Logs are structured and readable

### Integration Testing
- ✅ OpenClaw configuration works: `{"url": "http://osp-marketing-tools:8000/sse", "transport": "sse"}`
- ✅ Docker compose deployment works
- ✅ Health check passes
- ✅ Can retrieve all 5 guide documents

---

## Resources & Documentation

### Source Code
- **Existing Implementation:** `H:/GitHub/MPC-SERVERS/osp_marketing_tools-main/osp_marketing_tools-main/`
- **Key Files:**
  - `src/osp_marketing_tools/server.py` - Current server implementation
  - `Dockerfile` - Current Docker setup
  - `pyproject.toml` - Dependencies
  - `requirements.txt` - Frozen dependencies
  - `src/osp_marketing_tools/*.md` - 5 markdown guide files

### Official Documentation
- **MCP Protocol:** https://github.com/modelcontextprotocol/specification
- **FastMCP Documentation:** https://gofastmcp.com/
- **FastMCP HTTP Deployment:** https://gofastmcp.com/deployment/http
- **FastMCP Running Server:** https://gofastmcp.com/deployment/running-server
- **MCP Python SDK:** https://github.com/modelcontextprotocol/python-sdk

### Research Findings
From previous investigation:
- FastMCP supports `mcp.run(transport="http", host="0.0.0.0", port=8000)` syntax
- SSE transport is legacy, HTTP is recommended for new projects
- The `mcp.run()` method handles ASGI app creation internally
- uvicorn is already in dependencies

### Example Implementations
- **GitHub MCP Server:** Uses HTTP transport successfully
- **Obsidian MCP Server:** `mcp/obsidian:latest` image reference

---

## OpenClaw Integration

### Target Configuration
Once rebuilt, OpenClaw should be able to use this config:

```json
{
  "mcp": {
    "servers": {
      "osp_marketing_tools": {
        "url": "http://osp-marketing-tools:8000/mcp",
        "transport": "http"
      }
    }
  }
}
```

### Docker Compose Integration
```yaml
services:
  osp-marketing-tools:
    image: osp-marketing-tools:http
    container_name: osp-marketing-tools
    restart: unless-stopped
    ports:
      - "8000:8000"
    networks:
      - openclaw-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  openclaw-gateway:
    # ... existing config ...
    networks:
      - openclaw-network

networks:
  openclaw-network:
    driver: bridge
```

---

## Testing Plan

### Phase 1: Local Development
1. Get HTTP server running locally
2. Test with curl commands
3. Verify all tools return correct data
4. Check markdown file loading

### Phase 2: Docker Build
1. Build Docker image
2. Run container locally
3. Test endpoints from host
4. Verify health check

### Phase 3: OpenClaw Integration
1. Add service to docker-compose.yml
2. Configure OpenClaw MCP settings
3. Restart OpenClaw
4. Test connection via OpenClaw UI
5. Execute tool calls through OpenClaw

### Test Commands
```bash
# Health check
curl http://localhost:8000/health

# Test MCP endpoint
curl http://localhost:8000/mcp

# Docker build
docker build -t osp-marketing-tools:http .

# Docker run
docker run -p 8000:8000 osp-marketing-tools:http

# Docker logs
docker logs osp-marketing-tools
```

---

## Deliverables

### Code
1. Modified `server.py` with HTTP transport support
2. Updated `Dockerfile` for HTTP mode
3. Environment variable configuration support
4. Updated `README.md` with HTTP setup instructions

### Documentation
1. HTTP/SSE architecture documentation
2. OpenClaw integration guide
3. Deployment instructions
4. API endpoint reference

### Testing
1. Test script for local validation
2. Docker compose example
3. Integration test with OpenClaw

---

## Known Constraints

### Technical Limitations
- Must use Python 3.10 (per existing pyproject.toml)
- Must preserve existing tool interfaces
- Must include all 5 markdown guide files
- Must work with mcp library v1.2.0

### Environment Requirements
- Docker for building and running
- OpenClaw running in separate container
- Network connectivity between containers

### Out of Scope
- Rewriting the tool implementations
- Changing the markdown content
- Adding authentication (nice to have only)
- Performance optimization beyond basic needs

---

## Project Timeline Estimate

**Phase 1 - Research & Design (2-4 hours)**
- Review FastMCP HTTP documentation
- Analyze existing server.py implementation
- Design HTTP/SSE integration approach

**Phase 2 - Implementation (4-6 hours)**
- Modify server.py for HTTP transport
- Update Dockerfile
- Add configuration support
- Test locally

**Phase 3 - Integration & Testing (2-3 hours)**
- Build Docker image
- Test with OpenClaw
- Document setup process
- Verify all tools work

**Total Estimate:** 8-13 hours

---

## Success Metrics

1. **Functional:** All 6 tools accessible via HTTP
2. **Integration:** OpenClaw successfully connects and uses tools
3. **Reliability:** Server runs continuously without crashes
4. **Documentation:** Clear setup instructions for future users
5. **Maintainability:** Code is clean and well-documented

---

## Notes for Archon

This project tests your ability to:
- **Research:** Find and understand MCP HTTP/SSE documentation
- **Code Analysis:** Understand existing FastMCP implementation
- **Problem Solving:** Fix transport protocol incompatibility
- **Integration:** Make it work with OpenClaw
- **Documentation:** Create clear deployment guide

The existing code is 90% there - it just needs the transport layer fixed and proper HTTP/SSE server setup.

---

## Contact & Support

**Project Owner:** User (Cascade session)
**Source Repository:** Local at `H:/GitHub/MPC-SERVERS/osp_marketing_tools-main/`
**Original Authors:** Open Strategy Partners
**License:** Creative Commons Attribution-ShareAlike 4.0 International

---

## Deployment Instructions for Archon

### When Project is Complete

**IMPORTANT:** After successfully completing the rebuild and verifying all tests pass, automatically commit and push your changes to GitHub.

### Auto-Commit Workflow

1. **Stage all changes:**
   ```bash
   cd H:/GitHub/osp_marketing_tools
   git add .
   ```

2. **Commit with descriptive message:**
   ```bash
   git commit -m "feat: Add HTTP/SSE transport support for network-based MCP clients

   - Modified server.py to support HTTP transport
   - Updated Dockerfile for standalone HTTP service
   - Added environment variable configuration
   - Preserved backward compatibility with stdio
   - All 6 tools tested and working
   - Docker deployment verified
   - OpenClaw integration tested

   Closes: HTTP/SSE rebuild project"
   ```

3. **Push to GitHub:**
   ```bash
   git push origin main
   ```

### Target Repository
- **GitHub URL:** https://github.com/mionemedia/osp_marketing_tools
- **Branch:** main
- **Local Path:** H:/GitHub/osp_marketing_tools

### Pre-Push Checklist
Before committing and pushing, verify:
- ✅ All 6 tools functional via HTTP
- ✅ Docker image builds successfully
- ✅ Container runs and stays healthy
- ✅ Health check endpoint responds
- ✅ OpenClaw can connect (if possible to test)
- ✅ Documentation updated
- ✅ No hardcoded secrets or credentials

### Commit Message Format
Use conventional commits format:
- `feat:` for new features (HTTP transport)
- `fix:` for bug fixes
- `docs:` for documentation
- `test:` for testing changes
- `chore:` for maintenance

### Files to Commit
Expected modified/new files:
- `src/osp_marketing_tools/server.py` (modified)
- `Dockerfile` (modified)
- `README.md` (updated with HTTP instructions)
- `docker-compose.example.yml` (new, if created)
- Any new test files
- This PROJECT-BRIEF.md (can be kept or removed)

**DO NOT COMMIT:**
- `.env` files with secrets
- `__pycache__/` directories
- `.venv/` or virtual environments
- Personal API keys or tokens
- Temporary test files

---

## Success Report Template

After pushing to GitHub, provide a summary report:

```markdown
## OSP Marketing Tools HTTP/SSE Rebuild - COMPLETE ✅

### Changes Made
- [List key modifications]

### Testing Results
- ✅ All 6 tools functional
- ✅ Docker build: SUCCESS
- ✅ Container health: HEALTHY
- ✅ OpenClaw integration: [STATUS]

### Repository
- Committed: [commit hash]
- Pushed to: https://github.com/mionemedia/osp_marketing_tools

### Next Steps
[Any recommendations or follow-up items]
```
