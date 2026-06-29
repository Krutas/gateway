# Gateway Deployment Guide

## Local Development

```bash
cd /root/gateway
python -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
gateway --help
```

## Production Server

```bash
gateway server --host 127.0.0.1 --port 8000
```

To expose over the network, bind to `0.0.0.0`. Combine with a reverse proxy (Caddy/NGINX) for TLS.

## systemd Service

Create `/etc/systemd/system/gateway.service`:

```ini
[Unit]
Description=AI Agent Gateway
After=network.target

[Service]
Type=simple
User=aiagent
Group=aiagent
WorkingDirectory=/opt/gateway
Environment="PATH=/opt/gateway/.venv/bin:/usr/local/bin:/usr/bin"
Environment="GATEWAY_CONFIG=/etc/gateway/agents.yaml"
ExecStart=/opt/gateway/.venv/bin/gateway server --host 127.0.0.1 --port 8000
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Reload and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gateway
sudo systemctl status gateway
```

## Docker

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . /app
RUN pip install --no-cache-dir -e ".[dev]"
ENV GATEWAY_CONFIG=/app/agents.yaml
EXPOSE 8000
CMD ["gateway", "server", "--host", "0.0.0.0", "--port", "8000"]
```

Build and run:

```bash
docker build -t gateway .
docker run -p 8000:8000 -v $(pwd)/agents.yaml:/app/agents.yaml gateway
```

## Reverse Proxy with Caddy

```
gateway.example.com {
    reverse_proxy 127.0.0.1:8000
}
```

## TLS / Security

- Run the gateway behind a reverse proxy that terminates TLS.
- Do not expose the gateway to the public internet without authentication (Phase 4).
- Store provider API keys in environment variables, never in the repo.
