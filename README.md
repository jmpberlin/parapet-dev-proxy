# parapet-dev-proxy

A forward proxy that restricts what the `parapet` dev server (`parapet-hetzner-dev`) is allowed to talk to on the internet — enforced as a domain allowlist, running on its own, separate box.

This repo holds the Squid configuration and the GitHub Action that deploys it. It exists so that changing the allowlist is a normal, reviewable git change, not a manual edit on a live server.

---

## Why this exists

The dev box runs code written by an AI pipeline (`ai-dev-orchestration`), often without a human having read every line before it runs. If that code — accidentally or maliciously — tried to send data somewhere it shouldn't, the only thing standing in the way is network egress control. The goal: the dev box can reach a short, explicit list of domains it actually needs (GitHub, GHCR, Anthropic's API, a couple of content sources) and **nothing else**.

### Why not just use the Hetzner Cloud Firewall for this

Hetzner's Cloud Firewall filters by **IP address / CIDR range only** — it has no concept of a domain name. Every domain on our allowlist (`api.github.com`, `ghcr.io`, Docker Hub, etc.) is served from CDN infrastructure with IP ranges that are large, shared with countless other tenants, and change over time. GitHub's own documentation explicitly advises against allowlisting by IP for this reason. So a domain-based allowlist has to be enforced by something that actually understands domains — that's what Squid does, via its `CONNECT` handling (it reads the hostname in the `CONNECT api.github.com:443` line and decides there, without ever decrypting the TLS session itself).

### Why a *second, separate* box, rather than a proxy container next to the app

If the proxy ran on the same machine as the app (even in a separate container), a sufficiently compromised app container or a host-level compromise could disable or bypass it — root on a box can always flush its own iptables rules or kill a sibling container. Putting the proxy on an **entirely separate machine** means the dev box would have to compromise a *different* machine — one it holds no credentials to — to defeat the restriction. That asymmetry (the app box can send traffic to the proxy, but cannot reconfigure it) is the actual security property here, not just tidiness.

### The two independent layers

1. **Hetzner Cloud Firewall** (enforced outside the guest OS, on both boxes):
   - `parapet-hetzner-dev` may only egress to this proxy's IP on `tcp:3128`, plus what Tailscale itself needs (see below).
   - This proxy box only accepts inbound from the dev box's IP on `tcp:3128`.
2. **Squid's own ACL** (`squid.conf` in this repo): even if something reached this box on 3128, Squid only relays to the domains explicitly listed — everything else gets denied at the application layer.

Two layers, two different machines enforcing them, so defeating one doesn't defeat the other.

---

## One-time manual setup

These steps can't be automated by the pipeline — provisioning cloud infrastructure and the very first Squid install have to happen before there's anything for a GitHub Action to deploy *to*.

### 1. Provision the box

A small Hetzner Cloud server (CX22/CX23 class, x86) with a public IP. No private Cloud Network needed — this design deliberately uses public IPs with the Cloud Firewall as the real boundary (a Hetzner private network is explicitly *not* filtered by their Cloud Firewall, so it would have been a false sense of security here).

### 2. Initial access

Use whatever SSH key or root password Hetzner issues at creation to log in exactly once:

```bash
ssh root@<proxy-public-ip>
```

### 3. Join the tailnet (for your own management access, not for app traffic)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --advertise-tags=tag:proxy-server --ssh
```

Follow the browser auth link. Afterwards, confirm in the Tailscale admin console that the machine registered with `tag:proxy-server` (not as an untagged personal device). From then on, `tailscale ssh root@<proxy-hostname>` is how you and CI reach this box — never a public SSH port.

The tailnet policy file needs `tag:proxy-server` defined and a grant allowing `tag:ci` to reach it on `tcp:22`:

```jsonc
{
  "tagOwners": {
    "tag:ci":           ["autogroup:admin"],
    "tag:dev-server":   ["autogroup:admin"],
    "tag:proxy-server": ["autogroup:admin"],
  },
  "grants": [
    { "src": ["autogroup:member"], "dst": ["*"], "ip": ["*"] },
    { "src": ["tag:ci"], "dst": ["tag:dev-server"],   "ip": ["tcp:22"] },
    { "src": ["tag:ci"], "dst": ["tag:proxy-server"], "ip": ["tcp:22"] },
  ],
  "ssh": [
    { "action": "check",  "src": ["autogroup:member"], "dst": ["tag:dev-server"],   "users": ["root"] },
    { "action": "accept", "src": ["tag:ci"],           "dst": ["tag:dev-server"],   "users": ["root"] },
    { "action": "check",  "src": ["autogroup:member"], "dst": ["tag:proxy-server"], "users": ["root"] },
    { "action": "accept", "src": ["tag:ci"],           "dst": ["tag:proxy-server"], "users": ["root"] },
  ],
}
```

### 4. Hetzner Cloud Firewall rules

**On this proxy box:**
- Inbound: allow only `<dev-server-public-ip>/32` on `tcp:3128`. Deny everything else inbound.
- Outbound: leave open. Squid's own ACL is what restricts *where* this box relays traffic — restricting this box's own egress by IP would just reintroduce the same "CDN IPs change" problem this whole proxy exists to solve.

**On `parapet-hetzner-dev`** (documented here since the two are a matched pair, even though this rule lives on the other box):
- Outbound, allow only:
  - `<this-proxy-public-ip>/32` on `tcp:3128` — carries the app's API calls, `git fetch`, and `docker pull`, all tunneled through here.
  - `tcp:443` to `0.0.0.0/0` — Tailscale's control plane. This **cannot** be narrowed to just the DERP relay IPs (a separate, smaller set) without risk of breaking Tailscale itself, since the control-plane servers aren't in that list and don't publish a stable range. Tailscale's own guidance is to leave this wide rather than enumerate it.
  - `udp:41641` to `0.0.0.0/0` — direct WireGuard.
  - `udp:3478` to `0.0.0.0/0` — STUN, for NAT traversal.
  - `udp:53` — DNS, needed for Tailscale's own bootstrap resolution.
- Inbound: deny everything. Access is via Tailscale only.

### 5. Install Squid

```bash
apt update && apt install -y squid
```

This starts Squid via systemd with a default config. The **content** of that config (the allowlist) is what this repo manages from here on — don't hand-edit `/etc/squid/squid.conf` on the box after this point; treat this repo as the source of truth (see below).

### 6. Configure this repo

**Secrets** (repo Settings → Secrets and variables → Actions → Secrets):
- `TS_OAUTH_CLIENT_ID`, `TS_OAUTH_SECRET` — the Tailscale OAuth client credentials tagged `tag:ci`. These are the **same credentials** used by the `deploy-dev.yml` workflow in the main `parapet` repo — one OAuth client, scoped by tag, reused across repos. If you ever regenerate them, update both repos.

**Variables** (Settings → Secrets and variables → Actions → Variables):
- `PROXY_HOST` — this proxy box's Tailscale MagicDNS name (e.g. `parapet-hetzner-proxy.taile7829a.ts.net`). Find it via `tailscale status` on the box, or the Machines page in the admin console.
- `APP_BOX_PUBLIC_IP` — `parapet-hetzner-dev`'s public IP. Used to render `allowed_src` in the Squid config template.

### 7. First deploy

Push to `main` (or merge the PR that adds this setup). The Action renders `squid.conf.template`, validates it, connects to the tailnet, ships it to this box, and reloads Squid. See "Verifying it's working" below to confirm end to end.

---

## Ongoing usage — changing the allowlist

Edit `squid.conf.template` in a branch, open a PR, get it reviewed, merge to `main`. The Action handles the rest automatically.

**This repo is deliberately excluded from the AI dev pipeline's (`ai-dev-orchestration`) repo access.** That pipeline can open PRs and push code in `ai-dev-orchestration` and `parapet` — never here. The isolation this proxy provides only means something if the thing it's isolating can't also edit the isolation itself. Keep it that way.

Branch protection on `main` (requiring a PR review before merge, even from a single reviewer) is recommended for the same reason — "push to main" should mean "a human looked at this diff," since this repo enforces a security boundary.

---

## How the deploy pipeline works

```
push to main
  → checkout
  → envsubst renders squid.conf.template → squid.conf (using APP_BOX_PUBLIC_IP)
  → squid -k parse inside a container, on the runner — catches syntax errors
    before anything touches the real box
  → connect to the tailnet as an ephemeral tag:ci node
    (via TS_OAUTH_CLIENT_ID / TS_OAUTH_SECRET)
  → ssh to PROXY_HOST:
      - back up the current /etc/squid/squid.conf to squid.conf.bak
      - write the new config
      - squid -k parse again, on the box itself
      - systemctl reload squid only if that parse succeeds
  → smoke test: confirm an allowed domain works and a non-listed one is denied
```

The double validation (once on the runner, once on the box right before reload) means a bad config fails the deploy without ever taking down the currently-running Squid — reload only happens if the new file parses cleanly.

---

## Verifying it's working

From `parapet-hetzner-dev`, after any deploy:

```bash
# should succeed — an allowlisted domain
curl -x http://<proxy-ip>:3128 --fail -sv https://api.github.com

# should be denied — anything not on the list
curl -x http://<proxy-ip>:3128 --fail -sv https://example.com
```

On this proxy box, to check Squid's own state without starting a conflicting second instance:

```bash
systemctl status squid      # should show active (running)
squid -k parse              # validates the live config, doesn't touch the running process
ss -tlnp | grep 3128         # confirms Squid is actually listening
```

---

## Troubleshooting

**`git fetch`/`curl` on the dev box hangs or refuses instantly when using the proxy.**
Check in this order: (1) is Squid actually running on this box (`systemctl status squid`)? (2) does this box's inbound Cloud Firewall rule have the dev box's *current* public IP, correctly, as source? (3) from the dev box, does a raw TCP connect to `<proxy-ip>:3128` succeed at all (`curl -v telnet://<proxy-ip>:3128`), independent of git/docker? That isolates whether it's a firewall problem or a Squid/config problem.

**`squid` CLI says "FATAL: Squid is already running."**
Not an error — it means the systemd-managed instance is already up and you tried to start a second one from the command line. Use `systemctl status squid` to check status, and `squid -k parse` (which doesn't try to bind the port) to validate config without conflicting with the running process.

**Domains that should be blocked aren't, or allowed ones are being blocked.**
Check `squid.conf`'s `allowed_dst` list — remember a leading dot (`.githubusercontent.com`) matches the domain *and all subdomains*, while a bare domain (`ghcr.io`) matches only that exact host. GHCR in particular serves manifests from `ghcr.io` but image layer blobs from a `githubusercontent.com` subdomain — missing the latter causes pulls that authenticate fine and then hang.

---

## Related

- `parapet` repo — the application this proxy protects; its `.env` sets `HTTP_PROXY`/`HTTPS_PROXY` to point at this box, and its `deploy-dev.yml` shares the same Tailscale OAuth client.
- Linear PAR-97 — the ticket tracking this restriction.
- Linear PAR-98 — the broader dev-environment runbook this proxy is one piece of.