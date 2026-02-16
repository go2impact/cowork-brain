# 3. MCP Integration

[< Back to index](README.md)

**Their claim:** "Support for MCP (Model Context Protocol) client with extensible tool capabilities"

**Maps to Cowork.ai:** [MCP Integrations](../../product/product-features.md#2-mcp-integrations) — same protocol, but we also provide MCP servers to third-party apps (they don't).

## How It Works

Users add MCP servers via config (stdio or HTTP transport). `ToolsManager` creates `MCPClient` instances from `@mastra/mcp`, discovers tools via `listTools()`, and makes them available to agents. Additionally, AIME exposes its own built-in tools as an MCP server via an Express endpoint.

**Key files:**
- `src/main/tools/index.ts` — ToolsManager (1,404 lines), MCPClient management
- `src/main/tools/mcp/oauth-client-provider.ts` — OAuth for MCP servers
- `src/main/mastra/index.ts` — Express `/mcp` endpoint
- `src/types/mcp.ts` — Type definitions

## MCP Connection Architecture

```
┌────────────────────── AIME CHAT (Electron Main) ──────────────────────┐
│                                                                        │
│  ToolsManager                                                          │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  mcpClients[]                                                  │   │
│  │                                                                │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │   │
│  │  │ MCPClient #1 │  │ MCPClient #2 │  │ MCPClient #3 │        │   │
│  │  │ (stdio)      │  │ (HTTP/SSE)   │  │ (stdio)      │        │   │
│  │  │ status: run  │  │ status: run  │  │ status: stop │        │   │
│  │  └──────┬───────┘  └──────┬───────┘  └──────────────┘        │   │
│  │         │                  │                                    │   │
│  └─────────┼──────────────────┼────────────────────────────────────┘   │
│            │                  │                                        │
│            ▼                  ▼                                        │
│   ┌────────────────┐  ┌────────────────┐                              │
│   │ External MCP   │  │ External MCP   │                              │
│   │ Server (local) │  │ Server (remote)│                              │
│   │ e.g. npx ...   │  │ e.g. https://  │                              │
│   └────────────────┘  └────────────────┘                              │
│                                                                        │
│  McpServer (built-in tools exposed AS an MCP server)                  │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  Express POST /mcp                                              │   │
│  │       │                                                         │   │
│  │       ▼                                                         │   │
│  │  StreamableHTTPServerTransport                                  │   │
│  │       │                                                         │   │
│  │       ▼                                                         │   │
│  │  toolsManager.mcpServer.connect(transport)                      │   │
│  │  ├─ bash, read, write, edit, glob, grep                        │   │
│  │  ├─ web_search, web_fetch                                      │   │
│  │  ├─ code_execution, vision, audio                              │   │
│  │  └─ ... (all 20+ built-in tools)                               │   │
│  └────────────────────────────────────────────────────────────────┘   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

## MCP Tool Lifecycle

```
1. CONFIG ──▶ User adds MCP server in Settings UI
                 │
                 ▼
             ToolsManager.saveMCPServer()
                 ├─ Parse JSON config { mcpServers: { name: definition } }
                 ├─ configToMastraMCPServerDefinition()
                 │    ├─ stdio: { command, args, env }
                 │    └─ HTTP:  { url, requestInit: { headers } }
                 ├─ new MCPClient({ servers: { [name]: definition } })
                 ├─ Save to DB (Tools entity, type: ToolType.MCP)
                 └─ Add to mcpClients[]

2. DISCOVER ──▶ mcpClient.listTools()
                 │
                 ▼
             Returns Record<"serverName_toolName", ToolDefinition>
             Stored in mcpClient.mcp.tools

3. BUILD ──▶ agentManager.buildAgent(agentId, { tools: [...] })
                 │
                 ▼
             toolsManager.buildTools(toolIds)
                 ├─ For "build-in:*" → instantiate class, createTool()
                 ├─ For "mcp:*" → filter running clients, extract tool
                 └─ For "skill:*" → load from skillManager

4. EXECUTE ──▶ agent.stream() → model decides tool_use
                 │
                 ▼
             MCPClient routes to MCP server → execute → result
                 │
                 ▼
             Result fed back into agent loop

5. CLEANUP ──▶ User deletes MCP server
                 └─ mcp.disconnect() → remove from DB
```

## OAuth Support

```
{userData}/.mcp/{serverUrlHash}_client_info.json    ◀── OAuth client metadata
{userData}/.mcp/{serverUrlHash}_tokens.json         ◀── Access/refresh tokens
{userData}/.mcp/{serverUrlHash}_code_verifier.txt   ◀── PKCE verifier

(where {userData} = app.getPath('userData'), e.g. ~/Library/Application Support/aime-chat/)
```

Implements `OAuthClientProvider` from `@modelcontextprotocol/sdk/client/auth.js`. Opens browser for login, stores tokens locally.

## What We Should Do

| Action | Detail |
|--------|--------|
| **Copy pattern** | Their MCPClient array management — status tracking ('starting'/'running'/'stopped'/'error'), reconnect logic, toggle on/off. Direct reference for our MCP Integrations connection health UI. |
| **Copy pattern** | The McpServer that exposes built-in tools — we need this for our "platform-provided MCPs" that apps access. |
| **Study** | OAuth credential flow stored in `~/.mcp/`. We need OAuth for MCP servers connecting to Zendesk, Gmail, Slack. |
| **Study** | `configToMastraMCPServerDefinition()` — normalizes stdio vs HTTP transport config. Clean abstraction. |
| **Note** | They are both MCP client (connecting to external servers) and MCP server (exposing built-in tools via Express `/mcp`). However, their server is for internal tool access, not for third-party apps. We additionally provide MCP servers to third-party apps with scoped permissions — a fundamentally different model. |
