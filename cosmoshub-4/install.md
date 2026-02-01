# Cosmos Hub (cosmoshub-4) — Full Node Installation & Validator Setup (Gaia v25.2.0)

This document describes **all required steps** to install, configure, and operate a **Cosmos Hub** node and, if applicable, **create a validator** using **Gaia (gaiad) v25.2.0**.

---

## 0) Variables (recommended)

Adjust these variables before starting:

```bash
GAIA_VERSION="v25.2.0"
CHAIN_ID="cosmoshub-4"
NODE_MONIKER="Cumulo"
GAIA_HOME="$HOME/.gaia"
```

---

## 1) System Requirements

### Hardware (recommended minimum)
- CPU: 4–8 vCPU
- RAM: 16–32 GB
- Storage: NVMe SSD recommended (Cosmos Hub grows steadily; plan >1TB)
- Network: 1 Gbps

### Software
- Ubuntu 22.04 LTS
- `git`, `make`, `gcc`, `curl`, `wget`, `jq`
- Go installed and configured

Install dependencies:

```bash
sudo apt-get update
sudo apt-get install -y git make gcc jq curl wget build-essential ca-certificates
```

---

## 2) Install Go (if not installed)

Check existing installation:

```bash
go version
```

If missing, install a stable version (example 1.22.x):

```bash
cd $HOME
GO_VERSION="1.22.6"
wget https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz

echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc

go version
```

---

## 3) Build and Install Gaia

```bash
cd $HOME
rm -rf cosmos
git clone https://github.com/cosmos/gaia cosmos
cd cosmos
git checkout $GAIA_VERSION
make install
```

Verify version:

```bash
gaiad version
# expected: v25.2.0
```

---

## 4) Initialize the Node

```bash
gaiad init "$NODE_MONIKER" --chain-id "$CHAIN_ID"
```

Directory created:

```text
$HOME/.gaia
```

---

## 5) Download Genesis and Addrbook

### Genesis
```bash
wget -O "$GAIA_HOME/config/genesis.json"   https://snapshots.polkachu.com/genesis/cosmos/genesis.json --inet4-only
```

### Addrbook
```bash
wget -O "$GAIA_HOME/config/addrbook.json"   https://server-1.itrocket.net/mainnet/cosmoshub/addrbook.json
```

---

## 6) Configure Seeds and Persistent Peers

> Keep seed and peer lists updated if connectivity issues arise.

```bash
SEEDS="00bf1f9d3c65137dc99c40cd03864384ce0ef7c3@cosmoshub-mainnet-seed.itrocket.net:34656"

PEERS="cbc6e2c364fec853fc74f01d4926c6046d9b2067@cosmoshub-mainnet-peer.itrocket.net:34656,36ad7bacc3a18b4deb647c60a0c1d8bbd24fde39@82.113.25.131:26656,27ad834c62dbefc5beb74be7575515927bd07c58@37.120.245.50:26656,0add711ee2dcedcfb4c575aa1ace3f4995c8d731@170.64.218.141:26090,bd2b5b30ee1a6f3d983bb3c1a083ea37aff18ce1@18.142.7.52:26656,7b15dce221b13ca353187b4f7219a94db6b71ad3@185.119.118.109:2000,be43400a8688132bd225b8cf9aad16042a37dac1@139.59.75.73:26090,66ca3161c5532da890815e40826ddbbbe2cb7f6c@176.9.101.44:26656,1a2ec6b643e4f6fda476dc94f579c2035e1de33f@178.128.238.183:26090,a1dc1d64c4412e0f9bf9ff6111b18fd68bc38d05@46.101.130.205:26090,c9326664143738dd027d96d658a28cb5dcc6c198@134.65.192.166:26656"
```

Apply configuration:

```bash
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}"        -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}"        "$GAIA_HOME/config/config.toml"
```

---

## 7) Configure Pruning

```bash
sed -i 's/^pruning *=.*/pruning = "custom"/' "$GAIA_HOME/config/app.toml"
sed -i 's/^pruning-keep-recent *=.*/pruning-keep-recent = "100"/' "$GAIA_HOME/config/app.toml"
sed -i 's/^pruning-interval *=.*/pruning-interval = "19"/' "$GAIA_HOME/config/app.toml"
```

---

## 8) Gas Price, Metrics and Indexer

### Minimum gas price
```bash
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0025uatom"|g'   "$GAIA_HOME/config/app.toml"
```

### Enable Prometheus
```bash
sed -i 's/prometheus = false/prometheus = true/'   "$GAIA_HOME/config/config.toml"
```

### Disable transaction indexer
```bash
sed -i 's/^indexer *=.*/indexer = "null"/'   "$GAIA_HOME/config/config.toml"
```

---

## 9) Create systemd Service

```bash
sudo tee /etc/systemd/system/gaiad.service > /dev/null <<EOF
[Unit]
Description=Cosmos Hub node (gaiad)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=$USER
WorkingDirectory=$HOME/.gaia
ExecStart=$(which gaiad) start --home $HOME/.gaia
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable gaiad
sudo systemctl start gaiad
```

Logs:

```bash
sudo journalctl -u gaiad -f --no-hostname -o cat
```

---

## 10) Verify Synchronization

```bash
gaiad status 2>/dev/null | jq .sync_info
```

When `catching_up` is `false`, the node is synced.

---

## 11) Create Wallet

```bash
WALLET="cumwallet"
gaiad keys add "$WALLET"
```

Check balance (after funding):

```bash
gaiad query bank balances "$(gaiad keys show "$WALLET" -a)"
```

---

## 12) Prepare validator.json

Get validator consensus pubkey:

```bash
gaiad tendermint show-validator
```

Example `validator.json`:

```json
{
  "pubkey": {
    "@type": "/cosmos.crypto.ed25519.PubKey",
    "key": "<YOUR_VALIDATOR_PUBKEY_BASE64>"
  },
  "amount": "1000000uatom",
  "moniker": "Cumulo",
  "identity": "77158D6796D16CD0",
  "website": "https://cumulo.pro",
  "security": "info@cumulo.pro",
  "details": "Cumulo is a professional validator focused on security, reliability and long-term network contribution.",
  "commission-rate": "0.05",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
```

---

## 13) Create the Validator

```bash
gaiad tx staking create-validator validator.json   --from "$WALLET"   --chain-id "$CHAIN_ID"   --node tcp://127.0.0.1:26657   --gas 450000   --fees 3000uatom   -y
```

Verify:

```bash
VAL_ADDR="$(gaiad keys show "$WALLET" --bech val -a)"
gaiad query staking validator "$VAL_ADDR"
```

---

## 14) Final Checklist

- [ ] gaiad v25.2.0 installed
- [ ] Genesis and addrbook in place
- [ ] Seeds and peers configured
- [ ] Pruning enabled
- [ ] Minimum gas price set
- [ ] Prometheus enabled
- [ ] Indexer disabled
- [ ] systemd service running
- [ ] Node fully synced
- [ ] Wallet backed up securely
- [ ] Validator created and verified on-chain

---

**End of document**
