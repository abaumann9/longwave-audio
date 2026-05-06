# Server Setup

This guide covers installing and configuring the server-side components using Docker. The [`sweisgerber/snapcast`](https://hub.docker.com/r/sweisgerber/snapcast) image bundles Snapserver, librespot, and shairport-sync (AirPlay) into a single container.

> For alternative installation methods (bare metal, packages, building from source), see the [Snapcast documentation](https://github.com/snapcast/snapcast) and [librespot documentation](https://github.com/librespot-org/librespot).

## Prerequisites

- A Linux machine (Raspberry Pi, old laptop, NAS, VM, etc.)
- Docker and Docker Compose installed
- Network connectivity to the Wi-Fi HaLow AP (wired or bridged)
- A Spotify Premium account (for Spotify Connect support)

## 1. Create the Directory Structure

```bash
sudo mkdir -p /opt/services/config/snapcast/{config,data}
sudo mkdir -p /opt/services/config/librespot
sudo chown -R 1000:1000 /opt/services/config/snapcast /opt/services/config/librespot
```

## 2. Create `docker-compose.yml`

Create a file at your preferred location (e.g. `/opt/services/docker-compose.yml`):

```yaml
services:
  snapcast:
    image: docker.io/sweisgerber/snapcast:latest
    hostname: snapcast
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - START_SNAPCLIENT=true
      # --host: name or ip of compose service or dockerhost
      # --soundcard: <ID> from `snapclient -l` from inside the container
      # - SNAPCLIENT_OPTS=--host snapcast --soundcard <ID>
      #   => Don't use quotes for SNAPCLIENT_OPTS="" !
      # - HOST_AUDIO_GROUP=<AUDIO-GID> # set to GID of host audio group
      - START_AIRPLAY=true
    restart: "unless-stopped"
    # If not running network_mode: host 
    # ports:
    #   - 1704:1704   # Snapcast audio streaming
    #   - 1705:1705   # Snapcast TCP JSON-RPC
    #   - 1780:1780   # SnapWeb web interface
    #   - 22382:22382 # Spotify Connect (librespot)
      - 4953:4953   # Custom streaming app
    # devices:
      # - /dev/snd:/dev/snd # optional, only if you want to use snapclient on the host
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /opt/services/config/snapcast/config:/config/
      - /opt/services/config/snapcast/data:/data/
      - /opt/services/config/librespot:/var/cache/librespot
      # /audio can be used for FIFOs for audio playback from mpd/mopidy/host/etc
```

## 3. Configure mDNS

Configure Avahi on the Docker host to advertise Snapcast's services on the local network:

> This is not the only way to do this and its not strictly required but it will simplify the snapclient configuration.

```bash
sudo apt install avahi-daemon
sudo systemctl enable --now avahi-daemon
```

Create a service definition file with all Snapcast services:

```bash
sudo tee /etc/avahi/services/snapcast.service << 'EOF'
<?xml version="1.0" standalone='no'?>
<!DOCTYPE service-group SYSTEM "avahi-service.tld">
<service-group>
  <name>Snapcast</name>
  <service>
    <type>_snapcast._tcp</type>
    <port>1704</port>
  </service>
  <service>
    <type>_snapcast-ctrl._tcp</type>
    <port>1705</port>
  </service>
  <service>
    <type>_snapcast-http._tcp</type>
    <port>1780</port>
  </service>
</service-group>
EOF

sudo systemctl restart avahi-daemon
```

## 4. Start the Container

```bash
cd /opt/services
docker compose up -d
```

Check that it's running:

```bash
docker compose logs -f snapcast
```

You should see Snapserver, librespot, and (if enabled) shairport-sync starting up.

## 5. Customize Snapserver Config

> The full default config with all options is at [`snapcast/server/etc/snapserver.conf`](https://github.com/snapcast/snapcast/blob/develop/server/etc/snapserver.conf). Start from that and adjust the settings below.

The container reads `/config/snapserver.conf` (mapped to `/opt/services/config/snapcast/config/snapserver.conf` on the host). Grab the [upstream default config](https://github.com/snapcast/snapcast/blob/develop/server/etc/snapserver.conf), save it there, and adjust the settings below.

```bash
docker compose restart snapcast  # after making changes
```

### Key settings to change

**Audio source** — Use Snapserver's built-in librespot source so you don't need a separate librespot process or FIFO pipe:

```ini
[stream]
source = librespot://librespot?name=LibreSpot&devicename=LibreSpot&bitrate=320&sampleformat=44100:16:2&cache=%2Fvar%2Fcache%2Flibrespot%2F&params=--zeroconf-port%2022382
```

Breaking down the parameters:

| Parameter | Value | Purpose |
|---|---|---|
| `name` | `LibreSpot` | Stream name in Snapserver |
| `devicename` | `LibreSpot` | Device name shown in Spotify |
| `bitrate` | `320` | Spotify streaming quality (96/160/320) |
| `sampleformat` | `44100:16:2` | CD quality (44.1 kHz, 16-bit, stereo) |
| `cache` | `/var/cache/librespot/` | Where OAuth tokens are cached (URL-encoded) |
| `params` | `--zeroconf-port 22382` | Port for Spotify Connect discovery (must match compose port mapping if using) |

> The `--zeroconf-port 22382` setting is required — it tells librespot which port to use for Spotify Connect device discovery on your network. This must match the `22382:22382` port mapping in the Docker Compose file, otherwise Spotify apps won't find the device.

See the [Snapcast stream configuration docs](https://github.com/snapcast/snapcast/blob/develop/doc/configuration.md) for other source types (AirPlay, pipe, ALSA, etc.) and the [librespot options](https://github.com/librespot-org/librespot/wiki/Options) for all available parameters.

**Codec** —use a compressed codec to reduce bandwidth:

```ini
codec = flac    # lossless, ~600-900 kbps (recommended)
# codec = opus  # lossy, ~64-256 kbps (use if bandwidth is tight)
# codec = pcm   # uncompressed, ~1.4 Mbps
```

**Buffer** — This is the amount of time clients will buffer data. The default is 1000ms
- If you connection is strong and network is stable try reducing down to 600 ~ 700ms for more responsivness
- If you are experiencing dropouts / resyncing (if using the xaio audio hat it will flash orange when this happens) try increase to 1500ms

```ini
buffer = 1000
```

## 4. Spotify OAuth Authentication

librespot requires a one-time OAuth login to link your Spotify account. Since the server is typically headless, you'll complete the browser step on a different device.

> For the most up-to-date instructions, see the [librespot Headless OAuth documentation](https://github.com/librespot-org/librespot/wiki/Options#headless-oauth).

### Summary

1. Watch the container logs for the OAuth URL:

   ```bash
   docker compose logs -f snapcast
   ```

   You'll see something like:

   ```
   Go to this URL to authorize Spotify: https://accounts.spotify.com/authorize?...
   ```

2. Open that URL in a browser on **any device** (phone, laptop — doesn't need to be the server).

3. Log in with your **Spotify Premium** account and click **Agree**.

4. Spotify redirects to a `http://127.0.0.1:...` URL. If the container is listening on the redirect port (22382 is mapped in the compose file), auth completes automatically. If the redirect page fails to load, copy the **full URL** from the browser address bar (including the `?code=...` parameter) and paste it back into the librespot prompt.

5. Once authenticated, credentials are cached in `/var/cache/librespot` (mapped to `/opt/services/config/librespot` on the host). You only need to do this once — the token refreshes automatically.

6. Verify: open **Spotify**, tap **Devices**, and look for the LongWave device.

### Troubleshooting

- **No auth URL in logs**: Credentials may already be cached. Check if the device appears in Spotify.
- **Token expired / device gone**: Clear the cache and restart:
  ```bash
  rm -rf /opt/services/config/librespot/*
  docker compose restart snapcast
  ```
  Then repeat the OAuth flow.

## 6. Verify Everything

1. Open the SnapWeb interface at `http://<server-ip>:1780`
2. Open Spotify and select the LongWave/Snapcast device
3. Play a track — you should see the stream appear in SnapWeb and hear audio if `START_SNAPCLIENT=true`
