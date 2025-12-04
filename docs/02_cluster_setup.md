# 02_cluster_setup.md — Cluster Creation & Membership Flow (Generic Guide)

## 1. Overview
This document explains how to create an Arcium cluster and how nodes join it.  
The guide is **generic**, not specific to Cumulo, and applies to any operator building a cluster on Solana Devnet or Mainnet in the future.

---

## 2. Creating a New Cluster

A cluster in Arcium is defined by:
- A **cluster admin** (the keypair that initializes the cluster)
- A **cluster offset** (unique numerical ID)
- A **maximum number of nodes**
- A Solana RPC endpoint

### 📘 Command — Initialize a New Cluster
Run this from the cluster administrator's machine:

```bash
arcium init-cluster \
  --keypair-path <ADMIN_KEYPAIR> \
  --offset <CLUSTER_OFFSET> \
  --max-nodes <MAX_NODES> \
  --rpc-url <SOLANA_RPC_URL>
```

### ✔ Expected output
```
Cluster initialized successfully: <CLUSTER_PUBKEY>
```

Save the resulting public key — it identifies the cluster on-chain.

---

## 3. Membership Process (2-Step Design)

Arcium uses a **permissioned join mechanism** for clusters:

1. **Cluster admin proposes a node**  
   (`propose-join-cluster`)
2. **Node accepts the invitation**  
   (`join-cluster`)

This prevents unauthorized nodes from joining clusters without approval.

---

## 4. Step 1 — Cluster Admin Proposes a Node

The cluster admin must explicitly invite a node using:

```bash
arcium propose-join-cluster \
  --keypair-path <ADMIN_KEYPAIR> \
  --node-offset <NODE_OFFSET> \
  --cluster-offset <CLUSTER_OFFSET> \
  --rpc-url <SOLANA_RPC_URL>
```

### ✔ Expected output
```
Success: <TRANSACTION_SIGNATURE>
```

At this stage the node is *pending approval*.

---

## 5. Step 2 — Node Accepts the Invitation

The node operator must now confirm joining the cluster.

### 🟢 Node-Side Command — Accept the Invitation
Run on the **node machine** using its own keypair:

```bash
arcium join-cluster true \
  --keypair-path <NODE_KEYPAIR> \
  --node-offset <NODE_OFFSET> \
  --cluster-offset <CLUSTER_OFFSET> \
  --rpc-url <SOLANA_RPC_URL>
```

### ✔ Expected output
```
Success: <TRANSACTION_SIGNATURE>
```

The `true` flag indicates the node is accepting the proposal.

---

## 6. Verification Steps

### 🔍 Check cluster membership
```bash
arcium arx-info <NODE_OFFSET> \
  --rpc-url <SOLANA_RPC_URL>
```

Expected section:

```
Cluster memberships:
Pubkey: <CLUSTER_PUBKEY>, Offset: <CLUSTER_OFFSET>
```

### 🔍 Check node activation
```bash
arcium arx-active <NODE_OFFSET> \
  --rpc-url <SOLANA_RPC_URL>
```

Expected:
```
true
```

### 🔍 Check node logs (if using Docker)
```bash
docker logs -f arx-node
```

---

## 7. Summary

| Step | Command | Who Executes It |
|------|---------|------------------|
| Create cluster | `init-cluster` | Cluster admin |
| Invite a node | `propose-join-cluster` | Cluster admin |
| Accept invitation | `join-cluster true` | Node operator |
| Verify | `arx-info`, `arx-active` | Both |

---

## 8. Notes

- Each **cluster-offset** must be globally unique.  
- Nodes require a **node-offset**, also unique.  
- Clusters can be discovered using:  
  ```bash
  arcium list-clusters
  ```
- Solana RPC stability affects node performance. Paid RPC providers are recommended.

---

End of document.
