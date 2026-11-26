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
| http://localhost:5000 | Local Frigate UI (unauthenticated; LAN use) |
| https://localhost:8971 | Authenticated UI / API (accept self-signed cert) |
| http://localhost:1984 | go2rtc web UI |
| mqtt://localhost:1883 | Mosquitto (no auth in this scaffold) |

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

- **Same PC / LAN:** set hub env `NEXT_PUBLIC_FRIGATE_URL=http://localhost:5000` (or your PC’s LAN IP, e.g. `http://192.168.x.x:5000`).
- **Remote hub → home Frigate:** Frigate is not public. Use a VPN, Tailscale, or reverse proxy with auth, then set `NEXT_PUBLIC_FRIGATE_URL` to that reachable HTTPS URL. Do not expose port 5000 to the internet without authentication (prefer 8971 + TLS).

## Troubleshooting

- **UI not loading:** `docker compose logs frigate` — first boot pulls a large image.
- **Config errors:** Frigate validates YAML on start; fix `config/config.yml` and restart.
- **Windows path / volume issues:** run Compose from `home/` so `./config` and `./storage` resolve correctly.
- **GPU / Coral:** leave disabled until you have hardware; this scaffold uses the CPU detector.
