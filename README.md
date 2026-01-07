# Project Zomboid + Tailscale Docker Server

This repo provides a small setup for running a private Project Zomboid dedicated server using:

- Docker / Docker Compose
- [indifferentbroccoli/projectzomboid-server-docker] as the game server image
- [tailscale/tailscale] as a sidecar container for networking

The Zomboid server is only reachable over your Tailscale tailnet, which makes it easy to host a friends‑only server without exposing ports to the public internet.

## Prerequisites

- Docker and Docker Compose installed
- A Tailscale account
- Access to create a Tailscale auth key

## Getting a Tailscale auth key

To populate the `TS_AUTHKEY` value in your `.env`:

1. Sign in to the Tailscale admin console in your browser.
2. Navigate to the **Keys** section (it may be under **Settings** or **Access controls**, depending on your account type).
3. Create a new auth key suitable for servers (e.g. a reusable key or a key with an appropriate expiry and tags).
4. Copy the generated key and paste it into `TS_AUTHKEY` in your `.env` file.

You can revoke or rotate this key from the Tailscale admin console at any time.

## Directory layout

- `docker-compose.yml` – defines the Tailscale sidecar and the Project Zomboid server
- `server-files/` – game files (server install, workshop content, etc.)
- `server-data/` – server configuration and save data (`servertest.ini`, world saves, etc.)
- `tailscale/` – Tailscale state directory (created by the container)

`server-files/`, `server-data/`, and `tailscale/` are all bind‑mounted into the containers so that your world and config survive image updates and container restarts.

## Quick start

1. **Clone this repo**

   ```bash
   git clone <your-repo-url>
   cd pz-b42
   ```

2. **Create your `.env` from the example**

   ```bash
   cp .env.example .env
   ```

3. **Edit `.env`**

   - Set `TS_AUTHKEY` to a valid Tailscale auth key
   - Change all the `changeme_*` passwords (`PASSWORD`, `RCON_PASSWORD`, `ADMIN_PASSWORD`)
   - Optionally tweak gameplay flags like `PVP`, `PAUSE_EMPTY`, `GLOBAL_CHAT`

   On first run, the Project Zomboid container will use these values to generate a default `servertest.ini` when `GENERATE_SETTINGS=true`.

4. **Start the server**

   Using the newer Docker Compose plugin:

   ```bash
   docker compose up -d
   ```

   Or with the legacy `docker-compose` binary:

   ```bash
   docker-compose up -d
   ```

   This will start two containers:

   - `ts-pz` – Tailscale sidecar, exposes the server inside your tailnet
   - `pz-b42` – Project Zomboid dedicated server, sharing the network stack with `ts-pz`

5. **Check logs (optional)**

   ```bash
   docker compose logs -f tailscale
   docker compose logs -f projectzomboid
   ```

   Wait for Tailscale to come up and connect, and for the Zomboid server to finish its initial install and config generation.

## Connecting to the server

Any machine that wants to join your server must:

- Be logged into Tailscale with an account that has been **invited to your tailnet** (or otherwise allowed to join it), and
- Have the Tailscale client installed, running and connected.

Once that’s true for a given client machine:

1. Log into Tailscale on your client machine (and ensure it shows as connected in the Tailscale UI).
2. Once the containers are running, find the Tailscale IP of the `ts-pz` container host in the Tailscale admin console or with `tailscale status` on the host.
3. In Project Zomboid, add a new server:
   - Address: the Tailscale IP of the host
   - Port: `16261` (default, unless you changed it via env/ini)
   - Server password: the `PASSWORD` you set in `.env`

Because the game server shares the network namespace with the Tailscale container (`network_mode: "service:tailscale"), the game ports are reachable directly via that Tailscale IP **but only to devices that are part of your tailnet and currently connected to Tailscale**.

## Managing data and updates

- **World data & config**: stored under `server-data/`
- **Server files & Workshop content**: stored under `server-files/`

You can safely update the server image by pulling the latest image and recreating containers; your saves and config remain in these directories.

```bash
# Pull latest images
docker compose pull

# Recreate containers with the new images
docker compose up -d
```

To stop the server:

```bash
docker compose down
```

This shuts down the containers but leaves your data in `server-data/`, `server-files/`, and `tailscale/` intact.

## Environment variables

This setup uses a shared `.env` file for both services.

### Tailscale

- `TS_AUTHKEY` – **required**; Tailscale auth key used to bring the sidecar online
- `TS_EXTRA_ARGS` – optional; any extra flags to pass to `tailscale up`

The compose file also sets:

- `TS_STATE_DIR=/var/lib/tailscale`
- `TS_USERSPACE=false`

You normally do not need to change those.

### Project Zomboid

The most important variables (see `.env.example` for defaults):

- `SERVER_NAME`, `SERVER_PUBLIC_NAME`, `SERVER_DESCRIPTION`
- `PASSWORD` – in‑game server password
- `RCON_PASSWORD` – password for RCON admin tools
- `ADMIN_USERNAME`, `ADMIN_PASSWORD` – in‑game admin account
- `PVP`, `PAUSE_EMPTY`, `GLOBAL_CHAT` – basic gameplay toggles
- `GENERATE_SETTINGS` – when `true`, generate `servertest.ini` from env vars on first run

For advanced tuning (XP multipliers, loot, zombies, etc.), refer to the upstream `indifferentbroccoli/projectzomboid-server-docker` documentation. Those additional env vars will also work here.

## Notes

- This repo is intentionally minimal: no automated backups, workshop management, or mod lists are handled here.
- Consider setting up periodic backups of `server-data/` and `server-files/` if you care about your world.
- Feel absolutely free to reach out to me in case of any comment!
