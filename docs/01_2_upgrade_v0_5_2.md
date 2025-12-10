# 🛰️ Cumulo -- ARX Node Upgrade Guide

## **Arcium v0.4.x → v0.5.2 (Solana Devnet)**

This document provides the official Cumulo migration guide for operators
upgrading an existing ARX node from **v0.4.x** to the new **v0.5.2**
release.

------------------------------------------------------------------------

# 0. IMPORTANT CHANGES IN v0.5.2

  Change                                                  Required?
  ------------------------------------------------------- ------------------
  New **BLS keypair** requirement                         ✅ Mandatory
  Re-run `init-arx-accs`                                  ✅ Mandatory
  Node must belong to a **cluster**                       ✅ Mandatory
  Old clusters (v0.4.x) **no longer valid**               ✅ Must recreate
  Updated ports: **8001 (HTTP)**, **8002 (BLS gossip)**   ✅ Mandatory
  Updated Docker run                                      ✅ Mandatory
  JSON BLS format bug fixed                               ✅ Yes

If you skip these steps, you will encounter errors like:

    Failed to create coordinator: Node is not part of any cluster
    Failed to read BLS_PRIVATE_KEY_FILE env var: NotPresent
    Failed to convert BLS keypair to 32 byte array

------------------------------------------------------------------------

# 1. UPGRADE ARCIUM TOOLING

Update CLI tools:

``` bash
curl --proto '=https' --tlsv1.2 -sSfL https://install.arcium.com/ | bash
export PATH="$HOME/.cargo/bin:$PATH"
```

Verify:

``` bash
arcup --version
arcium --version
```

Expected:

    arcup 0.5.2
    arcium-cli 0.5.2

------------------------------------------------------------------------

# 2. GENERATE THE NEW BLS KEYPAIR

This is **new in v0.5.x**.

``` bash
cd ~/arcium-node-setup
arcium gen-bls-key bls-keypair.json
```

✔ Use **only** JSON format in v0.5.2\
✘ Do **not** convert to `.bin` as required in v0.5.1

------------------------------------------------------------------------

# 3. RE-EXECUTE `init-arx-accs` (WITH BLS KEY)

You **must** re-register your node with the BLS key for v0.5.2.

``` bash
arcium init-arx-accs \
  --keypair-path node-keypair.json \
  --callback-keypair-path callback-kp.json \
  --peer-keypair-path identity.pem \
  --bls-keypair-path bls-keypair.json \
  --node-offset <NODE_OFFSET> \
  --ip-address <PUBLIC_IP> \
  --rpc-url https://api.devnet.solana.com
```

This updates all on-chain metadata to the new version.

------------------------------------------------------------------------

# 4. RECREATE YOUR CLUSTER (MANDATORY)

Clusters from v0.4.x **do not work** under v0.5.2.

Choose a new offset:

``` bash
CLUSTER_OFFSET=80123456
```

Create the new cluster:

``` bash
arcium init-cluster \
  --keypair-path node-keypair.json \
  --offset $CLUSTER_OFFSET \
  --max-nodes 10 \
  --mempool-size Small \
  --rpc-url https://api.devnet.solana.com
```

------------------------------------------------------------------------

# 5. JOIN YOUR NODE TO THE NEW CLUSTER

### 5.1 Propose join

``` bash
arcium propose-join-cluster \
  --keypair-path node-keypair.json \
  --node-offset <NODE_OFFSET> \
  --cluster-offset $CLUSTER_OFFSET \
  --rpc-url https://api.devnet.solana.com
```

### 5.2 Approve + join

``` bash
arcium join-cluster true \
  --keypair-path node-keypair.json \
  --node-offset <NODE_OFFSET> \
  --cluster-offset $CLUSTER_OFFSET \
  --rpc-url https://api.devnet.solana.com
```

Verify:

``` bash
arcium arx-info <NODE_OFFSET> --rpc-url https://api.devnet.solana.com
```

Expected output:

    Active Cluster: Offset: 80123456

------------------------------------------------------------------------

# 6. UPDATE YOUR node-config.toml

We strongly recommend using a private Devnet RPC (Helius).

``` toml
[solana]
endpoint_rpc = "https://devnet.helius-rpc.com/?api-key=YOUR_KEY"
endpoint_wss = "wss://devnet.helius-rpc.com/?api-key=YOUR_KEY"
cluster = "Devnet"
commitment.commitment = "confirmed"
```

------------------------------------------------------------------------

# 7. REMOVE THE OLD CONTAINER

``` bash
docker stop arx-node 2>/dev/null || true
docker rm arx-node 2>/dev/null || true
```

------------------------------------------------------------------------

# 8. LAUNCH THE NEW v0.5.2 DOCKER CONTAINER

``` bash
docker run -d \
  --name arx-node \
  -e NODE_IDENTITY_FILE=/usr/arx-node/node-keys/node_identity.pem \
  -e NODE_KEYPAIR_FILE=/usr/arx-node/node-keys/node_keypair.json \
  -e CALLBACK_AUTHORITY_KEYPAIR_FILE=/usr/arx-node/node-keys/callback_authority_keypair.json \
  -e BLS_PRIVATE_KEY_FILE=/usr/arx-node/node-keys/bls_keypair.json \
  -v "$(pwd)/node-config.toml:/usr/arx-node/arx/node_config.toml" \
  -v "$(pwd)/node-keypair.json:/usr/arx-node/node-keys/node_keypair.json:ro" \
  -v "$(pwd)/callback-kp.json:/usr/arx-node/node-keys/callback_authority_keypair.json:ro" \
  -v "$(pwd)/identity.pem:/usr/arx-node/node-keys/node_identity.pem:ro" \
  -v "$(pwd)/bls-keypair.json:/usr/arx-node/node-keys/bls_keypair.json:ro" \
  -v "$(pwd)/arx-node-logs:/usr/arx-node/logs" \
  -p 8001:8001 \
  -p 8002:8002 \
  arcium/arx-node
```

------------------------------------------------------------------------

# 9. VERIFY NODE STATUS

### Node Active?

``` bash
arcium arx-active <NODE_OFFSET> --rpc-url https://api.devnet.solana.com
```

### On-chain info:

``` bash
arcium arx-info <NODE_OFFSET> --rpc-url https://api.devnet.solana.com
```

### Logs:

``` bash
docker logs -f arx-node
tail -f arx-node-logs/*.log
```

Expected log entries:

    Coordinator started
    Node activated
    BLS unit listening on 0.0.0.0:8002

------------------------------------------------------------------------

# 10. COMMON ERRORS & FIXES

### ❌ `Node is not part of any cluster`

You skipped steps 4--5.\
Recreate cluster + join it.

------------------------------------------------------------------------

### ❌ `Failed to read BLS_PRIVATE_KEY_FILE`

You did not mount the file in Docker.\
Check:

    -v "$(pwd)/bls-keypair.json"

------------------------------------------------------------------------

### ❌ `Failed to convert BLS keypair...`

This was a v0.5.1 issue --- fixed in v0.5.2.\
Use JSON only.

------------------------------------------------------------------------

### ❌ Pub/Sub disconnections

Normal on Solana Devnet.\
Use Helius or a private RPC.

------------------------------------------------------------------------

# 🎉 UPGRADE COMPLETE

You now run a **fully updated ARX node (v0.5.2)** with:

✔ New BLS key\
✔ New cluster\
✔ Updated on-chain metadata\
✔ Updated Docker deployment\
✔ Stable RPC configuration\
✔ Active + healthy node status

This is the official Cumulo upgrade path tested in production.
