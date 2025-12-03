# 02 — Cluster Setup  
**How to create, manage and join clusters in Arcium**

This document explains the **generic workflow** for working with Arcium clusters on Solana (Devnet or Mainnet, depending on the network).  
It covers:

- What a cluster is  
- How to create a new cluster  
- How to propose nodes to a cluster  
- How nodes join a cluster  
- How to verify that the cluster and nodes are correctly registered on-chain  
- Expected logs during cluster synchronization  

---

# 🧩 1. What is an Arcium Cluster?

A **Cluster** is a set of ARX nodes registered on Solana that can:

- Participate in encrypted computations (MXE)
- Coordinate execution according to the cluster’s configuration
- Share workload limits and security constraints
- Operate in parallel as a trusted compute group

Each cluster has:

- A **Cluster Offset** (unique numeric identifier)
- A **Pubkey** (Solana on-chain account)
- A **Maximum number of nodes** (`max-nodes`)
- A list of **members** stored on-chain

A node **must be explicitly invited** before it can join a cluster.

---

# 🏗️ 2. Creating a New Cluster

To create a cluster, you need:

- Your **operator keypair** (the same used for node registration)
- A **Cluster Offset** (large unique integer chosen by you)
- Desired **max-nodes**

## Command
```bash
arcium init-cluster
--keypair-path <PATH_TO_OPERATOR_KEYPAIR>
--offset <CLUSTER_OFFSET>
--max-nodes <MAX_NODES>
--rpc-url <SOLANA_RPC_URL>
```

### Arguments
| Flag | Description |
|------|-------------|
| `--keypair-path` | Solana keypair authorized to create the cluster |
| `--offset` | Unique numeric identifier for the cluster |
| `--max-nodes` | Maximum number of ARX nodes allowed |
| `--rpc-url` | Solana RPC endpoint (Devnet or Mainnet) |

### Expected output
Cluster initialized successfully:
<CLUSTER_PUBKEY>


At this point:

✔ The cluster exists  
✔ It is empty  
✔ Ready to accept new members  

---

# 👥 3. Membership Process (2-Step Design)

Arcium uses a **2-step membership workflow**:

1. **Cluster admin invites a node**  
   (`propose--join-cluster`)

2. **Node accepts the invitation**  
   (`join-cluster`)

This prevents unknown nodes from joining a cluster without approval.

---

# 4. Step 1 — Propose a Node to Join a Cluster

This command must be executed by the **cluster admin**:
```bash
arcium propose-join-cluster
--keypair-path <ADMIN_KEYPAIR>
--node-offset <NODE_OFFSET>
--cluster-offset <CLUSTER_OFFSET>
--rpc-url <SOLANA_RPC_URL>
```


### Expected output
Success: <TX_SIGNATURE>

The node is now considered a **member** of the cluster.

---

# 📌 6. Verifying Cluster Membership

## A) Check node metadata
```bash
arcium arx-info <NODE_OFFSET>
--rpc-url <SOLANA_RPC_URL>
```

Example output:
Node authority: <PUBKEY>
Cluster memberships:
Pubkey: <CLUSTER_PUBKEY>, Offset: <CLUSTER_OFFSET>


## B) Check activation status
```bash
arcium arx-active <NODE_OFFSET>
--rpc-url <SOLANA_RPC_URL>
```

Should return:
true


---

# 🧾 7. Runtime Logs — What You Should See

Once the node is accepted into a cluster, ARX node logs typically show:

Processing cluster update
Coordinator received cluster unit message: status: Joined
Subscribing to cluster: <CLUSTER_PUBKEY>
Fetching initial state for account: <CLUSTER_PUBKEY>
Re-subscribing to account: <CLUSTER_PUBKEY>


These lines confirm:

✔ The cluster account is being monitored  
✔ The node is receiving cluster updates  
✔ Cluster membership is fully synchronized  

---

# 🔧 8. Troubleshooting

| Issue | Cause | Fix |
|-------|--------|------|
| Node cannot join | Missing proposal | Ensure `propose-join-cluster` was executed |
| RPC timeouts | Devnet instability | Switch to a private or premium RPC |
| Cluster not shown in `arx-info` | Node has not accepted the proposal | Run `join-cluster true` |
| Logs do not show cluster sync | Node not fully running | Check Docker logs & config paths |

---

# 📦 9. Summary

init-cluster → create a new cluster
propose-join-cluster → invite a node
join-cluster true → node accepts
arx-info → confirm membership
arx-active → confirm activation
logs → confirm runtime synchronization



This file describes the full generic lifecycle for working with Arcium clusters.


