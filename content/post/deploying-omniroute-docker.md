+++
date = '2026-08-29T15:38:19+08:00'
draft = false
title = 'Deploying Omniroute on Docker'
+++

Running omniroute on Docker is the best default for me simply because I don’t want a global npm install polluting my host node environment, and it composes cleanly whether the box is a home server, a VPS, or my throwaway VM.

<!--more-->

### Picking the right image target and compose profile
The project ships a multi-stage Dockerfile with 3 build targets. `runner-base` is the production Next.js standalone runtime with no provider CLIs baked in, and it’s the correct choice if omniroute is only going to proxy requests from a Codex instance running elsewhere.

`runner-cli` extends that with git, docker.io, docker-compose, and global installs of `@openai/codex`, `@anthropic-ai/claude-code`, `droid`, and `openclaw` - useful for workflows where the agent itself needs to live inside the container, but overkill for the common case of *"Agent runs on my laptop, OmniRoute runs on a server."*

Compose exposes these as profiles rather than making you juggle Dockerfile targets by hand: base maps to the minimal runner-base image, `cli` pulls in the bundled CLIs, and host is a Linux-oriented profile that bind-mounts host CLI binaries and config directories read-only. A 4th profile, `cliproxyapi`, runs the CLIProxyAPI sidecar on port 8317 for upstream CLI proxying and can be combined with any of the others, e.g. `docker compose --profile cli --profile cliproxyapi up -d`. For a straightforward setup, base is almost always the right starting point:
```bash
docker compose --profile base up -d
```
If you’d rather skip Compose entirely, the direct docker run invocation is just as viable and is what most single-box deployments end up using once memory sizing (below) is dialed in:

```bash
docker run -d --name omniroute --restart unless-stopped --stop-timeout 40 \
  -p 127.0.0.1:20128:20128 -v omniroute-data:/app/data \
  diegosouzapw/omniroute:latest
```

Binding to `127.0.0.1` instead of `0.0.0.0` is worth doing by default and only opening it up once you’ve decided how you actually want the dashboard and API exposed — Cloudflare Tunnel, Tailscale, or a reverse proxy with TLS in front, all of which the project documents separately.

Because the Docker image always sets `OMNIROUTE_MEMORY_MB` explicitly, the bare-metal launcher’s own RAM-calibrated fallback (roughly 35% of host RAM, clamped between 512 MB and 4 GB) never kicks in under Docker - you’re expected to set the number yourself for containerized deployments, which is a departure from how omniroute serve behaves outside a container and worth knowing before you go looking for autoscaling behavior that isn’t there.







