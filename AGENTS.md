# AGENTS.md — guidance for AI coding agents

This file captures non-obvious conventions and gotchas specific to this Pomerium
deployment. Read this before editing config, restarting services, or "fixing"
something that looks weird.

## Project shape

- Single-host Pomerium Core (image pinned to `pomerium/pomerium:git-1c02ad61`, latest main commit as of 2026-06-12) plus
  `pomerium/verify`, Prometheus, cAdvisor, and Grafana.
- All paths in this README are relative to `/home/demo/pomerium-demo/` on the host.
- The `demo` Linux user is the SSH-route upstream target; it has root's
  `authorized_keys` for non-Pomerium SSH access as a fallback.
- MCP routes (`devmcp`, `dev`) and reverse SSH tunnel (`upstream_tunnel`) are
  configured for exposing local MCP servers publicly.

## Critical files

| Path | Purpose |
| --- | --- |
| `docker-compose.yaml` | Service definitions, port mappings, mounts |
| `config/config.yaml` | All Pomerium settings (routes, autocert, SSH, IdP) |
| `config/pomerium_user_ca_key{,.pub}` | CA that signs short-lived user certs |
| `config/ssh_host_keys/*` | SSH host keys the proxy presents to clients |
| `.env` | Secrets (`COOKIE_SECRET`, `SHARED_SECRET`, `SIGNING_KEY`) — chmod 600, never committed |
| `data/databroker/` | Pebble KV store (do not edit) |
| `data/autocert/` | Let's Encrypt cert cache (delete to re-issue) |
| `/etc/ssh/pomerium_user_ca.pub` | User CA pubkey trusted by host sshd |
| `/etc/ssh/sshd_config.d/50-pomerium.conf` | `TrustedUserCAKeys` drop-in |

## Conventions (these matter)

1. **Mount `config/` as a directory, never as a single file.** Bind-mounting a
   single file pins the inode at container start; editors that write via temp
   files (vim/$EDITOR) silently break hot-reload.
2. **Mount `config/` read-only** (`:ro`). The container should never write to
   its own config tree.
3. **Secrets live in `.env`, not in `config.yaml`.** Reference them in YAML with
   `${VAR}` and Pomerium picks them up via `env_file:`. Anything sensitive
   (cookie/shared/signing) belongs in `.env`.
4. **Inside the container, Pomerium reads `/pomerium/config.yaml`**, NOT
   `/etc/pomerium/config.yaml`. This is the image's default `--config` path.
   Get this wrong and Pomerium starts with defaults and silently drops every
   route — symptom is logs saying "configuration has no policies".
5. **Don't run `docker compose up` as `demo`** unless you've added them to the
   `docker` group. Run as root or via sudo.

## Pomerium-specific footguns we already hit

- **Hosted authenticate disables native SSH.** If `authenticate_service_url` is
  unset or points at `*.pomerium.app`, `UseStatelessAuthenticateFlow()` returns
  true and `pkg/ssh/auth.go` rejects every SSH login with
  `FailedPrecondition: ssh login is not currently enabled`. Fix: set
  `authenticate_service_url` to a hostname under your own wildcard domain.
- **Stateful flow still needs an IdP.** Setting the URL alone gives
  `identity: provider is not defined`. Use `idp_provider: hosted` to keep using
  Pomerium's hosted IdP without providing your own client_id/client_secret.
- **The verify app shows "attestation token not found"** unless the route has
  `pass_identity_headers: true` AND `signing_key` is configured globally.
- **Pebble databroker requires `databroker_storage_type: file` PLUS
  `databroker_storage_connection_string: file:///<path>`.** Without the
  connection string, Pomerium silently falls back to in-memory.
- **SSH client command format is `user@route@host`** (two `@`s). Single-`@`
  drops the client into Pomerium's internal CLI (`whoami`, `logout`).
- **MCP routes need `mcp: true` in `runtime_flags`.** Without it, Pomerium
  ignores the `mcp:` block on a route and the MCP handshake fails.
- **`upstream_tunnel` routes need `ssh_upstream_tunnel: true` in `runtime_flags`.**
  Without it, Pomerium builds a normal Envoy cluster against the placeholder `to:`
  URL and you get `no_healthy_upstream` errors.
- **Reverse-tunnel SSH commands must NOT include a `route@` segment.**
  `ssh -R devmcp.your-domain.com:443:localhost:3000 -N ssh.your-domain.com -p 2200`
  (single `@`, no route) registers the tunnel. Adding `demo@jump@...` hands the
  stream to the upstream sshd, which rejects the `tcpip-forward` bind.
- **`mcp_allowed_client_id_domains` gates MCP client registration.** The domain
  of the MCP client's `client_id` URL must match an entry here or the authorize
  request 401s before any user-login redirect.

## Common commands

```sh
# Start / stop
docker compose up -d
docker compose down

# Restart just pomerium (config edits hot-reload — usually no restart needed)
docker compose restart pomerium

# Tail logs
docker compose logs -f pomerium

# Show only error/warn lines (skip envoy chatter)
docker compose logs pomerium 2>&1 \
  | grep -iE 'error|warn|fatal' \
  | grep -viE 'sync.*shutdown|context canceled|envoy.*name'

# Inspect what the container actually sees
docker run --rm -v ./config:/pomerium:ro alpine ls -la /pomerium

# Probe the SSH listener (banner should be "SSH-2.0-Envoy")
nc -w 2 127.0.0.1 2200 | head -c 32
```

## Verifying behavior

After any non-trivial change, run through this checklist:

1. `docker compose ps` shows both `pomerium` and `pomerium-verify` Up.
2. `docker compose logs pomerium` since restart has no `error`/`warn` lines
   other than transient autocert-account-not-found at startup.
3. `openssl s_client -connect 127.0.0.1:443 -servername verify.your-domain.com`
   returns a cert with issuer `Let's Encrypt CN=E*` (or `(STAGING) ...` if
   `autocert_use_staging: true`).
4. `ssh demo@jump@ssh.your-domain.com -p 2200` completes the OIDC flow
   and lands you in a shell as `demo` on the host.
5. <https://verify.your-domain.com/> shows decoded JWT claims for the
   authenticated user.

## When in doubt

- Pomerium source (Go): <https://github.com/pomerium/pomerium>, especially
  `pkg/ssh/auth.go` and `config/options.go`.
- The docs index `https://www.pomerium.com/llms.txt` lists every doc page in
  one place — easier than guessing URLs.
- The single `config.yaml` is the source of truth for everything except
  secrets (which live in `.env`). No hidden state in `data/` is meant to be
  human-edited.
