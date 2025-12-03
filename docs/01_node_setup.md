<h1> 🛰️ Cumulo – ARX Node Setup Guide (Solana Devnet) </h1>
=================================================

This guide covers the full installation of an ARX node on Solana Devnet,
following the same steps we executed on your server.

## 0. PRE-REQUISITES

Server:
- Ubuntu 22.04 LTS
- User with sudo (example: cumulo)
- Docker installed
- Solana CLI installed
- Rust installed
- RAM: 32GB or more
- CPU: 12 Core or more, 2.8GHz base speed or more
- BW: Min 1 Gbit/s


Linux dependencies:  
```bash
sudo apt update && sudo apt install -y \
  curl git openssl build-essential pkg-config libssl-dev jq
```

------------------------------------------------------------
1. INSTALL RUST
------------------------------------------------------------
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```
Check:  
```bash
rustc --version
cargo --version
```
------------------------------------------------------------
2. INSTALL SOLANA CLI
------------------------------------------------------------
```bash
curl --proto '=https' --tlsv1.2 -sSfL https://solana-install.solana.workers.dev | bash
export PATH="$HOME/.local/share/solana/install/active_release/bin:$PATH"
```
Check:  
```bash
solana --version
```
------------------------------------------------------------
3. INSTALL DOCKER ENGINE
------------------------------------------------------------
```bash
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

------------------------------------------------------------
4. CREATE WORKSPACE
------------------------------------------------------------

Create the directory:  
```bash
mkdir -p ~/arcium-node-setup  
cd ~/arcium-node-setup
```
------------------------------------------------------------
5. INSTALL ARCIUM TOOLING
------------------------------------------------------------

Install minimal Node.js/Yarn only to satisfy installer:  
```bash
sudo apt update  
sudo apt install -y nodejs npm  
sudo npm install -g yarn  
``` 
Install tooling (WITHOUT sudo):  
```bash
cd ~  
curl --proto '=https' --tlsv1.2 -sSfL https://install.arcium.com/ | bash  
```
Verify:  
```bash
export PATH="$HOME/.cargo/bin:$PATH"  
arcup --version  
arcium --version  
```
------------------------------------------------------------
6. GENERATE KEYPAIRS
------------------------------------------------------------
```bash
cd ~/arcium-node-setup
```
6.1 Node Authority Keypair  
```bash
solana-keygen new --outfile node-keypair.json --no-bip39-passphrase
```
6.2 Callback Authority Keypair  
```bash
solana-keygen new --outfile callback-kp.json --no-bip39-passphrase
```
6.3 Identity Keypair  
The ARX node requires an Ed25519 identity keypair, which is used for secure peer-to-peer identification.
Generate it with OpenSSL:  
```bash
openssl genpkey -algorithm Ed25519 -out identity.pem
```
Check:  
```bash
ls -l ~/arcium-node-setup
```
This confirms that the three files are OK, and we continue with:  
➡ Step 4 – Fund accounts  
➡ Step 5 – init-arx-accs  
➡ Step 6 – Config file  
➡ Docker run (ARX Node activo)  

------------------------------------------------------------
7. FUND ACCOUNTS (DEVNET SOL)
------------------------------------------------------------

Ensure CLI is on Devnet:  
```bash
solana config set --url https://api.devnet.solana.com
```
Get pubkeys:  
- Node Authority:  
```bash
solana address --keypair node-keypair.json
```
- Callback Authority:  
```bash
solana address --keypair callback-kp.json
```
Airdrop:  
```bash
solana airdrop 2 <NODE_PUBKEY>  
solana airdrop 2 <CALLBACK_PUBKEY>
```
If an airdrop fails due to rate limiting:  
 👉 Try again  
 👉 Or use the web faucet: https://faucet.solana.com/  
 
Check balance:  
```bash
solana balance <pubkey>
```
------------------------------------------------------------
8. INIT ARX ACCOUNTS (REGISTER NODE ON-CHAIN)
------------------------------------------------------------

This command:  
• Registers your node on the Arcium network (Solana Devnet)  
• Creates your on-chain accounts  
• Associates your public IP  
• Saves your identity (identity.pem)  
• Uses your Node Offset as a unique node identifier  

```bash
cd ~/arcium-node-setup
```
We choose NODE_OFFSET  
We must choose a large, unique number.  
```bash
NODE_OFFSET=90441123
IP_ADDRESS=134.119.218.195
```
Official registration of the ARX node on the Arcium network (Solana Devnet)  
```bash
arcium init-arx-accs \
  --keypair-path node-keypair.json \
  --callback-keypair-path callback-kp.json \
  --peer-keypair-path identity.pem \
  --node-offset 90441123 \
  --ip-address 134.119.218.195 \
  --rpc-url https://api.devnet.solana.com
```
------------------------------------------------------------
9. CREATE node-config.toml
------------------------------------------------------------

Estamos en ~/arcium-node-setup, así que crea el archivo:
```bash
cd ~/arcium-node-setup  
sudo vi node-config.toml
```
```bash
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
```
------------------------------------------------------------
10. PREPARE LOG DIRECTORY
------------------------------------------------------------
```bash
mkdir -p arx-node-logs && touch arx-node-logs/arx.log
```
Now use:  
```bash
ls
```
You should see:  
node-keypair.json callback-kp.json identity.pem node-config.toml arx-node-logs/

------------------------------------------------------------
11. RUN ARX NODE WITH DOCKER
------------------------------------------------------------

We launched the official Arcium container:  
```bash
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
```
Check:  
```bash
docker ps
```
• If you don't have the image, Docker will download it automatically.  
• Make sure port 8080 is open in your firewall/provider.  

------------------------------------------------------------
12. VERIFY NODE IS OPERATIONAL
------------------------------------------------------------

Check if the node is active    
```bash
arcium arx-info 90441123 --rpc-url https://api.devnet.solana.com
```
View logs in real time  
```bash
arcium arx-active 90441123 --rpc-url https://api.devnet.solana.com
```
View logs:  
```bash
docker logs -f arx-node
```
Host logs:  
```bash
tail -f ~/arcium-node-setup/arx-node-logs/*.log
```

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
