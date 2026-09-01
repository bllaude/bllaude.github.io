+++
date = '2026-08-31T21:14:27+08:00'
draft = false
title = 'Wiring Codex on Dockerized Omniroute'
+++

For anyone running Codex against multiple backends 

<!--more-->
- official OpenAI, Kimi or DeepSeek endpoint, whatever happens to be cheapest or fastest that week - putting omniroute in front of it means Codex only ever needs to know about one base URL, while omniroute absorbs the churn of provider outages, rate limits, and credential rotation on the other side.

## Memory sizing
This caught me the first time I pointed codex at a self-hosted omniroute container. The image ships with `OMNIROUTE_MEMORY_MB=1024`, which derives a 1 GiB V8 old-space ceiling via `NODE_OPTIONS=--max-old-space-size`. That’s fine for the dashboard and light chat traffic, and it’s a hard floor, not a production size. Codex’s `POST /v1/responses` calls carry long message histories and tool definitions, and omniroute’s `RTK+Caveman` compression pipeline retains multiple in-memory graphs while processing them.

A single coding-agent session routinely needs `OMNIROUTE_MEMORY_MB=8192` with the container’s cgroup memory limit set comfortably above that — `--memory=10g` or higher - because native buffers, SQLite, and compression intermediates all sit outside the V8 heap and aren’t covered by the heap ceiling itself. Two overlapping long-context requests have been observed aborting a 12 GiB heap outright with `FATAL ERROR: Reached heap limit`, so if you’re running codex alongside anything else concurrently, size up further or keep concurrency to one heavy in-flight request per process rather than raising `OMNIROUTE_CHAT_MAX_HEAVY_IN_FLIGHT` and hoping the RAM keeps up.

The corrected command looks like this:
```bash
docker run -d --name omniroute --restart unless-stopped --stop-timeout 40 \
-e OMNIROUTE_MEMORY_MB=8192 --memory=10g \
-p 127.0.0.1:20128:20128 -v omniroute-data:/app/data \
diegosouzapw/omniroute:latest
```

This is the part that trips me up conceptually, omniroute’s `setup-codex` command writes real files into ~`/.codex` on whatever machine runs it, and if that machine is the container itself, the write lands in the container’s ephemeral home directory (`/home/node`, since the image runs as the unprivileged node user) and vanishes the moment the container is recreated. Omniroute actually detects this and refuses rather than silently reporting `success: the CLI exits with status 2`, and the `API layer returns a 422 with containerEphemeralTarget: true`.

The documented and recommended pattern keeps the 2 concerns separate: the container serves the OpenAI-compatible API, and the CLI - installed on your actual laptop or workstation, wherever codex itself runs - handles writing the config that codex reads:
```bash
docker compose --profile base up -d

npm install -g omniroute
omniroute connect http://localhost:20128
omniroute setup-codex
```

Omniroute connect points the locally installed CLI at the container’s exposed port, and omniroute setup-codex then writes the actual `~/.codex/*.config.toml` on the host, pointing codex at omniroute’s base URL and injecting whatever API key the dashboard generated under `Dashboard → Endpoints`. This is the right shape for the overwhelmingly common setup where codex runs locally and omniroute runs on a server or in a local container.

If you specifically want the container to be able to write host configs directly - say, orchestrating setup from inside a script that already runs in-container - the host `Compose` profile bind-mounts the relevant directories and sets `CLI_CONFIG_HOME` to the mount root:

```yaml
environment:
  - CLI_CONFIG_HOME=/host-home
  - CLI_ALLOW_CONFIG_WRITES=true
volumes:
  - ~/.codex:/host-home/.codex:rw
  - ~/.claude:/host-home/.claude:rw
```

Omniroute treats the presence of a genuine bind mount, verified by reading `/proc/self/mountinfo`, as the signal that a path is trustworthy enough to write into; it refuses writes to anything that isn’t actually mounted through from the host, which is what stops the ephemeral-write footgun from being possible even if you try.

There’s also a deliberate escape hatch for the case where codex is meant to live inside the container itself (the cli profile) - passing `--allow-container-write` to `setup-codex`, or setting `OMNIROUTE_ALLOW_CONTAINER_CONFIG_WRITE=true` on the server, lets the write proceed with an explicit warning that it won’t survive a container recreation.

## Redis, persistence, and the things that bite on restart
Redis backs omniroute’s distributed rate limiter and shared cache, and the redis service in `docker-compose.yml` has no profile gate - it starts alongside whichever profile you choose. It’s published only to `127.0.0.1` by default because it runs without requirepass; if you need it reachable from elsewhere on the network, set `REDIS_BIND_HOST=0.0.0.0` and add `--requirepass` to the service command in the same change, not as a follow-up.

Disabling Redis is possible but not recommended, since the rate limiter degrades to an in-memory fallback that doesn’t survive restarts or scale across processes.

Two operational details are easy to overlook and both matter specifically for a codex workflow that’s mid-session when something goes wrong. First, omniroute uses SQLite in WAL mode by default, and it needs docker stop to actually finish - not get killed - so it can checkpoint into `storage.sqlite;` the bundled Compose files set a 40-second stop grace period for this reason, and if you’re running the bare docker run form, keep `--stop-timeout 40` rather than trusting the Docker default.

Second, the stock deployment is one Node process backed by one SQLite writer, full stop - there is no supported way to run multiple replicas against the same SQLite file, and doing so corrupts the database. A recreate, restart, or failed healthcheck means a full outage of any in-flight codex session: SSE streams drop, and requests that land during the gap get a bare `502 Bad Gateway` from whatever’s in front rather than a JSON error omniroute itself would produce, which can look confusingly like a provider-side failure rather than an infrastructure one. If you genuinely need more than one or two concurrent long-context codex sessions, the documented path is `N` independent processes with separate `DATA_DIR` volumes and `QUOTA_STORE_DRIVER=redis` for shared quota counting - not `replicas > 1` against shared storage.

Always mount `/app/data` to a named volume regardless of which profile you pick; it’s where the db, encrypted provider credentials, and configuration live, and skipping it means every container recreation starts from zero.

## A reasonable baseline
Putting the pieces together, a single-box deployment intended to front codex reliably looks like this in Compose:
```yaml
services:
  omniroute:
    image: diegosouzapw/omniroute:latest
    container_name: omniroute
    restart: unless-stopped
    stop_grace_period: 40s
    environment:
      OMNIROUTE_MEMORY_MB: "8192"
      OMNIROUTE_WS_BRIDGE_SECRET: "<generate a strong random value>"
    volumes:
      - omniroute-data:/app/data
    ports:
      - "127.0.0.1:20128:20128"
    deploy:
      resources:
        limits:
          memory: 10g

volumes:
  omniroute-data:
```

Bring it up with `docker compose --profile base up -d`, then run `npm install -g omniroute`, `omniroute connect http://localhost:20128`, and `omniroute setup-codex` from the machine where codex actually runs.

Pin the image tag to a specific `X.Y.Z` release rather than tracking `:latest` if this is anything other than a personal box you don’t mind recreating on a whim — `:latest` only moves when a stable `SemVer version` is published and promoted, so it’s not a currency guarantee for whatever landed on main yesterday, and pinning is what makes the deployment reproducible when you inevitably need to roll back after a bad provider or memory-sizing surprise.
















