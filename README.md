# 🛰️ Cumulo — Arcium Node Infrastructure  
**Full Testnet Deployment on Solana Devnet (ARX Node + Custom Cluster)**

This repository contains Cumulo’s complete technical documentation for operating an **Arcium ARX node** and managing a **custom Arcium cluster** on Solana Devnet.  
It includes installation steps, key management, on-chain initialization, cluster membership, logs, operations, troubleshooting, and our roadmap for expansion.

The goal is to maintain a **fully reproducible**, **transparent**, and **production-grade** reference for Cumulo’s participation in the Arcium ecosystem.

## 📁 Repository Structure

/
├─ README.md → Main index of the project  
└─ docs/  

### Node & Cluster Setup

- **01_node_setup.md**  
  ARX node installation guide  
  https://github.com/Cumulo-pro/cumulo-arcium-infra/blob/main/docs/01_node_setup.md

- **01_node_setup_v0_5_2.md**  
  Updated ARX node installation guide for Arcium v0.5.2  
  https://github.com/Cumulo-pro/cumulo-arcium-infra/blob/main/docs/01_node_setup_v0_5_2.md

- **01_2_upgrade_v0_5_2.md**  
  Upgrade procedure to Arcium v0.5.2  
  https://github.com/Cumulo-pro/cumulo-arcium-infra/blob/main/docs/01_2_upgrade_v0_5_2.md

- **02_cluster_setup.md**  
  Cluster creation & membership flow  
  https://github.com/Cumulo-pro/cumulo-arcium-infra/blob/main/docs/02_cluster_setup.md


├─ 03_operations.md → Node operations: logs, restarts, updates  
├─ 04_security.md → Key management & security practices  
├─ 05_test_computations.md → How to run test computations (MXE)  

- **06_monitoring.md**  
  Logging, metrics & monitoring tools  
  https://github.com/Cumulo-pro/cumulo-arcium-infra/blob/main/docs/06_monitoring.md
  
├─ 07_troubleshooting.md → Common errors & solutions  
├─ 08_architecture.md → Internal architecture of ARX nodes  
└─ 09_roadmap.md → Cumulo's future plans for Arcium  
  
## 🚀 Current Status of Cumulo’s ARX Node

| Field | Value |
|-------|-------|
| **Node authority** | `2ytMsamiaVEckSqsNGpg8gNb43vavUdEy1Bhd89LMzHp` |
| **Callback authority** | `EGkHEuiXGLT81ufYZvsWqG12dxkbFtu7Qush2hJ4GRaG` |
| **Node Offset** | `90441123` |
| **Public IP** | `134.119.218.195` |
| **Cluster Offset** | `80123456` |
| **Cluster Pubkey** | `BNY6g5raPHr9Z2g8MimYJ3KtpLAHpBy5bQDHNyKJxqVP` |
| **Directory** | `~/arcium-node-setup` |
| **Status** | **Active** |
| **Membership** | Joined to cluster `80123456` |

## 🔑 Key Files

- node-keypair.json
- callback-kp.json
- identity.pem
- node-config.toml
- arx-node-logs/*.log

## 🧪 Essential Commands

### Node info  
```bash
arcium arx-info 90441123 --rpc-url https://api.devnet.solana.com
arcium arx-active 90441123 --rpc-url https://api.devnet.solana.com
```    

### Logs
```bash
tail -n 50 arcium-node-setup/arx-node-logs/*.log
```

## 🛠️ Technologien Involved  

- **Solana Devnet**  
  RPC: https://api.devnet.solana.com  
- **Docker**  
  Used for ARX node execution  
- **Yarn / Node.js / Rust / Anchor**  
  Required for the Arcium tooling and Solana integration  
- **Arcium CLI**  
  Version: `arcium-cli 0.4.0`  

---

## 📌 Project Objective

Cumulo participates as an advanced infrastructure operator across multiple ecosystems.  
The integration with Arcium follows three phases:  

- **ARX node active and operational** ✔ completed  
- **Cumulo-owned cluster** ✔ completed  
- **Execution of computations & client integration**  
- **Automation and monitoring**  
- **Contributions to the Arcium ecosystem**  

---

## 🧩 Why Arcium?  

Arcium enables **confidential compute** using multi-party computation (MPC), coordinated entirely on Solana.  
Its design provides:  

- **Confidentiality** — computations happen across multiple nodes without exposing data.  
- **Deterministic coordination** — Solana accounts orchestrate every step of the computation lifecycle.  
- **Scalability** — clusters can include multiple ARX nodes working in parallel.  
- **High-performance execution** — fast state updates and low-latency pub/sub through Solana’s runtime.  

For infrastructure operators, Arcium is an opportunity to run **distributed confidential compute nodes** early in the network lifecycle and help test cluster reliability, peer discovery, and execution flow before mainnet.  

## 📞 Contact
For internal coordination: Mon (Cumulo CTO).

