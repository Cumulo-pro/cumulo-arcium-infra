# 6. Monitoring the ARX Node & the Arcium Cluster  
This chapter explains how to build a **complete monitoring system** for your ARX node and the supporting infrastructure.  
Arcium does **not** currently expose a native Prometheus metrics endpoint, so node operators must build monitoring using the following:

- **`arcium-cli` outputs**  
- **Solana RPC health & latency checks**  
- **Node Exporter + Textfile Collector**  
- **Prometheus + Grafana dashboards**

This document describes a **production-grade custom Prometheus exporter**, widely used by professional operators.  
It allows you to track the live status of your ARX node, RPC health, latency, and slot progression.

---

# 6.1 Goals of the Monitoring System

The monitoring stack must answer — in real time — the following operational questions:

### ✔ Is my ARX node active on-chain?  
→ via `arcium arx-active <offset>`

### ✔ Is the Solana RPC (Helius Devnet) healthy?  
→ via JSON-RPC (`getLatestBlockhash`, `getSlot`)

### ✔ What is the latency to my RPC provider?  
→ CLI latency (`arx-info`)  
→ HTTP latency (`curl` + JSON-RPC)

### ✔ Is my RPC behind in slots?  
→ via `getSlot (commitment=confirmed)`

---

# 6.2 Requirements

Before you begin, you must have the following installed:

### ● Node Exporter  
https://github.com/prometheus/node_exporter

### ● Prometheus  
Configured to scrape `node_exporter:9100`.

### ● Grafana  
For dashboards.

### ● `arcium-cli`  
This guide was created using:

```
arcium-cli 0.4.0
```

### ● A Solana RPC endpoint (Helius Devnet)

```
https://devnet.helius-rpc.com/?api-key=<YOUR_HELIUS_API_KEY>
```

**Important:** never publish your API key in a public repo.  
This guide uses environment variables (`HELIUS_API_KEY`).

---

# 6.3 Enabling the Textfile Collector in Node Exporter

Prometheus will read metrics from `.prom` files placed inside a dedicated directory.

Create the directory:

```bash
sudo mkdir -p /var/lib/node_exporter/textfile_collector
sudo chown -R node_exporter:node_exporter /var/lib/node_exporter
```

Edit Node Exporter service:

```bash
sudo nano /etc/systemd/system/node_exporter.service
```

Add:

```ini
ExecStart=/usr/local/bin/node_exporter   --collector.textfile.directory=/var/lib/node_exporter/textfile_collector
```

Reload:

```bash
sudo systemctl daemon-reload
sudo systemctl restart node_exporter
```

---

# 6.4 Custom ARX Node Exporter (Prometheus Textfile Collector)

This exporter exposes the following metrics:

| Metric | Description |
|--------|-------------|
| `arcium_node_active` | Whether the ARX node is active on-chain |
| `arcium_node_rpc_latency_ms` | Latency of `arx-info` CLI call |
| `arcium_rpc_up` | Health of the Solana RPC endpoint |
| `arcium_rpc_http_latency_ms` | RPC latency via HTTP |
| `arcium_rpc_slot` | Latest slot returned by RPC |

This gives you full visibility over your node’s operational state.

---

## 6.4.1 Environment Variables

Add your Helius RPC key:

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

# Arcium node custom exporter (Prometheus textfile collector)

NODE_OFFSET="90441123"
RPC_URL="https://devnet.helius-rpc.com/?api-key=${HELIUS_API_KEY}"

OUT_FILE="/var/lib/node_exporter/textfile_collector/arcium_metrics.prom"
mkdir -p "$(dirname "$OUT_FILE")"

########################################
# Metric 1: ARX node on-chain status
########################################

ACTIVE_RAW=$(arcium arx-active "$NODE_OFFSET" --rpc-url "$RPC_URL" 2>/dev/null || echo "")
if echo "$ACTIVE_RAW" | grep -qi "true"; then ACTIVE=1; else ACTIVE=0; fi

########################################
# Metric 2: Latency of arx-info CLI
########################################

START=$(date +%s%3N)
arcium arx-info "$NODE_OFFSET" --rpc-url "$RPC_URL" >/tmp/arx-info.out 2>/dev/null
END=$(date +%s%3N)
LATENCY_MS=$((END-START))
[ -z "$LATENCY_MS" ] && LATENCY_MS=0

########################################
# Metric 3: RPC health (getLatestBlockhash)
########################################

START_RPC=$(date +%s%3N)
RPC_RESP=$(curl -s -m 5 -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","id":1,"method":"getLatestBlockhash"}' "$RPC_URL")
RPC_STATUS=$?
END_RPC=$(date +%s%3N)
RPC_LATENCY_MS=$((END_RPC-START_RPC))

if [ "$RPC_STATUS" -ne 0 ] || ! echo "$RPC_RESP" | grep -q '"blockhash"'; then
  RPC_UP=0
else
  RPC_UP=1
fi

########################################
# Metric 4: Latest slot (getSlot)
########################################

SLOT=$(curl -s -m 5 -H 'Content-Type: application/json'   -d '{"jsonrpc":"2.0","id":1,"method":"getSlot","params":[{"commitment":"confirmed"}]}'   "$RPC_URL" | jq -r '.result // empty')

[ -z "$SLOT" ] && SLOT=-1

########################################
# Write the .prom file
########################################

cat > "$OUT_FILE" <<EOF
# HELP arcium_node_active Whether the ARX node is active in Arcium (1=active,0=inactive)
# TYPE arcium_node_active gauge
arcium_node_active{node_offset="$NODE_OFFSET"} $ACTIVE

# HELP arcium_node_rpc_latency_ms Latency of arx-info call to Solana RPC (Helius) in milliseconds
# TYPE arcium_node_rpc_latency_ms gauge
arcium_node_rpc_latency_ms{node_offset="$NODE_OFFSET"} $LATENCY_MS

# HELP arcium_rpc_up Whether the Solana RPC endpoint is reachable (1=up,0=down)
# TYPE arcium_rpc_up gauge
arcium_rpc_up{rpc="helius_devnet"} $RPC_UP

# HELP arcium_rpc_http_latency_ms Latency of Solana RPC getLatestBlockhash HTTP call (Helius HTTP) in milliseconds
# TYPE arcium_rpc_http_latency_ms gauge
arcium_rpc_http_latency_ms{rpc="helius_devnet"} $RPC_LATENCY_MS

# HELP arcium_rpc_slot Latest slot returned by Solana RPC getSlot (commitment=confirmed)
# TYPE arcium_rpc_slot gauge
arcium_rpc_slot{rpc="helius_devnet"} $SLOT
EOF
```

Make executable:

```bash
sudo chmod +x /usr/local/bin/arcium_exporter.sh
```

---

# 6.4.3 systemd Service

Create the service:

```bash
sudo nano /etc/systemd/system/arcium-exporter.service
```

Content:

```ini
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

# 6.4.4 systemd Timer (runs every 30 seconds)

```bash
sudo nano /etc/systemd/system/arcium-exporter.timer
```

Content:

```ini
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

# 6.4.5 Verify Output

```bash
cat /var/lib/node_exporter/textfile_collector/arcium_metrics.prom
```

---

# 6.5 Grafana Dashboards

Recommended panels:

- Node active status
- RPC health
- RPC latency
- Slot progression
- arx-info latency

---

# 6.6 Troubleshooting
Refer to earlier steps for debugging PATH, permissions, and RPC errors.

---
