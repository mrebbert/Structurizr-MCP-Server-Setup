# Structurizr + MCP Server Setup

> Self-hosted [Structurizr](https://structurizr.com/) (on-premises) with MCP Server integration for [Claude Desktop](https://claude.ai/download).

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [1. Build the Image](#1-build-the-image)
- [2. Create Directory & Configuration](#2-create-directory--configuration)
- [3. Docker Compose](#3-docker-compose)
- [4. Caddy Reverse Proxy (optional)](#4-caddy-reverse-proxy-optional)
- [5. Claude Desktop Configuration](#5-claude-desktop-configuration)
- [6. Claude Project System Prompt](#6-claude-project-system-prompt)
- [7. Available MCP Tools](#7-available-mcp-tools)
- [8. Typical Workflow](#8-typical-workflow)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Docker & Docker Compose
- Git, Java 21, Maven (for the build)
- Node.js & npx (for Claude Desktop)
- Caddy as reverse proxy (**optional**, see [Section 4](#4-caddy-reverse-proxy-optional))
- Two DNS CNAMEs (**optional**, only required when using the reverse proxy):
  - `structurizr.domain.lan` → Structurizr Web UI
  - `structurizr-mcp.domain.lan` → MCP Server

> **Without a reverse proxy**, CNAMEs are not required. Structurizr is accessible at `http://<host-ip>:9000` and the MCP Server at `http://<host-ip>:3000`. Use `http://<host-ip>:3000/mcp` with the `--allow-http` flag in the Claude Desktop config (see [Section 5](#5-claude-desktop-configuration)).

---

## 1. Build the Image

```bash
git clone https://github.com/structurizr/structurizr.git
cd structurizr
./mvnw -DexcludedGroups=IntegrationTest package
docker build . -t structurizr
```

---

## 2. Create Directory & Configuration

```bash
mkdir structurizr
```

Create `structurizr/structurizr.properties`:

```properties
structurizr.feature.ui.dslEditor=true
structurizr.url=https://structurizr.domain.lan
```

> Without reverse proxy: `structurizr.url=http://<host-ip>:9000`

---

## 3. Docker Compose

```yaml
services:
  structurizr:
    image: structurizr
    container_name: structurizr
    restart: unless-stopped
    ports:
      - "9000:8080"
    volumes:
      - "./structurizr:/usr/local/structurizr"
    command: server
    networks:
      - structurizr-net

  structurizr-mcp:
    image: structurizr
    container_name: structurizr-mcp
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - PORT=3000
    command: mcp -dsl -server
    networks:
      - structurizr-net

networks:
  structurizr-net:
    driver: bridge
```

```bash
docker compose up -d structurizr structurizr-mcp
```

> **Note:** Both containers share the Docker network `structurizr-net`. The MCP container reaches the Structurizr server internally via `http://structurizr:8080` — no TLS, no reverse proxy required for container-to-container communication.

---

## 4. Caddy Reverse Proxy (optional)

Only required when using the CNAMEs. Caddy provides HTTPS with an internal TLS certificate.

```caddy
# Structurizr MCP Server
structurizr-mcp.domain.lan {
    encode gzip zstd
    tls internal
    reverse_proxy 127.0.0.1:3000
}

# Structurizr Web UI
structurizr.domain.lan {
    encode gzip zstd
    tls internal
    reverse_proxy 127.0.0.1:9000
}
```

> `tls internal` generates a self-signed certificate via Caddy's internal CA. Browsers may need to trust this CA. Node.js (`mcp-remote`) requires `NODE_TLS_REJECT_UNAUTHORIZED=0`.

---

## 5. Claude Desktop Configuration

**Mac:** `~/Library/Application Support/Claude/claude_desktop_config.json`  
**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

### With reverse proxy (HTTPS)

```json
{
  "mcpServers": {
    "structurizr-mcp": {
      "command": "npx",
      "args": ["mcp-remote", "https://structurizr-mcp.domain.lan/mcp"],
      "env": {
        "NODE_TLS_REJECT_UNAUTHORIZED": "0"
      }
    }
  }
}
```

### Without reverse proxy (HTTP direct)

```json
{
  "mcpServers": {
    "structurizr-mcp": {
      "command": "npx",
      "args": ["mcp-remote", "http://<host-ip>:3000/mcp", "--allow-http"]
    }
  }
}
```

Restart Claude Desktop after making changes.

---

## 6. Claude Project System Prompt

The MCP Server has no configurable default server URL. To avoid specifying it on every prompt, create a **Project** in Claude Desktop and add the following as the system prompt under **Projects → System Prompt**:

```
You are an expert in Structurizr DSL and software architecture (C4 Model).

Server:
Always use: http://structurizr:8080

Rules (critical – follow strictly)

1. All style properties must be on separate lines – never inline.
   INVALID: { background #x color #y }
   VALID:
     {
       background #x
       color #y
     }

2. deploymentEnvironment must be defined inside model {},
   not inside views {}.

3. The deployment view references the environment name exactly
   as defined:
   deployment pim "Production" "DeploymentProduction"

4. External software systems use the 4th string parameter for tags:
   erp = softwareSystem "ERP System" "Description" "External"

5. Always validate the DSL before returning it.

6. Return only the DSL – no explanation, no markdown fences.
```

Every prompt within the project will automatically use the internal Docker hostname — no manual URL input required.

---

## 7. Available MCP Tools

| Tool | Description |
|---|---|
| `getWorkspace` | Load a single workspace from the server |
| `getWorkspaces` | Load all workspaces from the server |
| `validate` | Validate DSL |
| `parse` | Parse DSL → JSON workspace |
| `inspect` | Check DSL for violations |
| `export` (Mermaid/PlantUML) | Export a view as a diagram |

> **Note:** The MCP Server is currently **read-only**. Writing workspaces back to the server is not supported.

---

## 8. Typical Workflow

```
1. "Load workspace 1"
        ↓
2. Claude reads and analyses the architecture
        ↓
3. Claude modifies the DSL
        ↓
4. Claude validates the updated DSL
        ↓
5. Manually apply the DSL in Structurizr
```

---

## Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| `EHOSTUNREACH :443` | `mcp-remote` forces HTTPS | Use `--allow-http` flag or set up HTTPS |
| `UNABLE_TO_GET_ISSUER_CERT_LOCALLY` | Self-signed certificate unknown to Node.js | Set `NODE_TLS_REJECT_UNAUTHORIZED=0` |
| `tlsv1 alert internal error` | TLS handshake fails / no route to host | Check reverse proxy config and whether port is open |
| MCP→Structurizr SSL error | Caddy `tls internal` CA unknown to JVM | Use internal Docker hostname `http://structurizr:8080` instead of the CNAME |

---

## References

- [Structurizr Documentation](https://docs.structurizr.com/)
- [Structurizr MCP Server](https://docs.structurizr.com/ai/mcp)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [mcp-remote (npm)](https://www.npmjs.com/package/mcp-remote)
- [Caddy Documentation](https://caddyserver.com/docs/)
