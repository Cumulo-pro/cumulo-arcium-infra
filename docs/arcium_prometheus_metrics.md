# 📡 Arcium — Prometheus Metrics Reference  
**ARX Node & Cluster Monitoring**

This document describes all Prometheus metrics exposed by the **custom Arcium exporter** used by Cumulo to monitor ARX nodes and cluster health.

The metrics are generated via a **Node Exporter Textfile Collector**, combining:
- `arcium-cli` outputs
- Solana JSON-RPC health checks
- Synthetic cluster-level signals

These metrics are designed for **production-grade monitoring**, Grafana dashboards, and alerting.

---

## 📌 Metric Naming Convention

All metrics are prefixed with:

```
arcium_
```

Labels are used to identify:
- `node_offset` → ARX node identity
- `cluster_offset` → Arcium cluster identity
- `rpc` / `rpc_main` / `rpc_ref` → Solana RPC endpoints involved

---

## 🔹 Node-Level Metrics

### `arcium_node_active`
**Type:** `gauge`  

**Description:**  
Indicates whether the ARX node is **active on-chain** in the Arcium network.

**Labels:**
- `node_offset` — ARX node offset

**Values:**
- `1` → Node is active and registered
- `0` → Node is inactive or not properly registered

---

### `arcium_node_rpc_latency_ms`
**Type:** `gauge`  

**Description:**  
Latency (in milliseconds) of the `arcium arx-info` CLI call against the Solana RPC.

**Labels:**
- `node_offset` — ARX node offset

---

## 🔹 Solana RPC Metrics

### `arcium_rpc_up`
**Type:** `gauge`  

**Description:**  
Indicates whether the Solana RPC endpoint is reachable and responding correctly.

**Labels:**
- `rpc` — RPC identifier (e.g. `helius_devnet`)

---

### `arcium_rpc_http_latency_ms`
**Type:** `gauge`  

**Description:**  
HTTP latency (in milliseconds) of the Solana RPC `getLatestBlockhash` call.

**Labels:**
- `rpc` — RPC identifier

---

## 🔹 Slot Metrics

### `arcium_rpc_slot`
**Type:** `gauge`  

**Description:**  
Latest confirmed slot returned by the **primary Solana RPC**.

**Labels:**
- `rpc` — Primary RPC identifier

---

### `arcium_rpc_slot_ref`
**Type:** `gauge`  

**Description:**  
Latest confirmed slot returned by the **reference Solana RPC**.

**Labels:**
- `rpc` — Reference RPC identifier

---

### `arcium_rpc_slot_lag`
**Type:** `gauge`  

**Description:**  
Slot lag between the reference RPC and the primary RPC.

**Labels:**
- `cluster_offset`
- `rpc_main`
- `rpc_ref`

---

## 🔹 Cluster-Level Metrics

### `arcium_cluster_health`
**Type:** `gauge`  

**Description:**  
Synthetic health indicator of the Arcium cluster from the perspective of this node.

**Labels:**
- `cluster_offset`
- `rpc`

---

### `arcium_cluster_info`
**Type:** `gauge`  

**Description:**  
Static informational metric used to expose cluster metadata via labels.

**Labels:**
- `cluster_offset`
- `rpc`

---

## 🧠 Design Notes

- Textfile collector avoids custom HTTP exporters.
- All metrics are gauges for simplicity.
- Slot metrics enable objective RPC quality analysis.
- Synthetic health is intentionally conservative.

---
