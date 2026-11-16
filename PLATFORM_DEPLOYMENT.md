# Platform Deployment Guide

This guide deploys the **current project as it exists today** on Linux, Windows 11, or macOS. It does not add an MCP gateway, multiple inference engines, or a new automated installer.

The current stack is:

- **Ollama** runs natively on the host operating system.
- **Open WebUI** runs in Docker.
- **SearXNG** runs in Docker.
- **The SearXNG MCP server** runs in Docker using Streamable HTTP.
- **Qwen 3.5:9b** is the model used in the documented integration test. Other Ollama models may work if they support tool calling.

The platform-specific difference is how the Open WebUI container reaches Ollama on the host. The SearXNG/MCP deployment and Open WebUI-to-MCP Docker networking are otherwise the same.

> **Status:** These are deployment instructions for the existing prototype. They are not a one-command installer, and the project has not been tested end-to-end on every Linux distribution, Windows configuration, or Mac model. Follow the official installation instructions for your OS and verify each stage.

## 1. Architecture and ports

| Component | Runs where | Host endpoint / port |
|---|---|---|
| Ollama | Native host application | TCP 11434 |
| Open WebUI | Docker container | `http://localhost:3000` |
| SearXNG | Docker container | `http://localhost:8888` |
| SearXNG MCP | Docker container | `http://localhost:8889/mcp` |

The Docker containers listen internally on port 8080. Docker publishes host ports 3000, 8888, and 8889 for local access. Open WebUI calls the MCP server over the shared Docker network at `http://searxng-mcp:8080/mcp`; the MCP server calls SearXNG at `http://core:8080`. The published ports 8888 and 8889 are for local testing/diagnostics, not the normal container-to-container search path.

## 2. Prerequisites for all platforms

Install:

1. **Git**
2. **Docker with Docker Compose v2**
3. **Ollama** for your operating system: [https://ollama.com/download](https://ollama.com/download)
4. An Ollama model. The documented test uses:

```bash
ollama pull qwen3.5:9b
ollama list
```

5. On Linux and macOS, install `curl` and `openssl` if they are not already present. The upstream SearXNG startup script uses them.

Official container installation references:

- Docker Engine on Linux: [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)
- Docker Desktop for Mac: [https://docs.docker.com/desktop/setup/install/mac-install/](https://docs.docker.com/desktop/setup/install/mac-install/)
- Docker Desktop for Windows: [https://docs.docker.com/desktop/setup/install/windows-install/](https://docs.docker.com/desktop/setup/install/windows-install/)
- Docker Compose: [https://docs.docker.com/compose/](https://docs.docker.com/compose/)

### Important: keep Ollama's API private

Open WebUI needs to reach Ollama's API from its container. In the platform-specific sections below, Ollama is configured to listen on an address reachable from Docker. Do not forward TCP 11434 on your router or expose it to the public internet. Restrict host-firewall access to the local Docker/WSL environment wherever possible.

## 3. Common deployment: SearXNG and MCP

Run these commands in a terminal with Docker Compose available. On Windows, use the Ubuntu terminal inside WSL2 (not PowerShell) for this section.

Clone the upstream SearXNG/MCP project:

```bash
git clone https://github.com/shoutsid/searxng-mcp.git
cd searxng-mcp
```

Review the upstream compose file and configuration before starting containers. The upstream startup script generates a local `.env` file containing the SearXNG secret and restricts its permissions.

Start SearXNG and its HTTP MCP server:

```bash
chmod +x start.sh stop.sh
./start.sh --http
```

The script starts SearXNG and then starts the MCP service using the `mcp-http` Compose profile.

Verify the services:

```bash
curl http://localhost:8888/healthz
curl http://localhost:8889/health
curl 'http://localhost:8888/search?q=capital+of+France&format=json' | head
docker compose ps
```

Expected local endpoints:

- SearXNG: `http://localhost:8888`
- MCP Streamable HTTP: `http://localhost:8889/mcp`
- MCP health: `http://localhost:8889/health`

The upstream Compose project creates a Docker network named `searxng-mcp_default`. Keep the project directory and its generated `.env`, `core-config`, and `data` directory; they are part of the local deployment state. Do not commit the generated `.env` file.

## 4. Deploy Open WebUI

Open WebUI runs as a separate Docker container in the current project. Set `OLLAMA_URL` to the host address that is reachable **from inside a Docker container**, using the relevant platform section below.

Generate a fresh Open WebUI secret and create its persistent volume:

```bash
export WEBUI_SECRET_KEY="$(openssl rand -hex 32)"
docker volume create open-webui
```

Then run Open WebUI. Replace `<OLLAMA_URL>` with the address from the platform-specific instructions:

```bash
docker run -d \
  -p 127.0.0.1:3000:8080 \
  -v open-webui:/app/backend/data \
  -e WEBUI_SECRET_KEY="$WEBUI_SECRET_KEY" \
  -e OLLAMA_BASE_URL="<OLLAMA_URL>" \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

The `open-webui` volume stores application data, including account and chat information. Keep it when upgrading or recreating the container. The `main` image tag is mutable; for a more controlled deployment, select and pin a tested image version or digest rather than relying indefinitely on `main`.

## 5. Platform-specific Ollama connectivity

### A. Linux

**Recommended for:** a Linux workstation or server where Ollama and Docker run on the same Linux host.

1. Install Docker Engine and the Compose plugin using the official instructions for your distribution.
2. Install and start Ollama natively. Pull the model as shown in the prerequisites.
3. Configure Ollama to listen on an address reachable from Docker. For a systemd-based installation, create an override:

```bash
sudo systemctl edit ollama
```

Add:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

Then apply it:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
curl http://localhost:11434/api/version
```

If Ollama was installed using a different service manager or desktop package, configure its `OLLAMA_HOST` using that installation's documented method instead.

4. When creating Open WebUI, add the Docker host-gateway mapping and use the special hostname:

```bash
-e OLLAMA_BASE_URL="http://host.docker.internal:11434" \
--add-host=host.docker.internal:host-gateway
```

These two options belong in the `docker run` command from section 4. The complete command should include both.

5. Restrict access to TCP 11434 using the Linux host firewall. Allow only the Docker network(s) that need to reach Ollama; do not create a broad LAN or public allow rule. Docker networking and firewall behavior vary by distribution and firewall backend, so verify the effective rules for your host.

### B. Windows 11 with WSL2

**Recommended for:** Windows users who want Ollama installed natively on Windows, with Docker Engine and the containers running in Ubuntu under WSL2. This is the arrangement used for the original project test.

1. Install WSL2 and an Ubuntu distribution: [https://learn.microsoft.com/windows/wsl/install](https://learn.microsoft.com/windows/wsl/install).
2. Install Docker Engine and the Docker Compose plugin **inside WSL2 Ubuntu**, following the official Linux instructions. The original test used Docker Engine directly in WSL2, not Docker Desktop.
3. Install Ollama for Windows and pull the model from **Windows PowerShell**:

```powershell
ollama pull qwen3.5:9b
ollama list
```

4. Configure Ollama to listen beyond Windows loopback. From PowerShell:

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_HOST", "0.0.0.0:11434", "User")
```

Restart Ollama after setting the variable.

5. Determine the Windows host gateway from **WSL2 Ubuntu**:

```bash
ip route | awk '/default/ {print $3; exit}'
```

Use the returned address in the Open WebUI container's `OLLAMA_BASE_URL`, for example:

```text
http://<WSL2_WINDOWS_HOST_GATEWAY>:11434
```

Do not copy a gateway IP from another computer; WSL2 addresses can change.

6. Add a Windows Firewall inbound rule for TCP 11434 scoped to the actual WSL2 subnet. Determine the current subnet on that machine and use it in the rule. Example only (the subnet below is not universal):

```powershell
New-NetFirewallRule -DisplayName "Ollama API - WSL2 Only" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 11434 -RemoteAddress <YOUR_WSL2_SUBNET> -Profile Any
```

Replace the placeholder with the actual subnet in CIDR notation. Review existing firewall rules as well; this rule alone does not prove that no other rule permits access.

7. Run the Open WebUI Docker command from section 4 in WSL2 Ubuntu, setting `OLLAMA_BASE_URL` to the discovered gateway address.

### C. macOS

**Recommended for:** Mac users who want Ollama installed natively (including Apple Silicon systems) and the application containers managed by Docker Desktop.

1. Install Docker Desktop for Mac and start it.
2. Install Ollama for macOS and pull the model:

```bash
ollama pull qwen3.5:9b
ollama list
```

3. Configure Ollama to accept connections from Docker Desktop. In a terminal:

```bash
launchctl setenv OLLAMA_HOST "0.0.0.0:11434"
```

Quit and restart the Ollama application so it inherits the environment setting. Verify the API on the Mac:

```bash
curl http://localhost:11434/api/version
```

4. In the Open WebUI Docker command from section 4, set:

```bash
-e OLLAMA_BASE_URL="http://host.docker.internal:11434"
```

Docker Desktop provides `host.docker.internal` so a container can reach a service on the Mac host. Keep macOS firewall settings restrictive and do not expose Ollama to untrusted networks.

## 6. Connect Open WebUI to the MCP Docker network

After Open WebUI is running, connect it to the network created by the upstream SearXNG Compose project. Run in the same Docker environment where both deployments are running:

```bash
docker network connect searxng-mcp_default open-webui
```

If Docker says the endpoint is already connected, that is fine.

Verify connectivity from inside Open WebUI:

```bash
docker exec open-webui python3 -c "import urllib.request; print(urllib.request.urlopen('http://searxng-mcp:8080/health', timeout=5).read().decode())"
```

A healthy response should identify the SearXNG MCP server and HTTP transport.

## 7. Register the MCP server in Open WebUI

Open [http://localhost:3000](http://localhost:3000) in a browser. Create the first administrator account if this is a fresh deployment.

In **Admin Panel → Settings → Integrations → External Tool Servers**, add:

| Setting | Value |
|---|---|
| Type | MCP Streamable HTTP |
| Name | SearXNG |
| Connection ID | `searxng` |
| URL | `http://searxng-mcp:8080/mcp` |
| Authentication | None |

Save and refresh Open WebUI. Confirm the SearXNG search tool appears for the model.

## 8. Verify end-to-end operation

Check the host Ollama API:

```bash
curl http://localhost:11434/api/version
ollama list
```

On Windows, run those checks from PowerShell. On Linux/macOS, run them from the host terminal.

Check the containers and local endpoints:

```bash
docker ps
curl -I http://localhost:3000
curl http://localhost:8888/healthz
curl http://localhost:8889/health
```

In Open WebUI, select `qwen3.5:9b` and try:

> Search the web for the latest Intel Granite Rapids Xeon news. Summarize the three most relevant results and include their links.

The model should invoke the SearXNG MCP search tool before writing its response.

## 9. Start, stop, and preserve data

Start the SearXNG/MCP services:

```bash
cd /path/to/searxng-mcp
./start.sh --http
```

Start Open WebUI if it is stopped:

```bash
docker start open-webui
```

Stop the SearXNG/MCP stack:

```bash
cd /path/to/searxng-mcp
./stop.sh
```

Stop Open WebUI without deleting its data:

```bash
docker stop open-webui
```

Persistent data to protect:

- Docker volume `open-webui`
- The SearXNG project directory, including its local `.env`, `core-config`, and `data`
- Any local deployment notes or configuration overrides

Do not run `docker compose down -v` or delete the `open-webui` volume unless you intentionally want to remove persistent application data. Back up secrets securely; do not upload them to GitHub.

## 10. Troubleshooting

### Open WebUI cannot reach Ollama

- Verify Ollama is running and `/api/version` responds on the host.
- Confirm the `OLLAMA_BASE_URL` uses an address reachable from inside the Open WebUI container, not the container's own `localhost`.
- On Windows, rediscover the WSL2 host gateway and check Windows Firewall.
- On Linux, confirm the Docker host-gateway mapping, Ollama bind address, and host firewall.
- On macOS, restart Ollama after setting `OLLAMA_HOST` and use `host.docker.internal`.

### Open WebUI cannot reach the MCP server

- Check that the Compose MCP service is running: `docker compose --profile mcp-http ps`.
- Check that Open WebUI is attached to `searxng-mcp_default`.
- Verify the MCP URL is `http://searxng-mcp:8080/mcp`, not the host-published `8889` URL.
- Run the health check from inside the Open WebUI container.

### Search fails

- Check `docker compose logs --tail 100 core mcp` from the upstream SearXNG project directory.
- Verify `http://localhost:8888/healthz`.
- Search providers can throttle, block, or change their response behavior; individual engine failures do not necessarily indicate a broken MCP connection.

## 11. Current limitations and future work

This guide deploys the existing direct Open WebUI-to-SearXNG-MCP architecture. It does **not** implement:

- An MCP gateway/aggregator
- PostgreSQL or infrastructure MCP integrations
- A multi-LLM abstraction layer or non-Ollama inference runtimes
- A single cross-platform one-command installer
- Production-grade authentication, authorization, monitoring, or backup automation

Those remain future project work. The current prototype should remain locally bound and should not be exposed to the public internet without additional security controls. See [PRODUCTION_READINESS.md](PRODUCTION_READINESS.md) for hardening guidance.
