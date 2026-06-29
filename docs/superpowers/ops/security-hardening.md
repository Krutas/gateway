# Gateway Security Hardening Checklist

## Pre-Flight

- [ ] Run gateway behind a reverse proxy with TLS termination.
- [ ] Bind to `127.0.0.1` unless you intentionally need LAN access.
- [ ] Never commit `~/.gateway/agents.yaml` or `.env` files containing API keys.
- [ ] Use a dedicated non-root OS user to run the gateway service.

## Authentication

- [ ] Enable per-agent bearer tokens (Phase 4).
- [ ] Rotate tokens after sharing with third-party tools.
- [ ] Use different keys per agent to contain blast radius.

## Provider Secrets

- [ ] Store provider API keys in environment variables: `export OPENAI_API_KEY=...`
- [ ] Reference them in YAML with env var interpolation (after implementation):
  ```yaml
  providers:
    openai:
      base_url: https://api.openai.com/v1
      api_key: ${OPENAI_API_KEY}
  ```
- [ ] Audit `ps e` output to ensure secrets are not leaked in subprocess env.

## Rate Limiting

- [ ] Cap requests per minute per API key (Phase 4).
- [ ] Cap concurrent streams per agent.
- [ ] Add request body size limits (`--limit-request-body`) in reverse proxy.

## Audit

- [ ] Log metadata only: timestamp, agent_id, endpoint, status code, IP.
- [ ] Do not log full request/response bodies unless debug mode is on.
- [ ] Ship logs to a centralized log aggregator.

## Subprocess Isolation

- [ ] Run agent harnesses under separate OS users where feasible.
- [ ] Restrict filesystem access with chroot or containerization.
- [ ] Kill runaway agents via timeout and cgroup limits.
