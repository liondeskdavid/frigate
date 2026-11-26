# Remote access to Frigate (Windows)

**Chosen path: Cloudflare Tunnel → Frigate authenticated UI on port 8971.**

**Canonical hostname:** [`https://frigate.andersonfamily.ai`](https://frigate.andersonfamily.ai)

That URL is what the Anderson Family Hub uses for **Open Frigate** via `NEXT_PUBLIC_FRIGATE_URL`.

Do **not** tunnel port `5000` (unauthenticated). Do **not** invent camera IPs.

## Architecture

```
Browser / https://www.andersonfamily.ai/security
        │  Open Frigate → https://frigate.andersonfamily.ai
        ▼
Cloudflare Tunnel (+ optional Cloudflare Access)
        │  HTTP to Windows PC
        ▼
Frigate :8971  (JWT login — full admin UI + API + proxied MSE streams)
```

- **Local (PC only):** http://localhost:5000 (no auth — never put on the public hostname)
- **Remote:** https://frigate.andersonfamily.ai → `http://localhost:8971`
- **go2rtc WebRTC (8555):** LAN only. Remote live view uses MSE through the tunneled UI.

## DNS for `andersonfamily.ai` (exact steps)

Today the apex uses **GoDaddy DomainControl** nameservers (`ns15` / `ns16.domaincontrol.com`), and `www` already CNAMEs to Vercel. Cloudflare is **not** yet the DNS authority for this domain. You still need a Cloudflare account for the tunnel; then publish `frigate` with a CNAME.

### A. Add the domain to Cloudflare (required for Tunnel hostnames)

1. Sign in at [dash.cloudflare.com](https://dash.cloudflare.com/) (Free plan is enough).
2. **Add a site** → enter `andersonfamily.ai`.
3. Cloudflare will show two options. Pick **one**:

#### Option 1 — Keep GoDaddy as DNS (recommended first; least disruption)

Use Cloudflare **partial / CNAME setup** (or add the site and **do not** change nameservers yet):

1. Finish adding the site in Cloudflare so `andersonfamily.ai` appears in your account (needed for Zero Trust public hostnames).
2. Leave GoDaddy nameservers as `ns15.domaincontrol.com` / `ns16.domaincontrol.com` for now so Vercel `www` keeps working.
3. After you create the tunnel (section below), create the **GoDaddy DNS** record in section C.

#### Option 2 — Move DNS to Cloudflare (full setup)

1. In Cloudflare, switch the zone to Full DNS and copy the assigned Cloudflare nameservers.
2. At [GoDaddy DNS](https://dcc.godaddy.com/) → **andersonfamily.ai** → **Nameservers** → change to Cloudflare’s NS.
3. Recreate records Cloudflare needs for the hub (at minimum):

| Type | Name | Target / value |
| --- | --- | --- |
| CNAME | `www` | `cname.vercel-dns.com` (or the current Vercel target shown in the Vercel domain UI) |
| A / ALIAS | `@` | whatever Vercel documents for the apex (today apex resolves to a Vercel anycast IP) |

Wait for NS propagation, then continue. Cloudflare can auto-create the `frigate` CNAME when you add the tunnel public hostname.

### B. Create the tunnel + public hostname

1. Open [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) → **Networks → Tunnels → Create a tunnel** → Cloudflared → name `anderson-frigate`.
2. Install on Windows (copy the token / install command). Save the token as `CLOUDFLARE_TUNNEL_TOKEN` in `home/.env`.
3. **Public Hostname** (Published application):

| Field | Value |
| --- | --- |
| Subdomain | `frigate` |
| Domain | `andersonfamily.ai` |
| Type | HTTP |
| URL | `localhost:8971` |

Resulting public URL: **https://frigate.andersonfamily.ai**

If cloudflared runs **inside** Docker Compose (`--profile tunnel`), set URL to `http://frigate:8971` instead of `localhost:8971`.

Copy the tunnel UUID from the tunnel overview (looks like `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`). The Cloudflare-side DNS target is:

```text
<TUNNEL_UUID>.cfargotunnel.com
```

### C. GoDaddy DNS record (when NS stay on DomainControl)

In GoDaddy → **andersonfamily.ai** → **DNS** → **Add record**:

| Field | Value |
| --- | --- |
| Type | **CNAME** |
| Name | `frigate` |
| Value / Points to | `<TUNNEL_UUID>.cfargotunnel.com` |
| TTL | 1 hour (or default) |

Do **not** proxy/orange-cloud anything at GoDaddy (GoDaddy has no CF proxy). Cloudflare receives traffic because the CNAME ends at `cfargotunnel.com`.

Verify:

```powershell
nslookup frigate.andersonfamily.ai
# should show a CNAME to <uuid>.cfargotunnel.com (then Cloudflare IPs)
```

Open https://frigate.andersonfamily.ai — you should reach Frigate login on 8971 (after cloudflared is running).

### D. If Cloudflare already manages DNS (after Option 2)

When adding the Public Hostname `frigate.andersonfamily.ai`, Cloudflare usually creates:

| Type | Name | Target | Proxy |
| --- | --- | --- | --- |
| CNAME | `frigate` | `<TUNNEL_UUID>.cfargotunnel.com` | Proxied (orange cloud) |

Confirm under Cloudflare → **DNS → Records**. No GoDaddy change needed once NS point at Cloudflare.

## One-time setup on the Windows PC

### 1. Start Frigate locally

```powershell
cd home
copy .env.example .env
docker compose up -d
```

Open **http://localhost:8971**. Set the **admin** password under Settings → Users.

### 2. Run cloudflared

**Option A — recommended (native Windows service)**

```powershell
cloudflared service install YOUR_TUNNEL_TOKEN
```

**Option B — Compose profile**

```powershell
# CLOUDFLARE_TUNNEL_TOKEN in .env; dashboard origin http://frigate:8971
docker compose --profile tunnel up -d
```

### 3. (Recommended) Cloudflare Access

Zero Trust → **Access → Applications → Add** → Self-hosted.

- Application domain: `frigate.andersonfamily.ai`
- Policy: allow family emails

Visitors hit Access first, then Frigate’s JWT login.

### 4. Wire the family hub (Vercel)

On **clarus-holdings/andersonfamily-ai** (Production + Preview):

| Variable | Value |
| --- | --- |
| `NEXT_PUBLIC_FRIGATE_URL` | `https://frigate.andersonfamily.ai` |

Redeploy after saving. Local `.env.local` can use the same value.

Hub path: https://www.andersonfamily.ai/security → **Open Frigate UI**.

## go2rtc / cameras later

Placeholders stay `enabled: false` until real RTSP URLs exist. Remote live view uses **MSE** through https://frigate.andersonfamily.ai (port 8971). Keep 8555 on LAN only.

## Tailscale alternative (VPN-only)

Only if you skip the public hostname. Hub button then works only on the tailnet. Prefer `frigate.andersonfamily.ai` so any signed-in family member can open admin from the hub.

## Security checklist

- [ ] DNS CNAME `frigate` → `<TUNNEL_UUID>.cfargotunnel.com`
- [ ] Tunnel public hostname `frigate.andersonfamily.ai` → `localhost:8971` (or `frigate:8971` in Compose)
- [ ] Tunnel targets **8971 only**, never 5000
- [ ] Frigate admin password set
- [ ] Cloudflare Access on `frigate.andersonfamily.ai` (recommended)
- [ ] Vercel `NEXT_PUBLIC_FRIGATE_URL=https://frigate.andersonfamily.ai` + redeploy

## Secrets David must set

| Where | Secret | Purpose |
| --- | --- | --- |
| Windows `home/.env` | `CLOUDFLARE_TUNNEL_TOKEN` | Runs cloudflared |
| Windows Frigate UI | admin password | JWT auth on 8971 |
| Cloudflare Access | family emails | Outer gate for `frigate.andersonfamily.ai` |
| Vercel hub | `NEXT_PUBLIC_FRIGATE_URL=https://frigate.andersonfamily.ai` | Open Frigate button |
| Windows `home/.env` | `FRIGATE_RTSP_PASSWORD` | Camera stream auth later |
| GoDaddy DNS | CNAME `frigate` | Points hostname at the tunnel |
