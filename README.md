# Pomerium Demo — GitHub Open Source Friday

[![Open Source Friday — Pomerium Demo](https://img.youtube.com/vi/MQZGeKqqx88/maxresdefault.jpg)](https://www.youtube.com/watch?v=MQZGeKqqx88)

## What's running

- **Pomerium Core** (`git-1c02ad61`) — identity-aware access proxy. We pinned a main-branch commit because v0.32.7 and v0.32.8 contained a Model Context Protocol (MCP) bug (query strings in OpenAI's Client ID Meta Document), and the fix hadn't reached a stable release yet.
- **Pomerium Verify** (`latest`) — demo app that displays authenticated JSON Web Token (JWT) claims at `https://verify.your-domain.com`
- **Native SSH proxy** (port `2200` → upstream sshd on the host) gated by Pomerium's hosted Identity Provider (IdP)
- **MCP route** (`devmcp.your-domain.com`) — exposes a local MCP server via reverse SSH tunnel, gated by Pomerium authentication
- **Prometheus** (`v2.53.0`) + **cAdvisor** (`v0.49.1`) — metrics collection
- **Grafana** (`11.1.0`) — dashboards at `https://grafana.your-domain.com`; Pomerium handles authentication and passes JWT claims to it so you don't log in separately
- **Reverse-SSH-tunnel routes** — ngrok-style exposure of local MCP servers via `ssh -R`
- **TLS via Let's Encrypt** (autocert) for `*.your-domain.com`
- **File-backed databroker** ([Pebble](https://www.pomerium.com/blog/self-healing-file-based-databroker-without-the-postgres-headaches)) so sessions survive restarts

## Prerequisites

- A Linux host with Docker (`docker.io`) and the Docker compose plugin
- A wildcard DNS record `*.your-domain.com` pointing at the host's public IP address (required for Let's Encrypt autocert)
- Ports `80`, `443`, and `2200` open to inbound traffic
- A `demo` Linux user on the host (the SSH route's upstream target)

## Layout

```
.
├── .env                  # secrets (cookie/shared/signing) — gitignored
├── .env.example          # template
├── docker-compose.yaml  # pomerium, verify, prometheus, cadvisor, grafana
├── config/              # mounted into the container at /pomerium (read-only)
│   ├── config.yaml
│   ├── pomerium_user_ca_key{,.pub}
│   └── ssh_host_keys/
│       ├── pomerium_ssh_host_ed25519_key{,.pub}
│       ├── pomerium_ssh_host_rsa_key{,.pub}
│       └── pomerium_ssh_host_ecdsa_key{,.pub}
├── prometheus/          # Prometheus scrape configs
└── grafana/             # Grafana provisioning + dashboards
```

A writable `data/` directory (also gitignored) is bind-mounted to `/data` for
Pebble databroker state and Let's Encrypt cert cache.

## First-time setup

1. **Generate secrets** into `.env`:

   ```sh
   cp .env.example .env
   {
     printf 'COOKIE_SECRET=%s\n' "$(head -c32 /dev/urandom | base64)"
     printf 'SHARED_SECRET=%s\n' "$(head -c32 /dev/urandom | base64)"
     # signing_key must be a base64-encoded PEM ECDSA P-256 private key
     tmp=$(mktemp); openssl ecparam -genkey -name prime256v1 -noout -out "$tmp" >/dev/null
     printf 'SIGNING_KEY=%s\n' "$(base64 -w0 < "$tmp")"; rm -f "$tmp"
   } > .env
   chmod 600 .env
   ```

2. **Set your domain and email** in the config files:

   Replace all occurrences of `your-domain.com` and `your-email@example.com` with your actual domain and email address:

   ```sh
   # Set your domain (replace example.com with your actual domain)
   sed -i 's/your-domain.com/example.com/g' config/config.yaml docker-compose.yaml
   # Set your email (replace with your actual email)
   sed -i 's/your-email@example.com/you@example.com/g' config/config.yaml docker-compose.yaml
   ```

   You can also edit the files manually. The domain is used for routes and the authenticate service URL; the email is used in Pomerium policy rules to restrict access.

3. **Generate SSH host keys + the Pomerium user Certificate Authority (CA)**:

   ```sh
   mkdir -p config/ssh_host_keys
   ssh-keygen -q -N '' -t ed25519 -f config/ssh_host_keys/pomerium_ssh_host_ed25519_key
   ssh-keygen -q -N '' -t rsa -b 3072 -f config/ssh_host_keys/pomerium_ssh_host_rsa_key
   ssh-keygen -q -N '' -t ecdsa -b 256 -f config/ssh_host_keys/pomerium_ssh_host_ecdsa_key
   ssh-keygen -q -N '' -t ed25519 -f config/pomerium_user_ca_key
   ```

4. **Trust the Pomerium user CA in the host sshd** (so the upstream sshd
   accepts the short-lived user certs Pomerium issues):

   ```sh
   sudo install -m 644 config/pomerium_user_ca_key.pub /etc/ssh/pomerium_user_ca.pub
   sudo tee /etc/ssh/sshd_config.d/50-pomerium.conf <<'EOF'
   TrustedUserCAKeys /etc/ssh/pomerium_user_ca.pub
   EOF
   sudo sshd -t && sudo systemctl reload ssh
   ```

5. **Bring up the stack**:

   ```sh
   docker compose up -d
   docker compose logs -f pomerium    # watch cert acquisition
   ```

## Usage

**SSH through Pomerium** (browser-based OpenID Connect (OIDC) the first time, then cached):

```sh
ssh demo@jump@ssh.your-domain.com -p 2200
```

The triple-segment username is `<linux_user>@<route_name>@<proxy_host>` — `jump`
corresponds to `from: ssh://jump` in `config/config.yaml`.

**Verify identity headers**: open <https://verify.your-domain.com/> in a
browser. The verify app shows the JWT claims Pomerium injected via
`X-Pomerium-Jwt-Assertion`.

**Expose a laptop MCP server publicly** through a reverse SSH tunnel:

The `-R` syntax is `ssh -R <route-domain>:443:<local-host>:<local-port> -N <ssh-proxy-host> -p 2200`:

```sh
# On your laptop. No username needed — Pomerium looks up the session by SSH
# key fingerprint, so the user segment is optional for this route. `-N` keeps
# the connection open with no shell. Don't include a `route@` segment here;
# adding one puts Pomerium into jump-host mode, which passes the tcpip-forward
# request straight through to the upstream sshd instead of handling it internally.
ssh -R devmcp.your-domain.com:443:localhost:3000 -N ssh.your-domain.com -p 2200
```

Visit <https://devmcp.your-domain.com> in a browser; you'll go through the
same authentication flow as the verify route. Your laptop's app at `localhost:3000`
serves the response. Kill the `ssh -R` and the public URL stops working.

**MCP client URL**: when adding this server in an MCP client like ChatGPT or
Claude.ai, point it at `https://devmcp.your-domain.com/mcp`. It is an **HTTP
Streamable remote MCP server**. If the client asks for authentication, enable
OAuth — Pomerium handles the OIDC flow before the request reaches the MCP
server.

## Updating config

Because `config/` is mounted as a directory (not a single file), edits to
`config/config.yaml` propagate to the container immediately. Pomerium hot-reloads
most settings; if you change ports or top-level routing, run
`docker compose restart pomerium`.

## Toggling Let's Encrypt staging

While iterating, set `autocert_use_staging: true` in `config/config.yaml` and
wipe stale state to avoid burning the prod rate-limit budget:

```sh
rm -rf data/autocert/acme data/autocert/certificates
docker compose restart pomerium
```

Flip back to `false` once everything works.

## Why these specific settings

- `authenticate_service_url` is on our own subdomain so Pomerium avoids the
  stateless flow used by `*.pomerium.app`. That flow disables SSH login
  (`pkg/ssh/auth.go` rejects the connection with
  `FailedPrecondition: ssh login is not currently enabled`).
- `idp_provider: hosted` keeps Pomerium's hosted IdP for actual authentication —
  no need to set up Google/Okta/Auth0 credentials. Without this you get
  `identity: provider is not defined`.
- `pass_identity_headers: true` on the verify route surfaces the JWT claims;
  without it, the verify app reports "attestation token not found".
  `signing_key` must also be configured.
- `databroker_storage_type: file` uses Pebble, so sessions and other databroker
  state survive container restarts. You need both the type *and* the
  connection string (`file:///data/databroker`) — without it Pomerium
  silently falls back to in-memory.
- MCP routes need `mcp: true` in `runtime_flags`. Without it, Pomerium ignores
  the `mcp:` block on a route and the MCP handshake fails.
- Reverse-tunnel routes need `ssh_upstream_tunnel: true` in `runtime_flags`.
  Without it, Pomerium builds a normal Envoy cluster against the placeholder
  `to:` URL (`http://reverse-tunnel.invalid`) and you get `no_healthy_upstream`
  errors.
- The config directory is mounted at `/pomerium`, not `/etc/pomerium`.
  Pomerium's default image looks for `/pomerium/config.yaml`. Mount the
  **directory** (not a single file) so hot-reload works when editors write
  via temp files.

## Resources

- [Pomerium Core source code](https://github.com/pomerium/pomerium)
- [Pomerium Verify source code](https://github.com/pomerium/verify)
- [Pomerium Native SSH Access docs](https://www.pomerium.com/docs/capabilities/native-ssh-access)
- [Native SSH Access with Pomerium — Iximiuz Labs tutorial](https://labs.iximiuz.com/tutorials/native-ssh-access-with-pomerium-747d04bb)
- [Pomerium Policy Language (PPL) reference](https://www.pomerium.com/docs/reference/routes/policy)
- [Pomerium MCP Support docs](https://www.pomerium.com/docs/capabilities/mcp)
- [Model Context Protocol (MCP) Specification](https://modelcontextprotocol.io/)
- [MCP TypeScript Template](https://github.com/nickytonline/mcp-typescript-template) — scaffold an MCP server quickly
- [Self-Healing File-Based Databroker Without The Postgres Headaches](https://www.pomerium.com/blog/self-healing-file-based-databroker-without-the-postgres-headaches)
- [Sometimes Postgres and chill isn't the answer](https://www.youtube.com/watch?v=jKahQz7aVOk)
- [Stop AI Agents from Cooking the Books: Pomerium + MCP Demo](https://www.youtube.com/watch?v=Q8jTiNkCI3M)
- [Zero Trust: From Airports to Identity-Aware Proxies](https://www.nickyt.co/talks/zero-trust--from-airports-to-identity-aware-proxies-sreday-redmond-2025-q2)
- [Claws Out: Securing and Building with OpenClaw](https://www.nickyt.co/talks/claws-out-securing-and-building-with-openclaw-ai-engineer-europe-2026/)
- [Don't Get Rate-Limited: Use Let's Encrypt Staging](https://dev.to/nickytonline/dont-get-rate-limited-use-lets-encrypt-staging-4kk2)

