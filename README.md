# WMN System (Edge → Fog → Observability)

This repository is the **master setup guide** for the WMN project. It explains how to deploy the system end-to-end:

- **Edge**: Raspberry Pi (recommended) or WSL (Windows) running `wmn-collector`
- **Fog**: Proxmox VM/CT running `wmn-analyzer` and `wmn-explainer`
- **Observability (cloud)**: a hosted dashboard (Streamlit Cloud recommended; Grafana Cloud optional)

The components communicate through **MQTT**.

---

## Components and links

### GitHub repositories
- `wmn-collector` — https://github.com/irfanuruchi/wmn-collector  
- `wmn-analyzer` — https://github.com/irfanuruchi/wmn-analyzer  
- `wmn-explainer` — https://github.com/irfanuruchi/wmn-explainer  

### Docker Hub images
- `irfanuruchi/wmn-collector` — https://hub.docker.com/r/irfanuruchi/wmn-collector  
- `irfanuruchi/wmn-analyzer` — https://hub.docker.com/r/irfanuruchi/wmn-analyzer  
- `irfanuruchi/wmn-explainer` — https://hub.docker.com/r/irfanuruchi/wmn-explainer  

---

## Data flow and MQTT topics

Typical topic layout:

| Topic | Producer | Consumer | Meaning |
|---|---|---|---|
| `wmn/metrics/<device_id>` | collector | analyzer | raw measurements |
| `wmn/analysis/<device_id>` | analyzer | explainer + dashboard | score/alerts |
| `wmn/explain/<device_id>` | explainer | dashboard | explanation text |

Most subscribers use wildcards: `wmn/metrics/#`, `wmn/analysis/#`, `wmn/explain/#`.

---

## Before you start

### Decide your MQTT broker
You need an MQTT broker reachable by all nodes.

**Option A — HiveMQ Cloud (recommended)**
- Host: your HiveMQ cluster hostname
- Port: `8883`
- TLS: enabled
- Username/password: as created in HiveMQ

**Option B — Local Mosquitto on the fog node**
- Works fully offline/local
- Requires opening access (LAN or Tailscale)

This guide assumes **HiveMQ Cloud**, but the containers work the same with any broker.

### Have these ready
- Docker installed on each node where you run containers
- A way to access your fog node (SSH recommended)
- (Recommended) Tailscale for easy access between Proxmox VMs/CTs

---

# Part A — Edge: Raspberry Pi (wmn-collector)

## Install OS and basic setup
1. Flash **Raspberry Pi OS Lite (64-bit)** to SD card.
2. Boot Pi, login, and update:
   ```bash
   sudo apt update && sudo apt -y upgrade
   sudo reboot
   ```
3. Set hostname (example used in this project):
   ```bash
   sudo raspi-config
   # System Options → Hostname → set e.g. irfanwmn
   sudo reboot
   ```

## A2) Install Docker on Raspberry Pi

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
docker --version
```

## A3) Run the collector image
Pull:
```bash
docker pull irfanuruchi/wmn-collector:latest
```

Run interactively first (first-run config wizard):
```bash
docker run -it --name wmn-collector   --restart unless-stopped   -v wmn_collector_config:/config   irfanuruchi/wmn-collector:latest
```

When prompted, enter your broker details (HiveMQ Cloud example):
- Host: `xxxxx.s1.eu.hivemq.cloud`
- Port: `8883`
- TLS: `1`
- Username: your HiveMQ username
- Password: your HiveMQ password
- Publish base: usually `wmn/metrics`

The container saves settings to `/config/config.env` inside the Docker volume.

## A4) Verify collector is publishing
Check logs:
```bash
docker logs -f wmn-collector
```

---

# Part B — Edge alternative: Windows (WSL) running wmn-collector

> Use this if you want to demo from your laptop or you don’t have the Pi available.

## B1) Install Docker Desktop + WSL2
- Enable WSL2 on Windows
- Install Docker Desktop
- In Docker Desktop settings, enable integration for your WSL distro

## B2) Run the collector in WSL
Inside WSL:
```bash
docker pull irfanuruchi/wmn-collector:latest

docker run -it --name wmn-collector   --restart unless-stopped   -v wmn_collector_config:/config   irfanuruchi/wmn-collector:latest
```

---

# Part C — Fog: Proxmox deployment (wmn-analyzer + wmn-explainer)

You can run these in:
- **VMs** (simplest, especially for GPU)
- **CTs** (fine for analyzer; explainer recommended on VM if GPU passthrough)

## C1) Create the fog VM/CT and install Docker
Inside the fog machine (VM or CT):
```bash
sudo apt update && sudo apt -y upgrade
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
```

---

## C2) Run wmn-analyzer
Pull:
```bash
docker pull irfanuruchi/wmn-analyzer:latest
```

Run interactive first:
```bash
docker run -it --name wmn-analyzer   --restart unless-stopped   -v wmn_analyzer_config:/config   irfanuruchi/wmn-analyzer:latest
```

Use:
- Broker host/port/TLS/username/password: same as collector
- Subscribe topic: `wmn/metrics/#`
- Publish base: `wmn/analysis`

Verify logs:
```bash
docker logs -f wmn-analyzer
```

---

## C3) Run wmn-explainer (GPU optional)

### C3.1 CPU-only run
```bash
docker pull irfanuruchi/wmn-explainer:latest

docker run -it --name wmn-explainer   --restart unless-stopped   -p 8000:8000   -v wmn_explainer_config:/config   -v ollama_models:/root/.ollama   -e OLLAMA_MODEL=phi3:mini   irfanuruchi/wmn-explainer:latest
```

### C3.2 NVIDIA GPU run (VM with passthrough)
First confirm GPU works inside the VM:
```bash
nvidia-smi
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

Then run explainer:
```bash
docker rm -f wmn-explainer 2>/dev/null || true

docker run -it --name wmn-explainer   --restart unless-stopped   --gpus all   -p 8000:8000   -v wmn_explainer_config:/config   -v ollama_models:/root/.ollama   -e OLLAMA_MODEL=llama3.2:3b   irfanuruchi/wmn-explainer:latest
```

### C3.3 Verify explainer API
On the fog node:
```bash
curl -s http://localhost:8000/docs | head
```

---

# Part D — Networking between fog VMs/CTs: Tailscale (recommended)

Tailscale makes it easy to:
- access the explainer API from your laptop (`http://<tailscale-ip>:8000/docs`)
- avoid Proxmox NAT/port-forwarding
- use stable names (MagicDNS)

## D1) Install Tailscale on a VM/CT

On each machine you want in the tailnet (analyzer, explainer, dashboard VM):
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Optional: set a hostname:
```bash
sudo tailscale up --hostname wmn-explainer
```

Check IP:
```bash
tailscale ip -4
```

## D2) Enable MagicDNS (optional but recommended)
In the Tailscale admin panel:
- DNS → enable **MagicDNS**

Then you can reach:
- `http://wmn-explainer:8000/docs`

> CT note: if you install Tailscale inside an LXC container, the container must have `/dev/net/tun` available. (VMs work without special steps.)

---

# Part E — Cloud dashboard (hosted)

The system works without a dashboard, but a cloud dashboard makes visualization easier.

## Option 1 (recommended): Streamlit Cloud
Streamlit Cloud is a good fit for:

- a single page showing **score + alerts + latest explanation text**
- minimal setup

**Typical approach**
- Host a small Streamlit app in a repo (e.g. `wmn-dashboard`)
- The app consumes data via one of these patterns:
  1) a small “bridge” service on the fog node that exposes the latest values over HTTP
  2) a webhook-style endpoint that receives updates and stores them
  3) (advanced and optional) a managed MQTT-to-HTTP connector

If you want the cleanest setup for this project, use:
- Streamlit app reads from a simple HTTP endpoint you host on the fog node (reachable by Tailscale).

## Option 2: Grafana Cloud
Grafana Cloud is better for charts but usually needs an extra step:
- MQTT → InfluxDB/Prometheus (or an HTTP bridge)
- then Grafana reads from that data source
You can still show explanation text using a table/text panel.

---

# Part F — Final end-to-end checks

1) Collector is running and publishing:

```bash
docker logs --tail 50 wmn-collector
```

2) Analyzer is subscribing and publishing analysis:
```bash
docker logs --tail 50 wmn-analyzer
```

3) Explainer is subscribed and API reachable:
```bash
docker logs --tail 50 wmn-explainer
curl -s http://localhost:8000/docs | head
```

4) If using Tailscale, open from laptop:
- `http://<tailscale-ip>:8000/docs` (or `http://wmn-explainer:8000/docs` with MagicDNS)

---

## Troubleshooting

### “No space left on device” on fog VM
```bash
docker system df
docker system prune -a
```
Model caching uses disk (the `ollama_models` volume).

### Container name already in use
```bash
docker rm -f wmn-explainer
```

### GPU not visible in containers
Verify NVIDIA container runtime:
```bash
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi
```

---

## Notes
- Components stay independent and communicate via MQTT.
- The fog node hosts the analysis and explanation services.
- The dashboard is optional and can be hosted in the cloud for easier access.
