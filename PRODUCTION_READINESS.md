# Production Readiness and Hardening Plan

## Status

The current deployment is a **working local prototype**, not a production-hardened service. The items below are a target-state checklist. They are recommendations and have not all been implemented or verified in the current environment.

The first target should be a reliable single-user/local deployment. A shared or organizational deployment requires additional identity, network, operational, and governance controls.

## 1. Deployment and supply-chain controls

- Replace floating container tags such as `latest` and `main` with explicitly tested release versions. For stronger reproducibility, pin image digests and record the update procedure.
- Review upstream Dockerfiles, Compose files, startup scripts, and configuration changes before deployment.
- Keep a documented upgrade and rollback process. Test upgrades against a backup before updating the working environment.
- Use supported, maintained versions of Open WebUI, Ollama, SearXNG, and the selected MCP server/gateway.
- Keep runtime services separate from source-control documentation. Do not commit container volumes, databases, logs, or local runtime state.

## 2. Network exposure

- Keep Open WebUI bound to loopback for a single-user local installation.
- Keep SearXNG and MCP host-published ports bound to loopback when host access is needed only for testing or administration.
- Prefer Docker-internal networking for service-to-service traffic. Do not publish MCP ports to the LAN or internet unless remote clients explicitly require access and the endpoint is protected.
- Restrict the Windows Ollama API firewall rule to the actual WSL2 source network or a narrower supported interface. Re-check the rule after WSL network changes.
- Do not configure router port forwarding for Ollama, Open WebUI administration, SearXNG, or MCP services.
- If remote browser access is required, use a VPN or a properly configured reverse proxy with HTTPS and authentication. Do not expose the development configuration directly to the public internet.

## 3. Authentication and authorization

- Retain Open WebUI's administrator account protections; use a unique, strong administrator credential and restrict who can administer the instance.
- Replace the current unauthenticated MCP connection with authenticated access when the MCP endpoint is reachable by anything beyond the tightly controlled local Docker network.
- Use separate credentials or identities for separate services. Store tokens in protected environment files or a secrets manager, never in source control.
- Apply least privilege to every backend tool. For example, a PostgreSQL integration should start with a dedicated read-only database role and narrowly scoped schema/table permissions.
- Allowlist only the tools needed by a given user or workflow. Do not expose destructive operations by default.
- For a shared deployment, define user/team roles, per-tool authorization, credential rotation, and an audit trail.

## 4. SearXNG and web-search controls

- Review SearXNG's limiter and bot-detection configuration before allowing multiple users or untrusted clients. The current lab configuration relaxed limiting for testing; production settings should be chosen for the expected traffic and deployment model.
- Monitor upstream search-engine errors, throttling, timeouts, and changes. SearXNG depends on external providers, so search availability and result counts are not guaranteed.
- Set reasonable request timeouts and limits at the gateway or reverse-proxy layer.
- Keep SearXNG's secret key unique to the deployment and protect its environment file.
- Treat retrieved web pages and search results as untrusted input. The model should not follow instructions found in search results that attempt to override system or tool policies.

## 5. MCP gateway and tool isolation

The current project connects Open WebUI directly to the SearXNG MCP server. This is appropriate for the initial integration.

For a future multi-tool deployment, evaluate an MCP gateway/aggregator that can expose selected tools from multiple upstream MCP servers through one client-facing endpoint. The gateway should not erase backend separation: each tool server should retain its own identity, permissions, secrets, network boundary, and logs.

Desired gateway capabilities:

- One stable Streamable HTTP MCP endpoint for the client.
- Routing to multiple upstream MCP servers and, where useful, REST/OpenAPI sources.
- Tool allowlists/denylists and per-profile or per-user tool exposure.
- Authentication and authorization at the gateway and appropriate backend boundaries.
- Secret handling that avoids putting credentials in plain-text repository files.
- Request timeouts, concurrency limits, rate limits, and sensible failure behavior.
- Audit events identifying the caller, selected tool, target backend, and outcome without logging secrets or sensitive payloads.
- Health checks and useful metrics.
- Clear support for the MCP protocol version and transport used by Open WebUI.

A gateway is a routing and policy layer, not a substitute for secure backend services. Keep high-impact tools such as database writes, shell execution, and network configuration changes disabled until explicit authorization and approval workflows exist.

## 6. Persistence, backups, and recovery

- Back up the Open WebUI named volume and any persistent SearXNG/gateway configuration.
- Protect backups with access controls and encryption appropriate to their contents. Open WebUI data may contain chats, user data, and connection configuration.
- Store backup copies separately from the running Docker host.
- Test restoration periodically; an untested backup is not a recovery plan.
- Document how to restore the environment, rotate credentials after a suspected compromise, and rebuild containers from pinned images.
- Avoid destructive cleanup commands such as `docker compose down -v` unless deletion of named volumes is intentional.

## 7. Monitoring and operations

Monitor at least:

- Container state, health-check status, restart counts, and resource usage.
- Open WebUI availability and errors.
- Ollama API availability, model-load failures, and inference resource pressure.
- MCP endpoint availability, tool-call failures, latency, and timeouts.
- SearXNG search success/failure rates and upstream engine errors.
- Disk capacity, volume growth, and backup completion.

Centralize logs only when the destination and retention policy are appropriate. Redact authorization headers, tokens, passwords, session identifiers, and sensitive query data. Define alert thresholds and an owner for responding to failures.

## 8. Shared or organizational deployment

Before serving multiple users or connecting to internal infrastructure:

- Put services on segmented networks and expose only the required entry point.
- Use HTTPS for browser and remote MCP traffic.
- Integrate an identity provider or equivalent strong authentication.
- Enforce role-based access and per-tool authorization.
- Maintain audit logs and retention policies.
- Use dedicated service accounts and narrowly scoped backend permissions.
- Establish patching, vulnerability review, backup, restore, incident response, and change-management procedures.
- Review data classification and privacy requirements before sending internal or subscriber-related data to an LLM or tool.

## Suggested implementation order

### Phase A — Local hardening

1. Pin and record container versions.
2. Review host port bindings and Windows Firewall scope.
3. Protect and back up Open WebUI data and SearXNG configuration.
4. Add service checks and a repeatable restore procedure.
5. Review SearXNG rate limiting and search-engine behavior.

### Phase B — Controlled tool expansion

1. Add a separate PostgreSQL MCP server with read-only credentials.
2. Add tool-specific permissions and test failure cases.
3. Keep each backend service isolated and document its network path.
4. Decide whether a gateway/aggregator provides enough value for the growing number of MCP connections.

### Phase C — Shared/organizational service

1. Select and deploy an MCP gateway after compatibility and security evaluation.
2. Add identity, TLS, authorization, audit, monitoring, and backup controls.
3. Pilot with read-only tools before enabling any write or administrative operations.
4. Perform security review and operational acceptance testing before production use.

## Acceptance criteria

Do not describe the service as production-ready until the deployment has documented and tested:

- Version-pinned deployments and a rollback path.
- Authenticated and authorized access to every exposed interface.
- Least-privilege access for every backend tool.
- Restricted network exposure and encrypted remote traffic where applicable.
- Backups with a successful restore test.
- Health monitoring, useful logs, and alerting.
- A documented patching and incident-response process.
- Successful end-to-end tests for both normal operation and dependency failures.
