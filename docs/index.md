# yorishiro-proxy

<p align="center">
  <img src="images/yorishiro-proxy_banner.png" alt="yorishiro-proxy" width="600">
</p>

<p align="center">
  <strong>AI-first MITM proxy tool</strong><br>
  A network proxy for AI agents — intercept, record, and replay traffic through MCP.
</p>

<p align="center">
  <a href="https://github.com/usk6666/yorishiro-proxy/actions/workflows/ci.yml"><img src="https://github.com/usk6666/yorishiro-proxy/actions/workflows/ci.yml/badge.svg" alt="CI"></a>
  <a href="https://goreportcard.com/report/github.com/usk6666/yorishiro-proxy"><img src="https://goreportcard.com/badge/github.com/usk6666/yorishiro-proxy" alt="Go Report Card"></a>
  <a href="https://github.com/usk6666/yorishiro-proxy/blob/main/LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-blue.svg" alt="License"></a>
</p>

---

yorishiro-proxy runs as an [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) server, giving AI agents full control over proxy operations through eleven MCP tools. Designed for use with Claude Code and other MCP-compatible agents, it enables automated security testing workflows without manual UI interaction. An embedded Web UI is also available for visual inspection and interactive use.

!!! warning "Beta"
    APIs and configuration may change between minor versions.

## Key features

- **Traffic interception & recording** — MITM proxy with automatic CA certificate management
- **Resender** — Replay requests with header/body/URL overrides, JSON patches, and raw HTTP editing
- **Fuzzer** — Automated payload injection with sequential/parallel modes and async execution
- **Macro** — Multi-step request sequences with variable extraction and template substitution
- **Intercept** — Hold and inspect requests/responses in real time, then release, modify, or drop
- **Auto-transform** — Automatic request/response modification rules for matching traffic
- **Target scope** — Two-layer security boundary (Policy + Agent) to restrict reachable hosts
- **Multi-protocol** — HTTP/1.x, HTTPS (MITM), HTTP/2 (h2c/h2), gRPC, WebSocket, Raw TCP, SOCKS5
- **AI safety** — SafetyFilter blocks destructive payloads and masks PII; rate limiting and diagnostic budgets
- **Plugin system** — Extend proxy behavior with [Starlark](https://github.com/google/starlark-go) scripts
- **Web UI** — Embedded React/Vite dashboard for visual inspection and interactive testing

## Quick start

### 1. Get the binary

Download a prebuilt binary from the [GitHub Releases](https://github.com/usk6666/yorishiro-proxy/releases) page, or build from source:

```bash
git clone https://github.com/usk6666/yorishiro-proxy.git
cd yorishiro-proxy
make build    # outputs bin/yorishiro-proxy
```

### 2. Configure MCP

Add to your MCP client configuration (e.g., `.mcp.json` for Claude Code):

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": []
    }
  }
}
```

The proxy starts as an MCP server on stdin/stdout. The CA certificate is automatically generated on first run and persisted to `~/.yorishiro-proxy/ca/`.

To also enable the Web UI, add the `-mcp-http-addr` flag:

```json
{
  "mcpServers": {
    "yorishiro-proxy": {
      "command": "/path/to/bin/yorishiro-proxy",
      "args": ["-mcp-http-addr", "127.0.0.1:3000"]
    }
  }
}
```

### 3. First capture

Once the MCP server is running, the AI agent can start capturing traffic:

```
1. Start the proxy       → proxy_start with listen_addr "127.0.0.1:8080"
2. Set HTTP_PROXY        → point your target application at the proxy
3. Install the CA cert   → query ca_cert to get the certificate path
4. Browse / send traffic → captured flows appear in query flows
5. Inspect & replay      → use resend to replay with modifications
```

For detailed setup instructions, see the [Getting started](getting-started/installation.md) guide.

## MCP tools

All proxy operations are exposed through eleven MCP tools:

| Tool | Purpose |
|------|---------|
| [`proxy_start`](tools/proxy-start.md) | Start a proxy listener with capture scope, TLS passthrough, intercept rules, auto-transform, TCP forwarding, and protocol settings |
| [`proxy_stop`](tools/proxy-stop.md) | Graceful shutdown of one or all listeners |
| [`configure`](tools/configure.md) | Runtime configuration changes (upstream proxy, capture scope, TLS passthrough, intercept rules, auto-transform, connection limits) |
| [`query`](tools/query.md) | Unified information retrieval: flows, flow details, messages, proxy status, config, CA certificate, intercept queue, macros, fuzz jobs/results |
| [`resend`](tools/resend.md) | Replay recorded requests with mutations and compare two flows structurally |
| [`fuzz`](tools/fuzz.md) | Execute fuzz testing campaigns with payload sets, positions, concurrency control, and stop conditions |
| [`macro`](tools/macro.md) | Define and execute multi-step macro workflows with variable extraction, guards, and hooks |
| [`intercept`](tools/intercept.md) | Act on intercepted requests: release, modify and forward, or drop |
| [`manage`](tools/manage.md) | Manage flow data (delete/export/import) and CA certificate regeneration |
| [`security`](tools/security.md) | Configure target scope rules, rate limits, diagnostic budgets, and SafetyFilter (Policy Layer + Agent Layer) |
| [`plugin`](tools/plugin.md) | List, reload, enable, and disable Starlark plugins at runtime |

See the [MCP tools overview](tools/overview.md) for details.

## Supported protocols

| Protocol | Detection | Notes |
|----------|-----------|-------|
| HTTP/1.x | Automatic | Forward proxy mode |
| HTTPS | CONNECT | MITM with dynamic certificate issuance |
| HTTP/2 | h2c / ALPN | Both cleartext and TLS, with per-stream flow display |
| gRPC | HTTP/2 content-type | Service/method extraction, streaming support, structured metadata display |
| WebSocket | HTTP Upgrade | Message-level recording with per-message display |
| Raw TCP | Fallback | Captures any unrecognized protocol, with TCP forwarding mappings |

See the [Protocols](protocols/http.md) section for detailed documentation on each protocol.

## Web UI

When Streamable HTTP mode is enabled (`-mcp-http-addr`), the embedded Web UI is served at the same address.

| Page | Description |
|------|-------------|
| [Flows](webui/flows.md) | Flow list with filtering by protocol, method, status code, and URL pattern |
| [Dashboard](webui/dashboard.md) | Flow statistics overview with real-time traffic summary |
| [Intercept](webui/intercept.md) | Real-time request/response interception with inline editing |
| [Resender](webui/resender.md) | Replay requests with overrides, JSON patches, raw HTTP editing, and dry-run preview |
| [Fuzzer](webui/fuzzer.md) | Create and manage fuzz campaigns with payload sets and result analysis |
| [Macros](webui/macros.md) | Multi-step request workflows with variable extraction |
| [Security](webui/security.md) | Target scope configuration (Policy + Agent Layer) with URL testing |
| [Settings](webui/settings.md) | Proxy control, TLS passthrough, auto-transform rules, CA management, and more |

The Web UI communicates with the backend via Streamable HTTP MCP — the same protocol used by AI agents. See the [Web UI overview](webui/overview.md) for more information.

## Quick links

| Section | Description |
|---------|-------------|
| [Getting started](getting-started/installation.md) | Installation, MCP configuration, CA certificate setup, and first capture |
| [Concepts](concepts/architecture.md) | Architecture, flows, MCP-first design, and security model |
| [MCP tools](tools/overview.md) | Complete reference for all eleven MCP tools |
| [Features](features/resender.md) | Detailed guides for resender, fuzzer, macros, intercept, and more |
| [Protocols](protocols/http.md) | HTTP, HTTPS MITM, HTTP/2, gRPC, WebSocket, Raw TCP, SOCKS5 |
| [Web UI](webui/overview.md) | Visual dashboard for inspection and interactive testing |
| [Plugins](plugins/overview.md) | Extend proxy behavior with Starlark scripts |
| [Configuration](configuration/cli-flags.md) | CLI flags, config files, TLS, upstream proxy, and retention settings |
| [Guides](guides/vulnerability-assessment.md) | Practical tutorials for vulnerability assessment, API testing, and multi-agent setups |
