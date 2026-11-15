# Local LLM Web Search

A local AI assistant that combines **Ollama**, **Qwen**, **Open WebUI**, **SearXNG**, and the **Model Context Protocol (MCP)** so a locally hosted language model can search the web and summarize results.

This repository documents the build, configuration, verification, startup, shutdown, and security practices for the lab project.

## Project goals

- Run a local language model with Ollama.
- Provide a browser-based chat interface with Open WebUI.
- Provide web search through a self-hosted SearXNG instance.
- Expose SearXNG to Open WebUI as an MCP Streamable HTTP tool.
- Keep service ports bound to localhost where possible.
- Keep credentials and machine-specific configuration out of source control.

## Architecture

See [ARCHITECTURE.md](ARCHITECTURE.md) for the component diagram and network flow.

## Components

| Component | Purpose | Local endpoint |
|---|---|---|
| Ollama | Runs the local language model | `http://localhost:11434` on the Windows host |
| Qwen 3.5:9b | Language model used for the integration test | Selected in Open WebUI |
| Open WebUI | Chat interface and MCP tool client | `http://localhost:3000` |
| SearXNG | Metasearch service | `http://localhost:8888` |
| SearXNG MCP server | Makes SearXNG search available as an MCP tool | `http://localhost:8889/mcp` |
| Docker Engine | Runs Open WebUI, SearXNG, and the MCP server | WSL2 Ubuntu |

The Windows host runs Ollama natively. Docker Engine runs inside WSL2 Ubuntu. Open WebUI and the SearXNG/MCP containers communicate over a shared Docker network.

## Lab environment

The completed test environment used:

- Windows 11
- WSL2 with Ubuntu
- Docker Engine and Docker Compose installed inside WSL2
- Ollama installed on Windows
- Open WebUI in Docker
- SearXNG and its MCP server in Docker
- Qwen 3.5:9b

Versions and hardware can differ on another machine. The project does not require the same CPU, GPU, memory, or exact WSL2 network address.

## How the project was built

### 1. Prepare Windows and WSL2

1. Enable WSL2 and install an Ubuntu distribution.
2. Update Ubuntu packages.
3. Install Docker Engine and the Docker Compose plugin inside Ubuntu, following the official Docker Engine instructions for the Ubuntu release in use.
4. Verify Docker from the Ubuntu terminal:

```bash
docker --version
docker compose version
sudo docker run hello-world
```

Docker Desktop was not used in this setup; Docker Engine runs directly inside WSL2.

### 2. Install Ollama on Windows

1. Install Ollama for Windows from [ollama.com](https://ollama.com/).
2. Download the model used for this test:

```powershell
ollama pull qwen3.5:9b
```

3. Confirm the model is available:

```powershell
ollama list
```

4. Verify the local Ollama API:

```powershell
Invoke-RestMethod http://localhost:11434/api/version
```

### 3. Allow the WSL2 environment to reach Ollama

Ollama is running on Windows, while Open WebUI is running inside Docker in WSL2. The container therefore needs a route to the Windows host.

The working setup configured Ollama to listen beyond Windows loopback and added a Windows Firewall inbound rule scoped to the WSL2 virtual network. Do not copy a machine-specific IP address or subnet from another computer.

Set the Windows user environment variable from PowerShell:

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_HOST", "0.0.0.0:11434", "User")
```

Restart Ollama after changing the variable.

Determine the current Windows host gateway from the Ubuntu/WSL2 terminal:

```bash
ip route | awk '/default/ {print $3; exit}'
```

Use the returned gateway address as the host portion of the Ollama URL for containers. The Windows Firewall rule should permit TCP port 11434 only from the actual WSL2 subnet used by that machine. Determine the current subnet rather than reusing another system's subnet.

**Security note:** Listening on `0.0.0.0` makes the Ollama API listen on available interfaces. Restrict access with Windows Firewall, do not create a router port-forward, and do not expose the API to the public internet or an untrusted LAN. If your environment supports a narrower bind address, prefer it.

### 4. Run Open WebUI

The container was created with a persistent Docker volume and a randomly generated secret key. The following is a template; replace `<WINDOWS_HOST_GATEWAY>` with the gateway discovered from WSL2. Generate a fresh secret locally. Never commit the generated value.

```bash
docker run -d \
  -p 127.0.0.1:3000:8080 \
  -v open-webui:/app/backend/data \
  -e WEBUI_SECRET_KEY="$(openssl rand -hex 32)" \
  -e OLLAMA_BASE_URL="http://<WINDOWS_HOST_GATEWAY>:11434" \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Open the interface at [http://localhost:3000](http://localhost:3000) and create the initial administrator account.

The `open-webui` Docker volume persists application data across container restarts and recreation. Protect that volume because it may contain chat history, account information, and application configuration.

### 5. Deploy SearXNG and its MCP server

The project used the upstream [searxng-mcp repository](https://github.com/shoutsid/searxng-mcp).

Clone it inside WSL2 Ubuntu:

```bash
cd ~
git clone https://github.com/shoutsid/searxng-mcp.git
cd ~/searxng-mcp
```

Review the upstream compose file, SearXNG settings, and startup script before running them. The startup script creates a local `.env` file containing a generated SearXNG secret.

Start SearXNG:

```bash
./start.sh
```

Test the JSON search endpoint:

```bash
curl 'http://localhost:8888/search?q=capital+of+France&format=json' | jq '.results[:3]'
```

Start the MCP server using the HTTP transport:

```bash
./start.sh --http
```

The MCP endpoint is available locally at `http://127.0.0.1:8889/mcp`. The health endpoint is `http://localhost:8889/health`.

### 6. Connect the containers over a Docker network

Open WebUI and the MCP server must be able to resolve and reach one another. In this setup, the SearXNG Compose project created a network named `searxng-mcp_default`.

Connect the existing Open WebUI container to that network:

```bash
docker network connect searxng-mcp_default open-webui
```

If Docker reports that the endpoint is already connected, the network connection is already present.

Test the MCP health endpoint from inside Open WebUI:

```bash
docker exec open-webui python3 -c "import urllib.request; print(urllib.request.urlopen('http://searxng-mcp:8080/health', timeout=5).read().decode())"
```

A healthy response should report the MCP server status and HTTP transport.

### 7. Configure the MCP connection in Open WebUI

In Open WebUI:

1. Open **Admin Panel**.
2. Go to **Settings → Integrations → External Tool Servers**.
3. Add a connection with these values:

| Setting | Value |
|---|---|
| Type | MCP Streamable HTTP |
| Name | SearXNG |
| Connection ID | `searxng` |
| URL | `http://searxng-mcp:8080/mcp` |
| Authentication | None |

4. Save the connection.
5. Refresh Open WebUI.
6. Confirm the SearXNG search tool is listed and available to the model.

The MCP URL uses the Docker service/container DNS name because Open WebUI and the MCP server share a Docker network. It is not the browser-facing localhost URL.

## Starting the project after a clean reboot

Use this sequence after restarting Windows.

### Step 1 — Start Ollama on Windows

Ollama normally starts with Windows if its startup option is enabled. If it is not running, launch Ollama from the Windows Start menu.

Verify from **Windows PowerShell**:

```powershell
Invoke-RestMethod http://localhost:11434/api/version
ollama list
```

### Step 2 — Start Docker Engine in WSL2

Open **Ubuntu (WSL2)**:

```bash
sudo service docker start
docker ps
```

If Docker requires elevated permissions on your installation, use `sudo docker ps`.

### Step 3 — Start SearXNG and MCP

```bash
cd ~/searxng-mcp
./start.sh --http
```

The script starts the SearXNG service and the MCP HTTP service.

### Step 4 — Check Open WebUI

The Open WebUI container was created with `--restart always`, so it should start when the Docker daemon starts. Check:

```bash
docker ps
```

If it is not running:

```bash
docker start open-webui
```

### Step 5 — Verify the services

From **Ubuntu (WSL2)**:

```bash
curl http://localhost:8889/health
curl -I http://localhost:8888
docker ps
```

From inside the Open WebUI container, verify it can reach MCP:

```bash
docker exec open-webui python3 -c "import urllib.request; print(urllib.request.urlopen('http://searxng-mcp:8080/health', timeout=5).read().decode())"
```

### Step 6 — Open the chat interface

Open [http://localhost:3000](http://localhost:3000), select **Qwen 3.5:9b**, and try:

> Search the web for the latest Intel Granite Rapids Xeon news. Summarize the three most relevant results and include their links.

If the tool is available, Open WebUI should show that the model explored the SearXNG search tool before composing its answer.

## Stopping the project

From **Ubuntu (WSL2)**:

```bash
cd ~/searxng-mcp
./stop.sh
docker stop open-webui
```

To stop the Docker Engine as well:

```bash
sudo service docker stop
```

Stopping containers does not delete the persistent Open WebUI volume. Do not remove Docker volumes unless you intentionally want to delete the stored application data.

## Successful end-to-end test

The completed integration was tested using Qwen 3.5:9b in Open WebUI.

**Test prompt:**

> Search the web for the latest Intel Granite Rapids Xeon news. Summarize the three most relevant results and include their links.

**Observed tool execution:**

`Explored 3 auto_searxng_web_search`

This demonstrates that Qwen invoked the SearXNG search tool through the MCP connection and then summarized the returned results.

### Example results returned during the test

1. **ServeTheHome — Intel Xeon 658X Review: Granite Rapids For Workstations**  
   https://www.servethehome.com/intel-xeon-658x-review-granite-rapids-for-workstations/  
   Review of the Xeon 658X workstation processor and Granite Rapids architecture, including chip configurations, chiplet design, benchmarks, and performance.

2. **Intel Newsroom — Intel Launches New Intel Xeon 600 Processors for Workstation**  
   https://www.intel.com/content/www/us/en/newsroom/news/client-computing/intel-launches-new-intel-xeon-600-processors-for-workstation.html  
   Intel's announcement of the Xeon 600 workstation family and the W890 platform, including processor, memory, and PCIe platform updates.

3. **Tom's Hardware — Intel Returns to Boxed Workstation CPUs with Xeon 600**  
   https://www.tomshardware.com/pc-components/cpus/intel-returns-to-boxed-workstation-cpus-with-xeon-600-granite-rapids-ws-delivers-up-to-86-cores-4tb-of-memory-and-128-pcie-5-lanes  
   Article about Granite Rapids-WS and the Xeon 600 workstation lineup, including configurations described with up to 86 cores, 4TB of memory, and 128 PCIe 5.0 lanes.

*These are the example results returned during the project test. They are documented as an observed tool-use example, not as independently re-verified reporting.*

## Troubleshooting

### Open WebUI cannot connect to Ollama

- Confirm Ollama is running on Windows.
- Test `http://localhost:11434/api/version` from Windows PowerShell.
- Confirm the WSL2 gateway used in `OLLAMA_BASE_URL` is current.
- Confirm Windows Firewall permits the WSL2 subnet to reach TCP port 11434.
- Restart Ollama after changing `OLLAMA_HOST`.

### Open WebUI does not show the SearXNG tool

- Confirm both `searxng` and `searxng-mcp` containers are running.
- Confirm Open WebUI is connected to `searxng-mcp_default`.
- Verify the MCP health endpoint from inside the Open WebUI container.
- Recheck the MCP connection URL: `http://searxng-mcp:8080/mcp`.
- Refresh Open WebUI after saving the connection.
- Inspect logs with `docker logs --tail 80 open-webui` and `docker logs --tail 80 searxng-mcp`.

### SearXNG returns few results or errors

SearXNG relies on external search engines. Individual engines may throttle, block, or change their response behavior. A search failure from one engine does not necessarily mean the MCP connection is broken.

## Security and repository hygiene

This repository is intended to contain documentation and safe examples, not live secrets or private runtime data.

- Never commit `.env`, passwords, API keys, bearer tokens, session cookies, private keys, or generated secret values.
- Never commit Open WebUI's data volume, database, chat history, or user account data.
- Never commit local logs, database files, Docker volume exports, or personal backups.
- Do not publish machine-specific IP addresses, private network ranges, Windows usernames, home-directory paths, or firewall rule scopes.
- Keep the Ollama, Open WebUI, SearXNG, and MCP endpoints bound to localhost or private container networks where possible.
- Do not expose Ollama's API, Open WebUI administration, SearXNG, or the MCP endpoint directly to the public internet.
- Use a newly generated secret for each installation. Do not reuse a secret from a tutorial or another machine.
- Review upstream Docker images and scripts before running them. Pin image versions or digests for a more reproducible deployment.
- A private repository is not a secrets manager. Do not store credentials in GitHub even if the repository is private.

See [SECURITY.md](SECURITY.md) for the reporting and disclosure guidance.

## Project source

The SearXNG MCP server used for this lab is maintained separately:
https://github.com/shoutsid/searxng-mcp

This repository records the local integration and learning process; it does not claim ownership of the upstream components.
