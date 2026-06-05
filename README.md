# 🖥️ Home Server Grafana Dashboards

[![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)](https://grafana.com)
[![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)](https://prometheus.io)
[![Home Assistant](https://img.shields.io/badge/Home_Assistant-41BDF5?style=for-the-badge&logo=homeassistant&logoColor=white)](https://home-assistant.io)
[![Tailscale](https://img.shields.io/badge/Tailscale-242424?style=for-the-badge&logo=tailscale&logoColor=white)](https://tailscale.com)

A collection of Grafana dashboards for monitoring a home server stack — room temperatures via Home Assistant, system-level PC metrics via Prometheus node_exporter, and Tailscale VPN health.

---

## 📡 Monitoring Architecture

```
┌─────────────────────────────────────────────────────────┐
│                   Grafana (Visualization)                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Home        │  │  PC          │  │  Tailscale   │  │
│  │  Assistant   │  │  Monitoring  │  │  Health      │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼─────────────────┼─────────────────┼───────────┘
          │                 │                 │
          ▼                 ▼                 ▼
┌─────────────────────────────────────────────────────────┐
│              Prometheus (Metrics Database)               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ HA Exporter  │  │node_exporter │  │Tailscale     │  │
│  │ (built-in)   │  │              │  │Metrics       │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼─────────────────┼─────────────────┼───────────┘
          │                 │                 │
          ▼                 ▼                 ▼
   Home Assistant      Linux Hosts      Tailscale
   (climate sensors)   (CPU, RAM, disk) (VPN mesh)
```

---

## 📊 Dashboards

### 1. Home Assistant — Room Temperatures

**File:** `grafana-dashboard.json` | **Refresh:** 1m

> Track temperatures across your home with real-time stats and historical trends.

| Panel | Type | What it shows |
|-------|------|---------------|
| **Room Temperatures Over Time** | Time Series | All AC units' temperatures plotted on one timeline with multi-tooltip |
| **AC Escritório — Temp** | Stat (bg color) | Office A/C current temp — blue/green/yellow/red thresholds |
| **AC Sala — Temp** | Stat (bg color) | Living room A/C current temp |
| **AC Quarto — Temp** | Stat (bg color) | Bedroom A/C current temp |
| **AC Escritório — Status** | Stat (bg color) | Office A/C ON/OFF — green when running, red when off |
| **AC Sala — Status** | Stat (bg color) | Living room A/C ON/OFF |
| **AC Quarto — Status** | Stat (bg color) | Bedroom A/C ON/OFF |

**Key Metrics:**
- `hass_climate_current_temperature_celsius` — per-room temperature
- `hass_climate_mode` — ON/OFF status per AC unit
- Color thresholds: <20°C blue | 20-25°C green | 25-28°C yellow | >28°C red

#### Preview

<!-- Replace with actual screenshot -->
<!-- ![Home Assistant Dashboard](screenshots/ha-dashboard.png) -->
```
┌──────────────────────────────────────────────────────────┐
│  Home Assistant Dashboard          Refresh: 1m           │
├──────────────────────────────────────────────────────────┤
│  ┌─ Temperatures ──────────────────────────────────────┐ │
│  │  Room Temperatures Over Time                        │ │
│  │  ┌──────────────────────────────────────┐ 35°C ─── │ │
│  │  │  ╱╲    ╱╲    ╱╲                      │ 30°C ─── │ │
│  │  │ ╱  ╲  ╱  ╲  ╱  ╲  ╱╲               │ 25°C ─── │ │
│  │  │╱    ╲╱    ╲╱    ╲╱  ╲╱╲            │ 20°C ─── │ │
│  │  └──────────────────────────────────────┘ 15°C ─── │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐            │ │
│  │  │Escritório│ │   Sala   │ │  Quarto  │            │ │
│  │  │  24°C    │ │  26°C    │ │  22°C    │            │ │
│  │  │  🟢 ON   │ │  🟢 ON   │ │  🔴 OFF  │            │ │
│  │  └──────────┘ └──────────┘ └──────────┘            │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

### 2. PC Monitoring

**File:** `grafana-pc-dashboard.json` | **Refresh:** 15s

> Full system monitoring for your Linux machines — CPU, RAM, disk, temperatures, and network.

| Section | Panel | Type |
|---------|-------|------|
| **CPU** | CPU Usage Over Time | Time Series (overall + per-core) |
| | CPU Usage | Stat (bg color: green/yellow/red) |
| **Memory** | RAM Usage Over Time | Time Series (% used) |
| | RAM Used | Gauge (% — green/yellow/red) |
| | RAM Used (bytes) | Gauge (absolute) |
| | RAM Available | Stat (bg color) |
| **Temperatures** | Temperatures Over Time | Time Series (CPU + NVMe sensors) |
| **Disk** | Disk Usage | Table (per mount, % + bytes) |
| | Disk I/O | Time Series (read/write throughput) |
| **Network** | Network Traffic | Time Series (RX/TX per interface) |
| | Network Errors | Time Series (error/drop rates) |
| **System** | Uptime | Stat |
| | Load Average | Time Series (1/5/15 min) |
| | Running Processes | Stat |
| | Top Processes | Table (sorted by CPU or RAM) |

**Templating:**
- Drop-down selector for **data source** (supports multiple Prometheus instances)
- Drop-down selector for **instance** (pick which machine to view)

**Key Metrics:**
- `node_cpu_seconds_total` → CPU usage %
- `node_memory_MemAvailable_bytes / MemTotal_bytes` → RAM %
- `node_hwmon_temp_celsius` → CPU/NVMe temps
- `node_disk_io_time_seconds_total` → Disk I/O
- `node_network_receive/transmit_bytes_total` → Network throughput

#### Preview

```
┌──────────────────────────────────────────────────────────┐
│  PC Monitoring           Instance: [pandora ▼]  15s     │
├──────────────────────────────────────────────────────────┤
│  ┌─ CPU ──────────────────────────────────────────────┐ │
│  │  CPU Usage Over Time       [████████░░]  23%  🟢   │ │
│  │  ┌──────────────────────────────────────┐  100%    │ │
│  │  │       ╱╲                            │          │ │
│  │  │  ╱╲  ╱  ╲  ╱╲                      │   0%    │ │
│  │  └──────────────────────────────────────┘         │ │
│  ├─ Memory ──────────────────────────────────────────┤ │
│  │  RAM: [████████████████░░░░░░]  62%  🟡  12.4/32GB│ │
│  ├─ Temperatures ────────────────────────────────────┤ │
│  │  CPU: 52°C  🟢  │  NVMe: 38°C  🟢                │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

### 3. Tailscale Health

**File:** `grafana-tailscale-dashboard.json` | **Refresh:** 15s

> Monitor your Tailscale VPN mesh — node status, peer connections, bandwidth, and P2P vs relay quality.

| Panel | Type | What it shows |
|-------|------|---------------|
| **Node Online** | Stat (bg: red/green) | Is this node connected to Tailscale? |
| **Peers Online** | Stat (bg: blue/green) | How many peers are currently reachable |
| **Total Peers** | Stat | Total peers in the tailnet |
| **Exit Node** | Stat (bg: blue/orange) | Is exit node functionality active? |
| **Traffic In (total)** | Stat | Total bytes received over `tailscale0` |
| **Traffic Out (total)** | Stat | Total bytes sent over `tailscale0` |
| **Tailscale Bandwidth** | Time Series | Inbound/outbound throughput (Bps) |
| **Packets per Second** | Time Series | Packet rate (in/out) |
| **Receive Path** | Time Series | **Direct P2P (green)** vs **DERP relay (red)** |
| **Send Path** | Time Series | **Direct P2P (green)** vs **DERP relay (red)** |

**Key Metrics:**
- `tailscale_node_online` — 0/1 node connectivity
- `tailscale_peers_online` — peer count
- `magicsock_netmap_num_peers` — total tailnet size
- `magicsock_recv_data_bytes_{ipv4,ipv6,derp}` — direct vs relay traffic
- `node_network_{receive,transmit}_bytes_total{device="tailscale0"}` — bandwidth

#### Preview

```
┌──────────────────────────────────────────────────────────┐
│  Tailscale Health                          Refresh: 15s  │
├──────────────────────────────────────────────────────────┤
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐   │
│  │  Node    │ │  Peers   │ │  Total   │ │  Exit    │   │
│  │  🟢 ON   │ │  3/3 🟢  │ │   3      │ │  🔵 OFF  │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘   │
│  ┌─ Direct vs DERP ───────────────────────────────────┐ │
│  │  Direct (P2P) ━━━━━━━━━━━━━━━━━━━━━━━━━━ 🟢 98%   │ │
│  │  DERP Relay  ────────────────────────────── 🔴 2%  │ │
│  └──────────────────────────────────────────────────────┘ │
│  ┌─ Tailscale Bandwidth ───────────────────────────────┐ │
│  │  ┌──────────────────────────────────────┐   1 Mbps  │ │
│  │  │  ╱╲     ╱╲     ╱╲     ╱╲            │           │ │
│  │  │ ╱  ╲ ╱  ╲  ╱╲ ╱  ╲  ╱  ╲          │   0      │ │
│  │  └──────────────────────────────────────┘          │ │
│  │  Inbound──── Outbound - - -                         │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
```

---

## 🚀 How to Import

### In Grafana UI:
1. Go to **Dashboards → New → Import**
2. Paste the JSON content or upload the file
3. Select your Prometheus data source
4. Click **Import**

### Using the API:
```bash
curl -X POST "https://grafana.yourdomain/api/dashboards/db" \
  -H "Authorization: Bearer $GRAFANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d @grafana-dashboard.json
```

> **Note:** The PC dashboard uses templating variables (`$datasource`, `$instance`) — make sure your Prometheus has `node_exporter` metrics. The Tailscale dashboard requires the [Tailscale Prometheus metrics](https://tailscale.com/kb/1282/metrics) to be enabled.

---

## 🛠️ Dependencies

| Dashboard | Requires | Exporter |
|-----------|----------|---------|
| Home Assistant | Prometheus + Home Assistant | Built-in Prometheus integration in HA |
| PC Monitoring | Prometheus | [`node_exporter`](https://github.com/prometheus/node_exporter) |
| Tailscale Health | Prometheus | Tailscale's built-in metrics endpoint |

---

## 📁 File Structure

```
grafana-dashboards/
├── README.md                         ← You are here
├── grafana-dashboard.json            ← Home Assistant temps
├── grafana-pc-dashboard.json         ← Full PC monitoring
├── grafana-tailscale-dashboard.json  ← Tailscale VPN health
└── postmortem-grafana-2026-05.md     ← Outage post-mortem
```

---

## 📝 License

These dashboards are shared for reference. Feel free to use, modify, and adapt them for your own home server.
