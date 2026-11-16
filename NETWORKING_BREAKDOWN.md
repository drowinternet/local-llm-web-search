# Networking Breakdown

This document explains how traffic moves between Windows, WSL2, Docker, Open WebUI, Ollama, the SearXNG MCP server, and SearXNG in the current local setup.

The important distinction is that we use **two different networking paths**:

1. **Host-to-container / container-to-host traffic** to reach Ollama, which runs natively on Windows.
2. **Container-to-container traffic** for Open WebUI, the MCP server, and SearXNG, which run in Docker.

## 1. High-level network diagram

```text
                         WINDOWS 11 HOST
  +------------------------------------------------------------+
  |                                                            |
  |  Ollama (native Windows process)                            |
  |  Listening on TCP 11434                                     |
  |          ^                                                 |
  |          |                                                 |
  +----------|-------------------------------------------------+
             |
             | WSL2 virtual network
             | Windows host gateway: 172.23.80.1
             | TCP 11434
             v
  +------------------------------------------------------------+
  |                  WSL2 Ubuntu / Docker                       |
  |                                                            |
  |  Browser                                                     |
  |    |                                                       |
  |    | http://localhost:3000                                 |
  |    v                                                       |
  |  +-------------------+                                     |
  |  | Open WebUI         |                                     |
  |  | container :8080    |                                     |
  |  +-------------------+                                     |
  |    |                         |                              |
  |    | Ollama API              | MCP Streamable HTTP          |
  |    | 172.23.80.1:11434       | http://searxng-mcp:8080/mcp  |
  |    |                         v                              |
  |    |                 +-------------------+                  |
  |    |                 | SearXNG MCP        |                  |
  |    |                 | container :8080    |                  |
  |    |                 +-------------------+                  |
  |    |                         |                              |
  |    |                         | http://core:8080             |
  |    |                         v                              |
  |    |                 +-------------------+                  |
  |    |                 | SearXNG            |                  |
  |    |                 | container :8080    |                  |
  |    |                 +-------------------+                  |
  |    |                         |                              |
  |    |                         | External search providers    |
  |    |                         v                              |
  |    |                  Internet / search engines             |
  |                                                            |
  +------------------------------------------------------------+
```

The diagram shows the logical traffic paths. The browser reaches Open WebUI through a published localhost port; the containers use Docker networking to reach one another; and Open WebUI reaches Ollama through the WSL2-to-Windows host gateway.

## 2. Ollama: Windows host to Docker container

Ollama is **not running in Docker** in this setup. It runs as a native Windows application and listens on TCP port `11434`.

Open WebUI runs inside a Docker container. From inside that container, `localhost` means the Open WebUI container itself—not Windows and not the WSL2 Ubuntu environment.

We therefore configured Open WebUI with:

```text
OLLAMA_BASE_URL=http://172.23.80.1:11434
```

Here, `172.23.80.1` is the Windows host's gateway address as seen from the WSL2 virtual network. It is **not a Docker bridge port**. The traffic path is:

```text
Open WebUI container
    |
    | HTTP to 172.23.80.1:11434
    v
WSL2 virtual network / Windows host gateway
    |
    v
Ollama running on Windows, TCP 11434
```

We also set Ollama to listen beyond Windows loopback and added a Windows Firewall inbound rule intended to allow the WSL2 subnet:

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_HOST", "0.0.0.0:11434", "User")

New-NetFirewallRule -DisplayName "Ollama API - WSL2 Only" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 11434 -RemoteAddress 172.23.80.0/20 -Profile Any
```

The firewall rule is scoped to the WSL2 subnet, but other firewall rules or network changes can affect effective access. Do not expose the Ollama API directly to the public internet. The WSL2 gateway address can change after networking changes, so verify it if Open WebUI can no longer reach Ollama.

## 3. Published ports: browser and host access

Docker port publishing uses this format:

```text
-p HOST_IP:HOST_PORT:CONTAINER_PORT
```

For example, Open WebUI is published with:

```text
127.0.0.1:3000 -> container port 8080
```

The host-side binding to `127.0.0.1` means the service is published on the local machine's loopback interface, rather than on every network interface.

The current published ports are:

| Service | Host binding | Container port | Purpose |
|---|---|---:|---|
| Open WebUI | `127.0.0.1:3000` | `8080` | Browser access to the chat UI |
| SearXNG | `127.0.0.1:8888` | `8080` | Local search testing and diagnostics |
| SearXNG MCP | `127.0.0.1:8889` | `8080` | Local MCP health checks and diagnostics |

The browser opens:

```text
http://localhost:3000
```

The published SearXNG and MCP host ports are useful for local testing. However, **Open WebUI does not need to use those published host ports to communicate with the MCP server**. It uses Docker's internal network instead.

## 4. Container-to-container routing

Docker provides an internal network for containers to communicate. On that network, containers can resolve one another by Docker service or container name and connect to the destination container port.

Initially, Open WebUI and the SearXNG stack were on separate Docker networks. Open WebUI was on the default `bridge` network, while the SearXNG Compose project created `searxng-mcp_default`.

We connected Open WebUI to the SearXNG network:

```bash
docker network connect searxng-mcp_default open-webui
```

Open WebUI now has access to both networks. The shared network allows it to resolve and reach the MCP container by name.

We verified connectivity from inside the Open WebUI container:

```bash
docker exec open-webui python3 -c "import urllib.request; print(urllib.request.urlopen('http://searxng-mcp:8080/health', timeout=5).read().decode())"
```

The response reported a healthy MCP server using HTTP transport. This confirmed that Docker DNS resolution and container-to-container connectivity were working.

### Open WebUI to the MCP server

The Open WebUI external tool server connection uses:

| Setting | Value |
|---|---|
| Type | MCP Streamable HTTP |
| Name | SearXNG |
| Connection ID | `searxng` |
| URL | `http://searxng-mcp:8080/mcp` |
| Authentication | None |

The hostname `searxng-mcp` resolves to the MCP container on the shared Docker network. Port `8080` is the MCP container's internal listening port. The `/mcp` path is the MCP endpoint.

### MCP server to SearXNG

The MCP server itself reaches SearXNG at:

```text
http://core:8080
```

Here, `core` is the SearXNG service name in the Docker Compose project. Docker's internal DNS resolves that service name on the Compose network. The MCP server does not need to route through the host's published port `8888`.

This gives us two separate internal hops:

```text
Open WebUI container
    |
    | http://searxng-mcp:8080/mcp
    v
SearXNG MCP container
    |
    | http://core:8080
    v
SearXNG container
```

## 5. What each address and port means

| Address | Where it is used | Meaning |
|---|---|---|
| `localhost:3000` | Windows browser | Published Open WebUI port |
| `127.0.0.1:8888` | Windows/WSL host | Published SearXNG port for local access |
| `127.0.0.1:8889` | Windows/WSL host | Published MCP port for local access and diagnostics |
| `172.23.80.1:11434` | Open WebUI container | Windows host gateway as seen from WSL2, reaching native Ollama |
| `searxng-mcp:8080` | Open WebUI container | MCP container's internal Docker-network address |
| `core:8080` | MCP container | SearXNG service's internal Docker-network address |

A useful rule: **localhost is relative to the machine or container where the command or application is running.** Inside Open WebUI, `localhost` points back to Open WebUI itself. It does not automatically mean the Windows host or another container.

## 6. Full search request flow

When a user asks Qwen to search the web, the request follows this sequence:

1. The browser sends the chat message to Open WebUI at `http://localhost:3000`.
2. Open WebUI sends the prompt to Ollama at `http://172.23.80.1:11434`.
3. Ollama runs the Qwen model. The model decides that the available web-search tool is appropriate.
4. Open WebUI calls the MCP tool through `http://searxng-mcp:8080/mcp`.
5. The SearXNG MCP server receives the tool request and queries SearXNG at `http://core:8080`.
6. SearXNG searches its configured external search providers.
7. Search results return through SearXNG, the MCP server, and Open WebUI.
8. Open WebUI provides the results to the model, and Qwen generates the response shown in the chat.

In simplified form:

```text
Browser
  -> Open WebUI
  -> Ollama / Qwen (tool selection)
  -> SearXNG MCP server
  -> SearXNG
  -> External search engines

Results return to Open WebUI / Qwen
  -> Final answer in browser
```

## 7. Why we used both published ports and Docker networking

These mechanisms solve different problems:

- **Published ports** make a container service reachable from the host or browser. For example, the browser uses `localhost:3000` to reach Open WebUI.
- **Docker internal networking** lets containers communicate directly using service/container names and internal ports. For example, Open WebUI uses `searxng-mcp:8080`, and the MCP server uses `core:8080`.
- **The WSL2 host gateway** lets a containerized application reach a service running natively on Windows. We use `172.23.80.1:11434` for Ollama.

We did not change Ollama to a Docker bridge port. We configured Ollama to be reachable through the Windows host's WSL2 gateway address. Separately, we connected Open WebUI to the Docker network used by the SearXNG MCP stack.

## 8. Useful checks when troubleshooting

Run these commands in **WSL2 Ubuntu** unless otherwise noted.

Check the containers and their published ports:

```bash
docker ps
```

Check which networks Open WebUI is attached to:

```bash
docker inspect open-webui --format '{{json .NetworkSettings.Networks}}' | jq
```

Check the MCP server health from inside Open WebUI:

```bash
docker exec open-webui python3 -c "import urllib.request; print(urllib.request.urlopen('http://searxng-mcp:8080/health', timeout=5).read().decode())"
```

Check SearXNG health from WSL2:

```bash
curl http://localhost:8888/healthz
```

Check the MCP server's published health endpoint from WSL2:

```bash
curl http://localhost:8889/health
```

Check Ollama's API from **Windows PowerShell**:

```powershell
Invoke-RestMethod http://localhost:11434/api/version
```

If Ollama works from Windows but not from Open WebUI, check the WSL2 gateway address, Ollama's listen configuration, Windows Firewall rules, and the `OLLAMA_BASE_URL` configured for Open WebUI.

---

## Current architecture note

This is a working local development setup, not a claim that the deployment is production-hardened. The current MCP connection uses no authentication and is intended to remain on the local Docker network. Keep the host-published ports bound to loopback unless there is a deliberate, secured reason to expose them. For production considerations, see [Production Readiness](PRODUCTION_READINESS.md).
