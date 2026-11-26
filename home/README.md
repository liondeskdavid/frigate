# Anderson home Frigate stack

Runnable Frigate + MQTT for David Anderson’s Windows PC (Docker Desktop). Cameras are **disabled placeholders** so the stack starts without hardware.

This folder is the **home deploy source of truth**. The rest of this repository is the upstream Frigate project fork.

## Prerequisites (Windows)

1. [Docker Desktop](https://www.docker.com/products/docker-desktop/) with WSL2 backend
2. Enough free disk for the `storage/` volume (recordings later)
3. No cameras required to start

## Quick start (no cameras)

```powershell
cd home
copy .env.example .env
docker compose up -d
```

Wait ~30–60s for the first image pull, then open:

| URL | Purpose |
| --- | --- |
| http://localhost:5000 | Local Frigate UI (**unauthenticated** — PC/LAN only; never tunnel) |
| http://localhost:8971 | Authenticated admin UI / API (**use this for remote / Cloudflare Tunnel**) |
| http://localhost:1984 | go2rtc web UI (LAN) |
| mqtt://localhost:1883 | Mosquitto (no auth in this scaffold) |

Set the Frigate **admin** password in the UI (Settings → Users) on port 8971.

### Remote access (Cloudflare Tunnel)

Full Windows steps: **[REMOTE-ACCESS.md](REMOTE-ACCESS.md)**.

Summary: Cloudflare Tunnel → `localhost:8971` → HTTPS hostname → set hub `NEXT_PUBLIC_FRIGATE_URL` on Vercel. Optional Compose profile:

```powershell
# after CLOUDFLARE_TUNNEL_TOKEN is in .env and the dashboard origin is http://frigate:8971
docker compose --profile tunnel up -d
```

Check health:

```powershell
docker compose ps
docker compose logs -f frigate
```

You should see Frigate running with **no active cameras** (placeholders are `enabled: false`).

Stop:

```powershell
docker compose down
```

## Ports

| Host | Container | Service |
| --- | --- | --- |
| 5000 | 5000 | Frigate UI (local) |
| 8971 | 8971 | Frigate authenticated UI/API |
| 8554 | 8554 | go2rtc RTSP restream |
| 8555 | 8555 tcp/udp | WebRTC |
| 1984 | 1984 | go2rtc API/UI |
| 1883 | 1883 | MQTT |

## Volumes

| Path | Role |
| --- | --- |
| `./config` | `config.yml` and Frigate DB/models |
| `./storage` | Recordings / clips / snapshots (`/media/frigate`) |
| tmpfs `/tmp/cache` | Short-lived recording segments |
| Docker volumes `mqtt_*` | Mosquitto data/logs |

## Environment variables

See `.env.example`:

- `TZ` — default `America/Denver`
- `FRIGATE_RTSP_PASSWORD` — substituted into camera RTSP URLs as `{FRIGATE_RTSP_PASSWORD}` (camera stream auth, not Frigate login)
- `PLUS_API_KEY` — optional Frigate+

## Adding cameras later

1. Get the camera’s RTSP URL from the vendor (do not invent IPs).
2. In `config/config.yml`:
   - Add a `go2rtc.streams` entry (recommended) pointing at the real RTSP URL
   - Point the camera `ffmpeg.inputs` at `rtsp://127.0.0.1:8554/<stream_name>`
   - Set `enabled: true` on that camera
   - Turn on `detect` / `record` / `snapshots` as needed
3. Reload config from the Frigate UI, or `docker compose restart frigate`

Example (replace placeholders with real values):

```yaml
go2rtc:
  streams:
    front_porch:
      - rtsp://viewer:{FRIGATE_RTSP_PASSWORD}@CAMERA_IP:554/Streaming/Channels/101
    front_porch_sub:
      - rtsp://viewer:{FRIGATE_RTSP_PASSWORD}@CAMERA_IP:554/Streaming/Channels/102

cameras:
  front_porch:
    enabled: true
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/front_porch
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/front_porch_sub
          roles:
            - detect
```

## Hub connection (andersonfamily.ai)

The family hub Security page (`/security`) explains stack status and links to Frigate when available.

1. Follow **[REMOTE-ACCESS.md](REMOTE-ACCESS.md)** (Cloudflare Tunnel to **8971**).
2. On Vercel (`clarus-holdings/andersonfamily-ai`), set:

```bash
NEXT_PUBLIC_FRIGATE_URL=https://frigate.yourdomain.com
```

3. Redeploy the hub. Signed-in users get **Open Frigate UI** on `/security`.

Never point the hub at port 5000 over the internet. Tailscale to `:8971` is a VPN-only alternative (see REMOTE-ACCESS.md).

## Troubleshooting

- **UI not loading:** `docker compose logs frigate` — first boot pulls a large image.
- **Config errors:** Frigate validates YAML on start; fix `config/config.yml` and restart.
- **Windows path / volume issues:** run Compose from `home/` so `./config` and `./storage` resolve correctly.
- **GPU / Coral:** leave disabled until you have hardware; this scaffold uses the CPU detector.
