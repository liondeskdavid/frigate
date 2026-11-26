# Remote access to Frigate (Windows)

**Chosen path: Cloudflare Tunnel → Frigate authenticated UI on port 8971.**

This gives a public HTTPS URL for the full Frigate admin UI (and live views via MSE once cameras exist) without opening router ports. The Anderson Family Hub “Open Frigate” button uses that URL via `NEXT_PUBLIC_FRIGATE_URL`.

Do **not** tunnel port `5000` (unauthenticated). Do **not** invent camera IPs.

## Architecture

```
Browser / andersonfamily.ai
        │  HTTPS
        ▼
Cloudflare Tunnel (+ optional Cloudflare Access)
        │  HTTP to PC
        ▼
Frigate :8971  (JWT login — admin UI + API + proxied streams)
```

- **Local (PC only):** http://localhost:5000 (no auth — LAN/Docker Desktop only)
- **Remote / reverse proxy:** https://YOUR_HOSTNAME → `http://localhost:8971`
- **go2rtc WebRTC (8555):** keep on LAN. Remote live view uses MSE through the tunneled UI; no need to expose 8554/8555.

## One-time setup on the Windows PC

### 1. Start Frigate locally

```powershell
cd home
copy .env.example .env
# edit .env — set a strong note for yourself; Frigate admin password is set in the UI on first use
docker compose up -d
```

Open **https://localhost:8971** (or http://localhost:8971 depending on image TLS). Create / change the **admin** password under Settings → Users. Prefer 8971 for all day-to-day use.

Confirm http://localhost:5000 works only on this PC and is never put behind the tunnel.

### 2. Create a Cloudflare Tunnel

1. Sign in at [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) (free plan is enough).
2. **Networks → Tunnels → Create a tunnel** → Cloudflared → name e.g. `anderson-frigate`.
3. Choose **Windows** and copy the install/token command, or copy the **tunnel token**.
4. Put the token in `home/.env`:

```env
CLOUDFLARE_TUNNEL_TOKEN=eyJ...your_token...
```

5. In the tunnel **Public Hostname** tab, add:

| Field | Value |
| --- | --- |
| Subdomain | e.g. `frigate` (or whatever you own) |
| Domain | your Cloudflare zone (e.g. a domain you control) |
| Type | HTTP |
| URL | `localhost:8971` |

Use **HTTP** to `localhost:8971` because Frigate’s reverse-proxy port is authenticated HTTP (TLS terminates at Cloudflare).

### 3. Run cloudflared (pick one)

#### Option A — recommended on Windows (native service)

Install [cloudflared for Windows](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/downloads/), then:

```powershell
cloudflared service install YOUR_TUNNEL_TOKEN
# or from Zero Trust UI: copy the Windows install command
```

This talks to Frigate at `http://localhost:8971` on the Docker-published port.

#### Option B — Docker Compose profile (same repo)

```powershell
cd home
# CLOUDFLARE_TUNNEL_TOKEN must be set in .env
docker compose --profile tunnel up -d
```

If cloudflared runs **inside** Compose, set the Cloudflare Public Hostname URL to:

`http://frigate:8971`

(not localhost), so the tunnel container can reach the Frigate service on the Docker network.

### 4. (Recommended) Cloudflare Access

Still in Zero Trust: **Access → Applications → Add** → Self-hosted.

- Application domain: the same hostname as the tunnel public hostname
- Policy: allow your family emails (e.g. `@gmail.com` for David / household)

Visitors hit Access first, then Frigate’s own login on 8971.

### 5. Wire the family hub

On Vercel project **clarus-holdings/andersonfamily-ai** (Production + Preview):

| Variable | Example | Notes |
| --- | --- | --- |
| `NEXT_PUBLIC_FRIGATE_URL` | `https://frigate.yourdomain.com` | No trailing slash. Public hostname from the tunnel. Redeploy after changing. |

Local hub `.env.local` can use the same value for testing the Open Frigate button.

After deploy, signed-in users open **Home Security** → **Open Frigate UI**.

## go2rtc / cameras later

Placeholders stay `enabled: false` until real RTSP URLs exist.

When a camera arrives:

1. Add `go2rtc.streams` and enable the camera in `config/config.yml` (see README).
2. Live view in the remote UI uses **MSE** through port 8971 — works over the tunnel with no extra ports.
3. Optional LAN WebRTC: keep host ports `8555/tcp` and `8555/udp` for same-network clients; do not publish them on the Cloudflare hostname.

```yaml
go2rtc:
  webrtc:
    candidates:
      - stun:8555
  streams:
    front_porch:
      - rtsp://viewer:{FRIGATE_RTSP_PASSWORD}@CAMERA_IP:554/...
```

## Tailscale alternative (VPN-only)

If you prefer no public hostname: install [Tailscale](https://tailscale.com/) on the Windows PC and on each device that should open Frigate. Then set:

```env
NEXT_PUBLIC_FRIGATE_URL=http://100.x.y.z:8971
```

(or the MagicDNS name). The hub button only works from devices on the tailnet. Cloudflare Tunnel is preferred so the hub works from any browser after Access + Frigate login.

## Security checklist

- [ ] Tunnel targets **8971 only**, never 5000 / 1984 / 8554
- [ ] Frigate admin password set (Settings → Users)
- [ ] Cloudflare Access policy for family emails (recommended)
- [ ] `NEXT_PUBLIC_FRIGATE_URL` is `https://…` on Vercel
- [ ] No camera credentials committed; use `{FRIGATE_RTSP_PASSWORD}` in config

## Secrets David must set

| Where | Secret | Purpose |
| --- | --- | --- |
| Windows `home/.env` | `CLOUDFLARE_TUNNEL_TOKEN` | Runs cloudflared |
| Windows Frigate UI | admin password | JWT auth on 8971 |
| Cloudflare Zero Trust | Access policy emails | Optional outer gate |
| Vercel hub | `NEXT_PUBLIC_FRIGATE_URL` | Hub Open Frigate button |
| Windows `home/.env` | `FRIGATE_RTSP_PASSWORD` | Camera stream auth later (not UI login) |
