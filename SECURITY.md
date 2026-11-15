# Security Policy

## Scope

This repository documents a local development/lab integration involving Ollama, Open WebUI, SearXNG, Docker, and an MCP server.

## Reporting a security issue

Do not post passwords, API keys, tokens, private keys, session cookies, or other sensitive information in a public issue or pull request.

If you discover a security issue in this repository, contact the repository maintainer privately through GitHub. For vulnerabilities in upstream projects, use the security reporting process provided by the relevant upstream project.

## Security expectations

- No live credentials, generated secrets, private keys, or session data belong in this repository.
- Local environment files and runtime data must remain untracked.
- Local service endpoints should not be exposed to the public internet.
- Firewall rules should be restricted to the minimum required source network.
- Review third-party scripts and container images before execution.
