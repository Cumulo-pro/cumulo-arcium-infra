# Arcium — Grafana Dashboards

This directory contains Grafana dashboards used by **Cumulo** to monitor **Arcium ARX nodes and clusters**.

These dashboards are built on top of:
- Prometheus
- Node Exporter (textfile collector)
- Custom Arcium exporter

They focus exclusively on **visualization**.  
Metric definitions, exporter logic, and monitoring architecture are documented elsewhere in the repository.

---

## Available dashboards

### `arcium-arx-node-cluster.json`

Main operational dashboard for Arcium infrastructure.

It provides visibility into:
- ARX node on-chain status
- Solana RPC availability
- RPC latency (CLI vs HTTP)
- Slot progression and slot lag
- Synthetic cluster health

The dashboard is intentionally compact and avoids duplicated panels, prioritizing:
- Fast operational insight
- Incident analysis
- Audit-friendly clarity

---

## Usage

1. Import the JSON file into Grafana
2. Select the appropriate Prometheus datasource
3. Ensure Prometheus is scraping the target node exporter

No additional configuration is required inside Grafana.

---

Maintained by **Cumulo**  
Validator & Infrastructure Operator
