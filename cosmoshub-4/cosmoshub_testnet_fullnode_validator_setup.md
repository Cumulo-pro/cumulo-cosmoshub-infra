# Cosmos Hub Testnet — Full Node Installation & Validator Setup (Gaia)

This document describes **all required steps** to install, configure, and operate a **Cosmos Hub testnet** node and, if applicable, **create a validator** using **Gaia (`gaiad`)**.

> **Network**
> - **Gaia version:** `v27.0.0-rc0`
> - **Chain ID:** `provider` (testnet)

---

## 0) Variables (recommended)

Adjust these variables before starting:

```bash
GAIA_VERSION="v27.0.0-rc0"
CHAIN_ID="provider"
NODE_MONIKER="CumuloRPC"
GAIA_HOME="$HOME/.gaia"
```

---

## 1) System Requirements

### Hardware (recommended minimum)
- CPU: 4–8 vCPU
- RAM: 16–32 GB
- Storage: NVMe SSD recommended (plan plenty of disk; testnets can still grow fast)
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
cd "$HOME"
GO_VERSION="1.22.6"
wget "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"

echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc

go version
```

---

## 3) Build and Install Gaia

```bash
cd "$HOME"
rm -rf cosmos
git clone https://github.com/cosmos/gaia cosmos
cd cosmos
git checkout "$GAIA_VERSION"
make install
```

Verify version:

```bash
gaiad version
# expected: v27.0.0-rc0
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

## 5) Download Genesis and Addrbook (Testnet)

### Genesis
```bash
wget -O "$GAIA_HOME/config/genesis.json" \
  https://snapshots.polkachu.com/testnet-genesis/cosmos/genesis.json \
  --inet4-only
```

### Addrbook
```bash
wget -O "$GAIA_HOME/config/addrbook.json" \
  https://server-2.itrocket.net/testnet/cosmos/addrbook.json
```

---

## 6) Configure Seeds and Persistent Peers

> Keep seed and peer lists updated if connectivity issues arise.

```bash
SEEDS="84871382a3ffa5e781034b6519126f2d5ea29f15@cosmos-testnet-seed.itrocket.net:21656,ade4d8bc8cbe014af6ebdf3cb7b1e9ad36f412c0@testnet-seeds.polkachu.com:14956"

PEERS="5359d38055601c523dbbec5c6fd7168d36354c16@cosmos-testnet-peer.itrocket.net:21656,77427588dd30d410736c241e31375cc747194c83@65.109.111.234:34656,5b0e5469e6615808ffb0bcf3228ca8d7cf3b9e94@144.76.30.36:15621,49d75c6094c006b6f2758e45457c1f3d6002ce7a@188.166.8.68:25002,7a4d3c66261d33288055750a872d47c55769ad7a@65.108.127.249:2000,bc24e4667f9db8a0265c9d8f31516ef1fa7c9368@168.119.37.164:30656,c299ee06a11addcccb4cbd0d600ca521ff143ff1@65.109.25.113:14956,ae35eb87fbe117392159e8ec8fa65f7f14cd613f@37.27.0.90:26656,c2ef8a0e7408d0a8dc8a546e4d5c17a42977639e@23.111.23.233:36656,6ce07afdbb0a325a491522bfa4e45ebe41e0aaee@148.251.245.34:13356,a3825c8fb9b8fc6e95b10279ccd898321fa37c20@54.37.31.127:14956"
```

Apply configuration:

```bash
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" \
       "$GAIA_HOME/config/config.toml"
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
sed -i 's|minimum-gas-prices =.*|minimum-gas-prices = "0.0025uatom"|g' "$GAIA_HOME/config/app.toml"
```

### Enable Prometheus
```bash
sed -i 's/prometheus = false/prometheus = true/' "$GAIA_HOME/config/config.toml"
```

### Disable transaction indexer
```bash
sed -i 's/^indexer *=.*/indexer = "null"/' "$GAIA_HOME/config/config.toml"
```

---

## 9) Create systemd Service

```bash
sudo tee /etc/systemd/system/gaiad.service > /dev/null <<EOF
[Unit]
Description=Cosmos Hub testnet node (gaiad)
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

## 12) Prepare validator.json (optional)

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

## 13) Create the Validator (optional)

```bash
gaiad tx staking create-validator validator.json \
  --from "$WALLET" \
  --chain-id "$CHAIN_ID" \
  --node tcp://127.0.0.1:26657 \
  --gas 450000 \
  --fees 3000uatom \
  -y
```

Verify:

```bash
VAL_ADDR="$(gaiad keys show "$WALLET" --bech val -a)"
gaiad query staking validator "$VAL_ADDR"
```

---

## 14) Final Checklist

- [ ] `gaiad $GAIA_VERSION` installed
- [ ] Genesis and addrbook in place (testnet)
- [ ] Seeds and peers configured
- [ ] Pruning enabled
- [ ] Minimum gas price set
- [ ] Prometheus enabled
- [ ] Indexer disabled
- [ ] systemd service running
- [ ] Node fully synced (`catching_up=false`)
- [ ] Wallet backed up securely
- [ ] Validator created and verified on-chain (if applicable)

---

**End of document**
