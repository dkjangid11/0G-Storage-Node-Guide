# 🪐 0G-Storage-Node Setup Guide

A simple guide to install and run a 0G Storage Node with snapshot support, optimized `.toml` configuration, and local SSD setup on VPS.

---

## ⚙️ System Requirements

- Ubuntu 22.04 or 24.04
- Minimum: 4 vCPUs, 16GB RAM
- At least 350GB SSD (preferably local NVMe SSD)

---

## 🛠 Pre-Setup

- Add 0G Galileo Testnet chain:  
  👉 https://docs.0g.ai/run-a-node/testnet-information

- Get faucet tokens:  
  👉 https://faucet.0g.ai/

---

## 🔧 Step 1: Install Required Dependencies

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install curl iptables build-essential git wget lz4 jq make cmake gcc nano \
automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev \
tar clang bsdmainutils ncdu unzip libleveldb-dev screen ufw -y
```

---

## 🦀 Step 2: Install Rust

```bash
curl https://sh.rustup.rs -sSf | sh
source $HOME/.cargo/env
rustc --version
```

---

## 🟨 Step 3: Install Go

```bash
wget https://go.dev/dl/go1.24.3.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.24.3.linux-amd64.tar.gz
rm go1.24.3.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
go version
```

---

## 🧱 Step 4: Clone the 0G Node Repository

```bash
git clone https://github.com/0glabs/0g-storage-node.git
cd 0g-storage-node
git checkout v1.0.0
git submodule update --init
cargo build --release
```

---

## ⚙️ Step 5: Download and Configure `config.toml`

```bash
rm -rf $HOME/0g-storage-node/run/config.toml
curl -o $HOME/0g-storage-node/run/config.toml https://raw.githubusercontent.com/Mayankgg01/0G-Storage-Node-Guide/main/config.toml
```

Then edit it:

```bash
nano $HOME/0g-storage-node/run/config.toml
```

### ✏️ In the config:
- Add your **wallet's private key** to `miner_key` (❗Don't use `0x` prefix)
- Change the data directory to:  
  `db_dir = "/zkcache/0g-node-data"`

---

## 💾 Step 6: Setup Local SSD

```bash
lsblk  # Check your SSD disk name (e.g., nvme0n1)

sudo mkfs.ext4 -F /dev/nvme0n1
sudo mkdir -p /zkcache/0g-node-data
sudo mount /dev/nvme0n1 /zkcache

df -h /zkcache  # Confirm mount
```

(Optional) For auto-mount on reboot:
```bash
sudo blkid /dev/nvme0n1
```

Then add the UUID to `/etc/fstab`:
```
UUID=xxxxx  /zkcache  ext4  defaults,nofail  0  2
```

---

## ⚡ Step 7: Download Snapshot to SSD

```bash
wget https://github.com/Mayankgg01/0G-Storage-Node-Guide/releases/download/v1.0/flow_db.tar.gz \
  -O /zkcache/0g-node-data/flow_db.tar.gz

tar -xzvf /zkcache/0g-node-data/flow_db.tar.gz -C /zkcache/0g-node-data/
rm /zkcache/0g-node-data/flow_db.tar.gz
```

---

## 🧩 Step 8: Create Systemd Service

```bash
sudo tee /etc/systemd/system/zgs.service > /dev/null <<EOF
[Unit]
Description=ZGS Node
After=network.target

[Service]
User=$USER
WorkingDirectory=$HOME/0g-storage-node/run
ExecStart=$HOME/0g-storage-node/target/release/zgs_node --config $HOME/0g-storage-node/run/config.toml
Restart=on-failure
RestartSec=10
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

---

## 🚀 Step 9: Start the Node

```bash
sudo systemctl daemon-reload
sudo systemctl enable zgs
sudo systemctl start zgs
```

---

## 📡 Step 10: Monitor Logs and Sync

### Check Status

```bash
sudo systemctl status zgs
```

### View Logs

```bash
tail -f ~/0g-storage-node/run/log/zgs.log.$(TZ=UTC date +%Y-%m-%d)
```

### Live Sync Status Script

```bash
while true; do
  response=$(curl -s -X POST http://localhost:5678 -H "Content-Type: application/json" \
    -d '{"jsonrpc":"2.0","method":"zgs_getStatus","params":[],"id":1}')
  logSyncHeight=$(echo $response | jq '.result.logSyncHeight')
  connectedPeers=$(echo $response | jq '.result.connectedPeers')
  echo -e "logSyncHeight: \033[32m$logSyncHeight\033[0m, connectedPeers: \033[34m$connectedPeers\033[0m"
  sleep 5
done
```

---

## ❌ To Stop and Remove the Node

```bash
sudo systemctl stop zgs
sudo systemctl disable zgs
sudo rm /etc/systemd/system/zgs.service
sudo systemctl daemon-reexec
```

---

## 🙌 Credits

- [0G Labs](https://0g.ai)
- [Testnet Docs](https://docs.0g.ai)
- [Faucet](https://faucet.0g.ai)
- Config & Snapshot: [@Mayankgg01](https://github.com/Mayankgg01)
