# 6. Monitoring the ARX Node & the Arcium Cluster

This chapter explains how to build a **complete monitoring system** for your ARX node and the supporting infrastructure.

Arcium does **not** expose a native Prometheus metrics endpoint, so node operators must build monitoring using:

- `arcium-cli` outputs  
- Solana RPC health & latency checks  
- Node Exporter + Textfile Collector  
- Prometheus + Grafana dashboards  

This document describes a **production‑grade custom Prometheus exporter**, identical to what professional validator operators use.

It monitors:

- ARX node on‑chain status  
- RPC health  
- RPC latency  
- Slot progression  
- Reference slot comparison  
- Cluster health (synthetic metric)  

---

## 6.1 Goals of the Monitoring System

The monitoring system must answer — in real time — the following:

### ✔ Is my ARX node active on‑chain?  
→ `arcium arx-active <offset>`

### ✔ Is the Solana RPC healthy?  
→ `getLatestBlockhash`  
→ `getSlot (confirmed)`

### ✔ What is the latency to the RPC provider?
- CLI latency (`arx-info`)
- HTTP RPC latency (`curl`)

### ✔ Is my RPC behind in slots?
→ compare primary RPC vs reference RPC

### ✔ Is the overall cluster healthy from my node’s perspective?
→ node active + RPC healthy + slot lag within threshold

---

## 6.2 Requirements

### ● Node Exporter  
https://github.com/prometheus/node_exporter

### ● Prometheus  
Configured to scrape `node_exporter:9100`.

### ● Grafana  
To build dashboards.

### ● `arcium-cli` 0.4.0

```
arcium --version
arcium-cli 0.4.0
```

### ● Solana RPC endpoint (Helius Devnet)

```
https://devnet.helius-rpc.com/?api-key=<YOUR_HELIUS_API_KEY>
```

**Never expose your API key in public repos.**

---

## 6.3 Enabling the Textfile Collector in Node Exporter

Prometheus will scrape metrics written in `.prom` files inside:

```
/var/lib/node_exporter/textfile_collector/
```

### Create the directory

```bash
sudo mkdir -p /var/lib/node_exporter/textfile_collector
sudo chown -R node_exporter:node_exporter /var/lib/node_exporter
```

### Edit Node Exporter service

```
sudo nano /etc/systemd/system/node_exporter.service
```

Add:

```
ExecStart=/usr/local/bin/node_exporter   --collector.textfile.directory=/var/lib/node_exporter/textfile_collector
```

Reload & restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart node_exporter
```

---

## 6.4 Custom ARX Node & Cluster Exporter (Prometheus Textfile Collector)

This exporter writes `arcium_metrics.prom` every 30 seconds.

### Metrics exposed

| Metric | Description |
|--------|-------------|
| `arcium_node_active` | ARX node active on‑chain |
| `arcium_node_rpc_latency_ms` | Latency of `arx-info` CLI |
| `arcium_rpc_up` | Whether the Solana RPC responds correctly |
| `arcium_rpc_http_latency_ms` | HTTP JSON-RPC latency |
| `arcium_rpc_slot` | Slot returned by primary RPC |
| `arcium_rpc_slot_ref` | Slot returned by reference RPC |
| `arcium_rpc_slot_lag` | ref_slot − helius_slot |
| `arcium_cluster_health` | 1 if node+RPC healthy |
| `arcium_cluster_info` | Static info metric |

---

## 6.4.1 Environment Variables

```bash
echo 'export HELIUS_API_KEY="YOUR_API_KEY_HERE"' >> ~/.bashrc
source ~/.bashrc
```

---

## 6.4.2 Full Exporter Script

Create:

```bash
sudo nano /usr/local/bin/arcium_exporter.sh
```

Paste:

```bash
#!/usr/bin/env bash

# Arcium node & cluster exporter (Prometheus textfile collector)

NODE_OFFSET="your_node_offset"
CLUSTER_OFFSET="your_cluster_offset"

RPC_URL="https://devnet.helius-rpc.com/?api-key=${HELIUS_API_KEY}"
REF_RPC_URL="https://api.devnet.solana.com"

OUT_FILE="/var/lib/node_exporter/textfile_collector/arcium_metrics.prom"
mkdir -p "$(dirname "$OUT_FILE")"

########################################
# 1) Node Active
########################################

ACTIVE_RAW=$(arcium arx-active "$NODE_OFFSET" --rpc-url "$RPC_URL" 2>/dev/null || echo "")
if echo "$ACTIVE_RAW" | grep -qi "true"; then ACTIVE=1; else ACTIVE=0; fi

########################################
# 2) arx-info CLI Latency
########################################

START=$(date +%s%3N)
arcium arx-info "$NODE_OFFSET" --rpc-url "$RPC_URL" >/tmp/arx-info.out 2>/dev/null
END=$(date +%s%3N)
LATENCY_MS=$((END-START))
[ "$LATENCY_MS" -lt 0 ] && LATENCY_MS=0

########################################
# 3) RPC Health
########################################

START_RPC=$(date +%s%3N)
RPC_RESP=$(curl -s -m 5 -H 'Content-Type: application/json'     -d '{"jsonrpc":"2.0","id":1,"method":"getLatestBlockhash"}' "$RPC_URL")
RPC_STATUS=$?
END_RPC=$(date +%s%3N)
RPC_LATENCY_MS=$((END_RPC-START_RPC))

if [ "$RPC_STATUS" -ne 0 ] || ! echo "$RPC_RESP" | grep -q '"blockhash"'; then
  RPC_UP=0
else
  RPC_UP=1
fi

########################################
# 4) Primary RPC Slot
########################################

SLOT=$(curl -s -m 5 -H 'Content-Type: application/json'    -d '{"jsonrpc":"2.0","id":1,"method":"getSlot","params":[{"commitment":"confirmed"}]}'    "$RPC_URL" | jq -r '.result // empty')

[ -z "$SLOT" ] && SLOT=-1

########################################
# 5) Reference RPC Slot
########################################

REF_SLOT=$(curl -s -m 5 -H 'Content-Type: application/json'    -d '{"jsonrpc":"2.0","id":1,"method":"getSlot","params":[{"commitment":"confirmed"}]}'    "$REF_RPC_URL" | jq -r '.result // empty')

[ -z "$REF_SLOT" ] && REF_SLOT=-1

########################################
# 6) Slot Lag
########################################

if [ "$SLOT" -ge 0 ] && [ "$REF_SLOT" -ge 0 ]; then
  SLOT_LAG=$((REF_SLOT - SLOT))
else
  SLOT_LAG=0
fi

########################################
# 7) Synthetic Cluster Health
########################################

if [ "$ACTIVE" -eq 1 ] && [ "$RPC_UP" -eq 1 ]; then
  CLUSTER_HEALTH=1
else
  CLUSTER_HEALTH=0
fi

########################################
# Write metrics
########################################

cat > "$OUT_FILE" <<EOF
# HELP arcium_node_active Whether the ARX node is active in Arcium (1=active,0=inactive)
# TYPE arcium_node_active gauge
arcium_node_active{node_offset="$NODE_OFFSET"} $ACTIVE

# HELP arcium_node_rpc_latency_ms Latency of arx-info CLI call
# TYPE arcium_node_rpc_latency_ms gauge
arcium_node_rpc_latency_ms{node_offset="$NODE_OFFSET"} $LATENCY_MS

# HELP arcium_rpc_up Whether the Solana RPC endpoint is reachable
# TYPE arcium_rpc_up gauge
arcium_rpc_up{rpc="helius_devnet"} $RPC_UP

# HELP arcium_rpc_http_latency_ms RPC HTTP latency (ms)
# TYPE arcium_rpc_http_latency_ms gauge
arcium_rpc_http_latency_ms{rpc="helius_devnet"} $RPC_LATENCY_MS

# HELP arcium_rpc_slot Latest slot from main RPC
# TYPE arcium_rpc_slot gauge
arcium_rpc_slot{rpc="helius_devnet"} $SLOT

# HELP arcium_rpc_slot_ref Reference RPC slot
# TYPE arcium_rpc_slot_ref gauge
arcium_rpc_slot_ref{rpc="solana_devnet_ref"} $REF_SLOT

# HELP arcium_rpc_slot_lag Slot lag (ref - main)
# TYPE arcium_rpc_slot_lag gauge
arcium_rpc_slot_lag{cluster_offset="$CLUSTER_OFFSET"} $SLOT_LAG

# HELP arcium_cluster_health Synthetic cluster health flag
# TYPE arcium_cluster_health gauge
arcium_cluster_health{cluster_offset="$CLUSTER_OFFSET"} $CLUSTER_HEALTH

# HELP arcium_cluster_info Static cluster info metric
# TYPE arcium_cluster_info gauge
arcium_cluster_info{cluster_offset="$CLUSTER_OFFSET",rpc="helius_devnet"} 1
EOF
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/arcium_exporter.sh
```

---

## 6.4.3 systemd Service

```bash
sudo nano /etc/systemd/system/arcium-exporter.service
```

```
[Unit]
Description=Arcium custom Prometheus exporter
After=network-online.target

[Service]
Type=oneshot
User=(user)
Group=(user)
Environment=HELIUS_API_KEY=%E_HELIUS_API_KEY%
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/bin:/home/(user)/.cargo/bin
ExecStart=/usr/local/bin/arcium_exporter.sh
```

---

## 6.4.4 systemd Timer (every 30 seconds)

```bash
sudo nano /etc/systemd/system/arcium-exporter.timer
```

```
[Unit]
Description=Run the Arcium exporter every 30 seconds

[Timer]
OnBootSec=30
OnUnitActiveSec=30
Unit=arcium-exporter.service

[Install]
WantedBy=timers.target
```

Enable:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now arcium-exporter.timer
```

---

## 6.4.5 Verify Output

```bash
cat /var/lib/node_exporter/textfile_collector/arcium_metrics.prom
```

Metrics are exposed at:

```
http://<server-ip>:9100/metrics
```

---

## 6.5 Prometheus Scrape Configuration

```yaml
scrape_configs:
  - job_name: 'arx-node'
    static_configs:
      - targets:
          - 'SERVER_IP:9100'
```

---

## 6.6 Grafana Dashboard Recommendations

### Node Metrics Panels

- `arcium_node_active`
- `arcium_node_rpc_latency_ms`
- `arcium_rpc_up`
- `arcium_rpc_http_latency_ms`

### Cluster Metrics Panels

- `arcium_cluster_health`
- `arcium_rpc_slot`
- `arcium_rpc_slot_ref`
- `arcium_rpc_slot_lag```

---

## 6.7 Troubleshooting

### `arcium_node_active = 0`
Check:

```bash
arcium arx-active <NODE_OFFSET> --rpc-url "$RPC_URL"
```

### `arcium_rpc_up = 0`
Check:

```bash
curl -s https://devnet.helius-rpc.com/?api-key=...
```

### Large slot lag
RPC provider may be behind reference RPC.

### No metrics in `/metrics`
Verify:

```bash
systemctl status arcium-exporter.timer
systemctl status node_exporter
```

---

## 6.8 Recommended Dashboard Behavior

For production dashboards:

- Show concise metrics by default  
- Provide a toggle for **Advanced Cluster Metrics**  
- Link to GitHub documentation for deeper context  

This keeps the UI clean while providing full transparency for third parties (auditors, project admins, Arcium team).

---
