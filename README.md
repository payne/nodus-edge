<!-- RELEASE: nodusrf/nodus-edge README.md -->
# NodusNet Edge Node

Run a local RF monitoring station for the NodusNet ecosystem.

## What You Need

- **RTL-SDR dongle** (any RTL2832U-based dongle)
- **Antenna** tuned for the band you want to monitor
- **Linux host** (Raspberry Pi, mini PC, or any x86_64/arm64 machine)

## Install

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/nodusrf/nodus-edge/main/install.sh)"
```

The installer handles everything: Docker, USB permissions, frequency selection, and container deployment. Takes about 5 minutes.

## After Install

```bash
cd ~/nodusedge

# View logs
docker compose logs -f

# Dashboard
# http://localhost:8073

# Update
docker compose pull && docker compose up -d

# Stop
docker compose down
```

## Links

- [NodusRF](https://nodusrf.com)
- [Discord](https://discord.gg/QqeERd7Ek)
