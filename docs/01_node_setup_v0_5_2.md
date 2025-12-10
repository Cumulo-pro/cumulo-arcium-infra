# 🛰️ Cumulo -- ARX Node Setup Guide (Solana Devnet, v0.5.2)

This guide provides a **complete, up‑to‑date, production‑grade
installation** of an ARX node for Arcium **v0.5.2**, including all new
requirements: - BLS key generation\
- Updated `init-arx-accs` flow\
- Mandatory cluster membership\
- Updated Docker run configuration (ports 8001/8002)\
- New troubleshooting notes

------------------------------------------------------------------------

# 0. PRE‑REQUISITES

### Server Requirements

  Component   Minimum
  ----------- -----------------------
  OS          Ubuntu 22.04 LTS
  CPU         8--12 cores @ 2.8GHz+
  RAM         32 GB
  Disk        100 GB SSD/NVMe
  Network     1 Gbit/s recommended
  User        With sudo permissions

### Linux Dependencies

``` bash
sudo apt update && sudo apt install -y \
  curl git openssl build-essential pkg-config libssl-dev jq
```

------------------------------------------------------------------------

# 1. INSTALL RUST

``` bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

Verify:

``` bash
rustc --version
cargo --version
```

------------------------------------------------------------------------

# 2. INSTALL SOLANA CLI

``` bash
curl --proto '=https' --tlsv1.2 -sSfL https://solana-install.solana.workers.dev | bash
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"
```

Verify:

``` bash
solana --version
```

------------------------------------------------------------------------

# 3. INSTALL DOCKER ENGINE

``` bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```

------------------------------------------------------------------------

# 4. CREATE WORKSPACE

``` bash
mkdir -p ~/arcium-node-setup
cd ~/arcium-node-setup
```

------------------------------------------------------------------------

# 5. INSTALL ARCIUM TOOLING (v0.5.2)

Arcium tooling requires minimal Node/Yarn packages:

``` bash
sudo apt update
sudo apt install -y nodejs npm
sudo npm install -g yarn
```

Now install Arcium tooling **without sudo**:

``` bash
cd ~
curl --proto '=https' --tlsv1.2 -sSfL https://install.arcium.com/ | bash
```

Add CLI tools to PATH:

``` bash
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

# 6. GENERATE KEYPAIRS

``` bash
cd ~/arcium-node-setup
```

### 6.1 Node Authority Keypair

``` bash
solana-keygen new --outfile node-keypair.json --no-bip39-passphrase
```

### 6.2 Callback Authority Keypair

``` bash
solana-keygen new --outfile callback-kp.json --no-bip39-passphrase
```

### 6.3 Identity Keypair (Ed25519)

``` bash
openssl genpkey -algorithm Ed25519 -out identity.pem
```

### 6.4 NEW (v0.5.x): BLS Keypair

Required for zero‑knowledge computation signatures.

``` bash
arcium gen-bls-key bls-keypair.json
```

Verify:

``` bash
ls -l
```

------------------------------------------------------------------------

# 7. FUND ACCOUNTS (Solana Devnet)

Configure Solana CLI:

``` bash
solana config set --url https://api.devnet.solana.com
```

Get pubkeys:

``` bash
solana address --keypair node-keypair.json
solana address --keypair callback-kp.json
```

Fund each:

``` bash
solana airdrop 2 <NODE_PUBKEY>
solana airdrop 2 <CALLBACK_PUBKEY>
```

If rate‑limited, retry or use: https://faucet.solana.com/

------------------------------------------------------------------------

# 8. REGISTER NODE ON-CHAIN (init-arx-accs)

Choose unique numbers:

``` bash
NODE_OFFSET=90441123
IP_ADDRESS=<YOUR_PUBLIC_IP>
```

Run:

``` bash
arcium init-arx-accs \
  --keypair-path node-keypair.json \
  --callback-keypair-path callback-kp.json \
  --peer-keypair-path identity.pem \
  --bls-keypair-path bls-keypair.json \
  --node-offset $NODE_OFFSET \
  --ip-address $IP_ADDRESS \
  --rpc-url https://api.devnet.solana.com
```

This creates all on-chain node accounts.

------------------------------------------------------------------------

# 9. CREATE CLUSTER (MANDATORY IN v0.5.x)

Nodes must belong to a cluster.

Choose cluster offset:

``` bash
CLUSTER_OFFSET=80123456
```

Create cluster:

``` bash
arcium init-cluster \
  --keypair-path node-keypair.json \
  --offset $CLUSTER_OFFSET \
  --max-nodes 10 \
  --mempool-size Small \
  --rpc-url https://api.devnet.solana.com
```

------------------------------------------------------------------------

# 10. JOIN NODE TO THE CLUSTER

### 10.1 Propose your node

``` bash
arcium propose-join-cluster \
  --keypair-path node-keypair.json \
  --node-offset $NODE_OFFSET \
  --cluster-offset $CLUSTER_OFFSET \
  --rpc-url https://api.devnet.solana.com
```

### 10.2 Approve + join

``` bash
arcium join-cluster true \
  --keypair-path node-keypair.json \
  --node-offset $NODE_OFFSET \
  --cluster-offset $CLUSTER_OFFSET \
  --rpc-url https://api.devnet.solana.com
```

Verify:

``` bash
arcium arx-info $NODE_OFFSET --rpc-url https://api.devnet.solana.com
```

Expected:

    Active Cluster: Offset: 80123456

------------------------------------------------------------------------

# 11. CREATE node-config.toml

Recommended to use a **private RPC provider** like Helius.

``` bash
sudo tee node-config.toml > /dev/null <<EOF
[node]
offset = 90441123
hardware_claim = 0
starting_epoch = 0
ending_epoch = 9223372036854775807

[network]
address = "0.0.0.0"

[solana]
endpoint_rpc = "https://devnet.helius-rpc.com/?api-key=YOUR_KEY"
endpoint_wss = "wss://devnet.helius-rpc.com/?api-key=YOUR_KEY"
cluster = "Devnet"
commitment.commitment = "confirmed"
EOF
```

------------------------------------------------------------------------

# 12. PREPARE LOG DIRECTORY

``` bash
mkdir -p arx-node-logs && touch arx-node-logs/arx.log
```

------------------------------------------------------------------------

# 13. RUN ARX NODE (Docker v0.5.2)

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

Check:

``` bash
docker ps
```

------------------------------------------------------------------------

# 14. VERIFY OPERATION

### Node active?

``` bash
arcium arx-active $NODE_OFFSET --rpc-url https://api.devnet.solana.com
```

### Get on-chain info:

``` bash
arcium arx-info $NODE_OFFSET --rpc-url https://api.devnet.solana.com
```

### Logs:

``` bash
docker logs -f arx-node
tail -f arx-node-logs/*.log
```

Expected log snippets:

    Coordinator started
    Node activated
    BLS unit listening on 0.0.0.0:8002

------------------------------------------------------------------------

# 15. TROUBLESHOOTING (v0.5.2)

### ❌ `Failed to create coordinator: Node is not part of any cluster`

Solution: You MUST join a cluster. Repeat steps 9--10.

------------------------------------------------------------------------

### ❌ `Failed to read BLS keypair`

Solution: Use JSON file directly in v0.5.2:

    --bls-keypair-path bls-keypair.json

------------------------------------------------------------------------

### ❌ Pub/Sub disconnections on Devnet

Normal behaviour due to Solana Devnet instability. Node auto-reconnects.

Use private RPC for stability (Helius, Triton).

------------------------------------------------------------------------

# 16. YOUR NODE IS READY 🎉

You now run a fully functional **Arcium ARX node v0.5.2** with: - BLS
signing\
- Cluster membership\
- Stable Solana RPC\
- Correct Docker deployment\
- On‑chain registration

This is the **official Cumulo‑grade setup**, matching our production
infrastructure.
