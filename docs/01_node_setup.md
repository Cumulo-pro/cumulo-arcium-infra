🛰️ Cumulo – ARX Node Setup Guide (Solana Devnet)
=================================================

This guide covers the full installation of an ARX node on Solana Devnet,
following the same steps we executed on your server.

------------------------------------------------------------
0. PRE-REQUISITES
------------------------------------------------------------

Server:
- Ubuntu 22.04 LTS
- User with sudo (example: cumulo)
- Docker installed
- Solana CLI installed
- Rust installed

Linux dependencies:
sudo apt update && sudo apt install -y \
  curl git openssl build-essential pkg-config libssl-dev jq

------------------------------------------------------------
1. INSTALL RUST
------------------------------------------------------------

curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

Check:
rustc --version
cargo --version

------------------------------------------------------------
2. INSTALL SOLANA CLI
------------------------------------------------------------

curl --proto '=https' --tlsv1.2 -sSfL https://solana-install.solana.workers.dev | bash
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"

Check:
solana --version

------------------------------------------------------------
3. INSTALL DOCKER ENGINE
------------------------------------------------------------

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

------------------------------------------------------------
4. CREATE WORKSPACE
------------------------------------------------------------

mkdir -p ~/arcium-node-setup
cd ~/arcium-node-setup

------------------------------------------------------------
5. INSTALL ARCIUM TOOLING
------------------------------------------------------------

Install minimal Node.js/Yarn only to satisfy installer:
sudo apt update
sudo apt install -y nodejs npm
sudo npm install -g yarn

Install tooling (WITHOUT sudo):
cd ~
curl --proto '=https' --tlsv1.2 -sSfL https://install.arcium.com/ | bash

Verify:
export PATH="$HOME/.cargo/bin:$PATH"
arcup --version
arcium --version

------------------------------------------------------------
6. GENERATE KEYPAIRS
------------------------------------------------------------

cd ~/arcium-node-setup

6.1 Node Authority Keypair
solana-keygen new --outfile node-keypair.json --no-bip39-passphrase

6.2 Callback Authority Keypair
solana-keygen new --outfile callback-kp.json --no-bip39-passphrase

6.3 Identity Keypair
openssl genpkey -algorithm Ed25519 -out identity.pem

------------------------------------------------------------
7. FUND ACCOUNTS (DEVNET SOL)
------------------------------------------------------------

Ensure CLI is on Devnet:
solana config set --url https://api.devnet.solana.com

Get pubkeys:
solana address --keypair node-keypair.json
solana address --keypair callback-kp.json

Airdrop:
solana airdrop 2 <NODE_PUBKEY>
solana airdrop 2 <CALLBACK_PUBKEY>

Check balance:
solana balance <pubkey>

------------------------------------------------------------
8. INIT ARX ACCOUNTS (REGISTER NODE ON-CHAIN)
------------------------------------------------------------

cd ~/arcium-node-setup

NODE_OFFSET=90441123
IP_ADDRESS=134.119.218.195

arcium init-arx-accs \
  --keypair-path node-keypair.json \
  --callback-keypair-path callback-kp.json \
  --peer-keypair-path identity.pem \
  --node-offset 90441123 \
  --ip-address 134.119.218.195 \
  --rpc-url https://api.devnet.solana.com

------------------------------------------------------------
9. CREATE node-config.toml
------------------------------------------------------------

[node]
offset = 90441123
hardware_claim = 0
starting_epoch = 0
ending_epoch = 9223372036854775807

[network]
address = "0.0.0.0"

[solana]
endpoint_rpc = "https://api.devnet.solana.com"
endpoint_wss = "wss://api.devnet.solana.com"
cluster = "Devnet"
commitment.commitment = "confirmed"

------------------------------------------------------------
10. PREPARE LOG DIRECTORY
------------------------------------------------------------

mkdir -p arx-node-logs && touch arx-node-logs/arx.log

------------------------------------------------------------
11. RUN ARX NODE WITH DOCKER
------------------------------------------------------------

docker run -d \
  --name arx-node \
  -e NODE_IDENTITY_FILE=/usr/arx-node/node-keys/node_identity.pem \
  -e NODE_KEYPAIR_FILE=/usr/arx-node/node-keys/node_keypair.json \
  -e OPERATOR_KEYPAIR_FILE=/usr/arx-node/node-keys/operator_keypair.json \
  -e CALLBACK_AUTHORITY_KEYPAIR_FILE=/usr/arx-node/node-keys/callback_authority_keypair.json \
  -e NODE_CONFIG_PATH=/usr/arx-node/arx/node_config.toml \
  -v "$(pwd)/node-config.toml:/usr/arx-node/arx/node_config.toml" \
  -v "$(pwd)/node-keypair.json:/usr/arx-node/node-keys/node_keypair.json:ro" \
  -v "$(pwd)/node-keypair.json:/usr/arx-node/node-keys/operator_keypair.json:ro" \
  -v "$(pwd)/callback-kp.json:/usr/arx-node/node-keys/callback_authority_keypair.json:ro" \
  -v "$(pwd)/identity.pem:/usr/arx-node/node-keys/node_identity.pem:ro" \
  -v "$(pwd)/arx-node-logs:/usr/arx-node/logs" \
  -p 8080:8080 \
  arcium/arx-node

Check:
docker ps

------------------------------------------------------------
12. VERIFY NODE IS OPERATIONAL
------------------------------------------------------------

arcium arx-info 90441123 --rpc-url https://api.devnet.solana.com
arcium arx-active 90441123 --rpc-url https://api.devnet.solana.com

View logs:
docker logs -f arx-node

Host logs:
tail -f ~/arcium-node-setup/arx-node-logs/*.log

------------------------------------------------------------
13. NOTES ABOUT DEVNET PUB/SUB INSTABILITY
------------------------------------------------------------

You will see frequent messages:
“Pub/sub connection closed... reconnecting”

This is normal on Solana Devnet due to:
- network resets
- unstable WebSockets
- rate limits
- dropped connections

Your node automatically reconnects — no error.

To improve reliability:
- use a private Solana RPC (Helius, Triton, Custom RPC)

------------------------------------------------------------
END OF FILE
------------------------------------------------------------
