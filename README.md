# parapet-dev-proxy
 
The Squid configuration and the GitHub Action that deploys it, for the **gateway** that restricts what the `parapet` dev box may talk to on the internet. Egress is enforced as a domain allowlist on a separate trusted machine.
 
This repo exists so that changing the allowlist is a normal, reviewable git change, not a manual edit on a live server.
 
> **Full architecture and from-scratch setup:** see `DEPLOY_DEV.md` in the `parapet` repo (Phases 0–15). This README covers only what lives in *this* repo: the allowlist and its deploy pipeline.
 
---
 
## Architecture in one picture
 
```
                         Tailscale tailnet
   laptop / CI (tag:ci) ───────────────┐
                                       ▼
                        ┌──────────────────────────┐
                        │ Gateway (trusted)        │  public IP
                        │ 10.0.0.2                 │  tag:gateway
                        │  • Tailscale subnet rtr  │
                        │  • Squid :3128 (private) │
                        │  • nftables (one-way+NAT)│
                        └────────────┬─────────────┘
              Hetzner private network│ 10.0.0.0/16
                        ┌────────────▼─────────────┐
                        │ Dev box (untrusted)      │  NO public IP
                        │ 10.0.0.4                 │  NO Tailscale
                        └──────────────────────────┘
```
 
- **Inbound** (CI/laptop → dev box): tailnet → gateway's advertised route `10.0.0.4/32` → dev box. The gateway masquerades this traffic.
- **Outbound** (dev box → internet): only via `HTTP_PROXY`/`HTTPS_PROXY` to Squid on the gateway (`10.0.0.2:3128`), which allows listed domains only.
- **Dev box → tailnet:** dropped by nftables on the gateway. The dev box cannot initiate connections into the tailnet.
## Why this exists
 
The dev box runs code written by an AI pipeline (`ai-dev-orchestration`), often before a human has read every line. If that code tried to send data somewhere it shouldn't, network egress control is what stands in the way. The dev box may reach a short, explicit list of domains (GitHub, GHCR, Docker, Anthropic's API, a couple of content sources) and nothing else.
 
**Non-goal:** preventing exfiltration entirely. An allowlisted domain such as GitHub or an LLM API is also a possible exfiltration channel. This design reduces blast radius; it does not eliminate it.
 
### Why not the Hetzner Cloud Firewall
 
- It filters by IP/CIDR only. Our domains sit behind CDNs with large, shifting IP ranges, so a domain allowlist needs something that reads the hostname (Squid, via the `CONNECT host:443` line, without decrypting TLS).
- It does **not** filter Hetzner private-network traffic, so it cannot be the control for the dev-box → gateway path.
### Why a separate box
 
Root on the app box can flush its own firewall rules or kill a sibling container. The proxy therefore runs on a different machine the dev box holds no credentials for. The dev box can *use* the proxy but cannot *reconfigure* it; that asymmetry is the security property.
 
---
 
## What lives where
 
| Concern | Where |
|---|---|
| Squid allowlist + deploy pipeline | **this repo** |
| Gateway provisioning (Tailscale subnet router, nftables, NAT, SSH keys), dev box setup, CI wiring | `parapet` repo → `DEPLOY_DEV.md` |
| Tailscale ACL policy | Tailscale admin console (copy kept in `DEPLOY_DEV.md`, Phase 3) |
 
The gateway is tagged `tag:gateway` on the tailnet. There is no `tag:proxy-server` and no `tag:dev-server`: the dev box is not on the tailnet at all.
 
---
 
## One-time setup
 
The gateway is provisioned by hand once, following `DEPLOY_DEV.md` Phases 0–6 (private network, `dev_deploy` key, Tailscale subnet router, ACL, nftables, NAT masquerade, Squid install). After that, this repo manages Squid's configuration. Do not hand-edit `/etc/squid/squid.conf` as a permanent fix.
 
### Repo configuration
 
**Secrets** (Settings → Secrets and variables → Actions → Secrets):
- `TS_OAUTH_CLIENT_ID`, `TS_OAUTH_SECRET`: the Tailscale OAuth client for `tag:ci`. These are the **same credentials** used by the workflows in the `parapet` repo. If you regenerate them, update both repos.
**Variables:**
- `GATEWAY_TAILNET_HOST`: the gateway's Tailscale MagicDNS name (find it with `tailscale status` on the gateway, or in the admin console).
- `DEV_SERVER_PRIVATE_IP`: the dev box's private IP (e.g. `10.0.0.4`), rendered into `allowed_src`.
- `GATEWAY_PRIVATE_IP`: the gateway's private IP (e.g. `10.0.0.2`), rendered into `http_port` and used by the smoke test.
---
 
## Changing the allowlist
 
Edit `squid.conf.template` on a branch, open a PR, get it reviewed, merge to `main`. The Action does the rest.
 
**This repo is deliberately outside the AI pipeline's repo access.** The pipeline may open PRs and push code in `ai-dev-orchestration` and `parapet`, never here. The containment only means something if the thing being contained can't edit the containment. Protect `main` with required PR review for the same reason: "push to main" should mean "a human looked at this diff".
 
### Domain-matching gotchas
 
`dstdomain` without a leading dot matches **only that exact hostname**; with a leading dot it matches the domain and all subdomains.
 
- `ghcr.io` does not match its blob-storage host (`.githubusercontent.com`). Missing it makes pulls authenticate and then hang.
- Plain `git fetch` over HTTPS hits bare `github.com`, not only `api.github.com`.
- Docker Hub pulls need `.docker.io` **and** `.docker.com` (the CDN host `production.cloudfront.docker.com` returned 403 without it).
- `bleepingcomputer.com` does not match `www.bleepingcomputer.com`; use `.bleepingcomputer.com`.
When adding an entry, check which host the *actual request* hits (Squid's `access.log` shows `TCP_DENIED` lines) rather than guessing.
 
Current allowlist: `api.github.com`, `github.com`, `.githubusercontent.com`, `ghcr.io`, `api.anthropic.com`, `.bleepingcomputer.com`, `socket.dev`, `.docker.io`, `.docker.com`.
 
---
 
## How the deploy pipeline works
 
```
push to main
  → checkout
  → envsubst renders squid.conf.template → squid.conf
  → squid -k parse on the runner (catches syntax errors early)
  → connect to the tailnet as an ephemeral tag:ci node
  → ssh to the gateway (GATEWAY_TAILNET_HOST):
      - back up current /etc/squid/squid.conf → squid.conf.bak
      - write the new config
      - squid -k parse again on the box
      - systemctl reload squid only if that parse succeeds
  → smoke test, run FROM THE DEV BOX (ssh to it via the gateway):
      - an allowlisted domain succeeds through the proxy
      - a non-listed domain is denied
```
 
Validation happens twice (runner, then box) so a bad config fails the deploy without touching the running Squid.
 
**The smoke test must run from the dev box**, not the gateway: `allowed_src` only permits the dev box's IP, so a curl from the gateway itself is denied (`TCP_DENIED`) for the wrong reason.
 
---
 
## Verifying
 
From the dev box:
 
```bash
# allowed: should succeed
curl -x http://<gateway-private-ip>:3128 --fail -sv https://api.github.com
 
# not on the list: should be denied (403)
curl -x http://<gateway-private-ip>:3128 --fail -sv https://example.com
```
 
On the gateway:
 
```bash
systemctl status squid
squid -k parse
ss -tlnp | grep 3128          # must show the PRIVATE IP, not 0.0.0.0
tail -f /var/log/squid/access.log
nft list ruleset               # one-way forward rule + masquerade present
```
 
---
 
## Troubleshooting
 
**Proxy requests hang or are refused from the dev box.** In order: (1) is Squid running (`systemctl status squid`)? (2) is it listening on the gateway's private IP on 3128? (3) from the dev box, does a raw TCP connect to `<gateway-private-ip>:3128` work? That separates a network problem from a Squid problem.
 
**A needed domain is blocked.** Read `/var/log/squid/access.log` for the `TCP_DENIED` host, then add it (with a leading dot if subdomains are needed) via a PR.
 
**Runner can't SSH to the gateway.** Check the runner joined the tailnet as `tag:ci` and the ACL still grants `tag:ci` → `tag:gateway` on `tcp:22`.
 
**Dev box unreachable from CI/laptop while the gateway is fine.** Almost always the NAT masquerade rule is missing (Hetzner drops forwarded packets with tailnet source addresses, silently and invisibly to `tcpdump`). Check `nft list ruleset` for the masquerade rule, and that the route `10.0.0.4/32` is approved in the Tailscale admin console.
 
**`squid` CLI says "Squid is already running".** Not an error; the systemd instance is up. Use `systemctl status squid` and `squid -k parse`.
 
---
 
## Related
 
- `parapet` repo: `DEPLOY_DEV.md` (full runbook + disaster recovery), `DEPLOY.md` (prod).
- Linear PAR-97 (original egress ticket, superseded design) and PAR-98 (runbook ticket, with the design rationale and dead ends).