# Hermes IBC Relayer : cumulo.me

Operational guide for running a Hermes IBC relayer using Cumulo's public infrastructure.

| Path | Type | Status |
|---|---|---|
| `mocha-4 ↔ provider` | Testnet - own channel | ✅ Live |
| `celestia ↔ cosmoshub-4` | Mainnet - own channel | ✅ Live |
| `celestia ↔ xrplevm_1440000-1` | Mainnet - own channel | ✅ Live |
| `celestia ↔ injective-1` | Mainnet - own channel | ✅ Live |
| `cosmoshub-4 ↔ xrplevm_1440000-1` | Mainnet - relayer on Peersyst channel | ✅ Live |
| `cosmoshub-4 ↔ injective-1` | Mainnet - relayer on existing channel | ✅ Live |
| `xrplevm_1440000-1 ↔ injective-1` | Mainnet - relayer on Peersyst channel | ✅ Live |
| `celestia ↔ seda-1` | Mainnet - channel pending | 🔜 Pending |
| `celestia ↔ osmosis-1` | Mainnet - channel pending | 🔜 Pending |

> **Hermes version:** v1.13.2

---

## Active Channels

### Testnet - mocha-4 ↔ provider

| | mocha-4 | provider |
|---|---|---|
| **client** | `07-tendermint-633` | `07-tendermint-436` |
| **connection** | `connection-684` | `connection-301` |
| **channel** | `channel-464` | `channel-586` |
| **port** | `transfer` | `transfer` |

### Mainnet - celestia ↔ cosmoshub-4

> **First IBC channel between Celestia mainnet and Cosmos Hub**

| | celestia | cosmoshub-4 |
|---|---|---|
| **client** | `07-tendermint-151` | `07-tendermint-1480` |
| **connection** | `connection-103` | `connection-1272` |
| **channel** | `channel-278` | `channel-1879` |
| **port** | `transfer` | `transfer` |

### Mainnet - celestia ↔ xrplevm_1440000-1

> **First IBC channel between Celestia mainnet and XRPL EVM**

| | celestia | xrplevm_1440000-1 |
|---|---|---|
| **client** | `07-tendermint-165` | `07-tendermint-13` |
| **connection** | `connection-105` | `connection-6` |
| **channel** | `channel-280` | `channel-6` |
| **port** | `transfer` | `transfer` |

### Mainnet - celestia ↔ injective-1

> **First active IBC channel between Celestia mainnet and Injective**

| | celestia | injective-1 |
|---|---|---|
| **client** | `07-tendermint-166` | `07-tendermint-327` |
| **connection** | `connection-106` | `connection-331` |
| **channel** | `channel-281` | `channel-453` |
| **port** | `transfer` | `transfer` |

### Mainnet - cosmoshub-4 ↔ xrplevm_1440000-1 (Peersyst channel)

| | cosmoshub-4 | xrplevm_1440000-1 |
|---|---|---|
| **client** | `07-tendermint-1411` | `07-tendermint-2` |
| **connection** | `connection-1134` | `connection-2` |
| **channel** | `channel-1377` | `channel-2` |
| **port** | `transfer` | `transfer` |

### Mainnet - cosmoshub-4 ↔ injective-1

| | cosmoshub-4 | injective-1 |
|---|---|---|
| **client** | `07-tendermint-444` | `07-tendermint-3` |
| **connection** | `connection-376` | `connection-1` |
| **channel** | `channel-211` | `channel-0` |
| **port** | `transfer` | `transfer` |

### Mainnet - xrplevm_1440000-1 ↔ injective-1 (Peersyst channel)

| | xrplevm_1440000-1 | injective-1 |
|---|---|---|
| **client** | `07-tendermint-0` | `07-tendermint-314` |
| **connection** | `connection-0` | `connection-314` |
| **channel** | `channel-0` | `channel-436` |
| **port** | `transfer` | `transfer` |

---

## Relayer Wallets

| Chain | Address |
|---|---|
| mocha-4 | `celestia1mlf0e4cx657a2mkt9t9clcw78zhqut4629v3s7` |
| provider | `cosmos1qqsrv08cvqpxcctmr7pyy3c5g3ukvurktejy2q` |
| celestia | `celestia1ahzs5c3g3z9u9wxwxmcmqgu6htagp4md627uas` |
| cosmoshub-4 | `cosmos1t24lx6zfx7hqrexppk6gh2yavytv39xqfg5u9f` |
| xrplevm_1440000-1 | `ethm1jl3w0f8r6688ghd30he8ddjtnmtuevkvtjwj6r` |
| injective-1 | `inj1xr30he02u6wpkzqj7c2g74qq7fvu68vywcl5fr` |
| seda-1 | *(pending channel)* |
| osmosis-1 | *(pending channel)* |

---

## chain-registry PRs

| PR | Channels |
|---|---|
| https://github.com/cosmos/chain-registry/pull/7712 | celestia ↔ cosmoshub + celestia ↔ xrplevm |
| Pending | celestia ↔ injective |

---

## Infrastructure (cumulo.me)

### Testnet

| Parameter | Celestia Mocha-4 | Cosmos Hub Provider |
|---|---|---|
| chain-id | `mocha-4` | `provider` |
| RPC | `https://mocha.celestia.rpc.cumulo.me` | `https://cosmos.rpc.testnet.cumulo.me` |
| API | `https://mocha.api.cumulo.me` | `https://cosmos.api.testnet.cumulo.me` |
| gRPC | `https://mocha.grpc.cumulo.me` | `https://cosmos.grpc.testnet.cumulo.me` |
| denom | `utia` | `uatom` |

### Mainnet

| Parameter | Celestia | Cosmos Hub | XRPL EVM | Injective | SEDA | Osmosis |
|---|---|---|---|---|---|---|
| chain-id | `celestia` | `cosmoshub-4` | `xrplevm_1440000-1` | `injective-1` | `seda-1` | `osmosis-1` |
| RPC | `https://celestia.cumulo.org.es` | `https://rpc.cosmos.cumulo.com.es` | `https://rpc.xrpl.cumulo.org.es` | `https://sentry.tm.injective.network:443` | `https://seda.rpc.cumulo.org.es` | `https://rpc.osmosis.zone` |
| API | `https://celestia.api.cumulo.org.es` | `https://api.cosmos.cumulo.com.es` | `https://api.xrpl.cumulo.org.es` | `https://sentry.lcd.injective.network` | `https://seda.api.cumulo.org.es` | `https://lcd.osmosis.zone` |
| gRPC | `https://celestia.grpc.cumulo.org.es` | `https://grpc.cosmos.cumulo.com.es` | `https://grpc.xrpl.cumulo.org.es` | `https://sentry.chain.grpc.injective.network:443` | `https://seda.grpc.cumulo.org.es` | `https://grpc.osmosis.zone:443` |
| denom | `utia` | `uatom` | `axrp` | `inj` | `aseda` | `uosmo` |
| addr prefix | `celestia` | `cosmos` | `ethm` | `inj` | `seda` | `osmo` |
| node type | Own node | Own node | Own node | Injective Sentry | Own node | Public |

---

## Installation

### 1. Install Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
rustup update stable
sudo apt update && sudo apt install -y build-essential pkg-config libssl-dev git curl jq
```

### 2. Install Hermes

```bash
cargo install ibc-relayer-cli --bin hermes --locked
hermes version
mkdir -p $HOME/.hermes
```

---

## Wallet Setup

```bash
# Generate wallets
celestia-appd keys add relayer-mocha      --keyring-backend test --output json 2>&1 | tee $HOME/.hermes/mocha-key.json
gaiad         keys add relayer-provider   --keyring-backend test --output json 2>&1 | tee $HOME/.hermes/provider-key.json
celestia-appd keys add relayer-celestia   --keyring-backend test --output json 2>&1 | tee $HOME/.hermes/celestia-mainnet-key.json
gaiad         keys add relayer-cosmoshub  --keyring-backend test --output json 2>&1 | tee $HOME/.hermes/cosmoshub-key.json
exrpd         keys add relayer-xrplevm    --keyring-backend test --output json 2>&1 | tee $HOME/.hermes/xrplevm-key.json
injectived    keys add relayer-injective  --keyring-backend test --output json 2>&1 | tee $HOME/.hermes/injective-key.json
sedad         keys add relayer-seda       --keyring-backend test --output json 2>&1 | tee $HOME/.hermes/seda-key.json
osmosisd      keys add relayer-osmosis    --keyring-backend test --output json 2>&1 | tee $HOME/.hermes/osmosis-key.json

# Extract mnemonics
for chain in mocha provider celestia-mainnet cosmoshub xrplevm injective seda osmosis; do
  jq -r '.mnemonic' $HOME/.hermes/${chain}-key.json > $HOME/.hermes/${chain}-mnemonic.txt
done

# Import into Hermes
hermes keys add --chain mocha-4           --mnemonic-file $HOME/.hermes/mocha-mnemonic.txt
hermes keys add --chain provider          --mnemonic-file $HOME/.hermes/provider-mnemonic.txt
hermes keys add --chain celestia          --mnemonic-file $HOME/.hermes/celestia-mainnet-mnemonic.txt
hermes keys add --chain cosmoshub-4       --mnemonic-file $HOME/.hermes/cosmoshub-mnemonic.txt
hermes keys add --chain xrplevm_1440000-1 --mnemonic-file $HOME/.hermes/xrplevm-mnemonic.txt
hermes keys add --chain injective-1       --mnemonic-file $HOME/.hermes/injective-mnemonic.txt
hermes keys add --chain seda-1            --mnemonic-file $HOME/.hermes/seda-mnemonic.txt
hermes keys add --chain osmosis-1         --mnemonic-file $HOME/.hermes/osmosis-mnemonic.txt
```

> **Note:** Injective uses ethermint derivation. The address generated by `injectived keys add` may differ from the one Hermes derives. Always use the address shown by `hermes keys list --chain injective-1`.

---

## Health Check

```bash
hermes health-check
```

### Expected warnings (non-blocking)

| Warning | Reason | Action |
|---|---|---|
| `potential compat_mode misconfiguration` | CometBFT v0.38, v0.37 configured | Ignore - intentional |
| `could not infer compatibility mode` | Injective with Sentry endpoints | Ignore |
| `does not provide minimum gas price` | Node doesn't advertise min gas | Ignore |

---

## Create IBC Channels

```bash
# celestia ↔ cosmoshub-4
hermes create channel --a-chain celestia --b-chain cosmoshub-4 \
  --a-port transfer --b-port transfer --new-client-connection --yes

# celestia ↔ xrplevm_1440000-1
hermes create channel --a-chain celestia --b-chain xrplevm_1440000-1 \
  --a-port transfer --b-port transfer --new-client-connection --yes

# celestia ↔ injective-1
hermes create channel --a-chain celestia --b-chain injective-1 \
  --a-port transfer --b-port transfer --new-client-connection --yes

# celestia ↔ seda-1 (pending)
hermes create channel --a-chain celestia --b-chain seda-1 \
  --a-port transfer --b-port transfer --new-client-connection --yes

# celestia ↔ osmosis-1 (pending)
hermes create channel --a-chain celestia --b-chain osmosis-1 \
  --a-port transfer --b-port transfer --new-client-connection --yes
```

---

## Run as systemd Service

```bash
sudo tee /etc/systemd/system/hermes.service > /dev/null <<EOF
[Unit]
Description=Hermes IBC Relayer -- cumulo.me
After=network-online.target

[Service]
User=$USER
ExecStart=$(which hermes) start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="RUST_LOG=info"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable hermes
sudo systemctl start hermes
```

---

## Monitoring

```
https://hermes-metrics.cumulo.me/metrics
```

```yaml
- job_name: hermes_ibc_relayer
  scheme: https
  metrics_path: /metrics
  static_configs:
    - targets: ['hermes-metrics.cumulo.me']
      labels:
        instance: 'hermes-cumulo'
```

---

## Maintenance

### Stuck Packets

```bash
# Testnet
hermes clear packets --chain mocha-4  --port transfer --channel channel-464
hermes clear packets --chain provider --port transfer --channel channel-586

# celestia <> cosmoshub-4
hermes clear packets --chain celestia    --port transfer --channel channel-278
hermes clear packets --chain cosmoshub-4 --port transfer --channel channel-1879

# celestia <> xrplevm
hermes clear packets --chain celestia          --port transfer --channel channel-280
hermes clear packets --chain xrplevm_1440000-1 --port transfer --channel channel-6

# celestia <> injective
hermes clear packets --chain celestia    --port transfer --channel channel-281
hermes clear packets --chain injective-1 --port transfer --channel channel-453

# cosmoshub-4 <> xrplevm
hermes clear packets --chain cosmoshub-4       --port transfer --channel channel-1377
hermes clear packets --chain xrplevm_1440000-1 --port transfer --channel channel-2

# cosmoshub-4 <> injective
hermes clear packets --chain cosmoshub-4 --port transfer --channel channel-211
hermes clear packets --chain injective-1 --port transfer --channel channel-0

# xrplevm <> injective
hermes clear packets --chain xrplevm_1440000-1 --port transfer --channel channel-0
hermes clear packets --chain injective-1       --port transfer --channel channel-436
```

### Wallet Balance

```bash
# Testnet
curl -s https://mocha.api.cumulo.me/cosmos/bank/v1beta1/balances/celestia1mlf0e4cx657a2mkt9t9clcw78zhqut4629v3s7 | jq .
curl -s https://cosmos.api.testnet.cumulo.me/cosmos/bank/v1beta1/balances/cosmos1qqsrv08cvqpxcctmr7pyy3c5g3ukvurktejy2q | jq .

# Mainnet
curl -s https://celestia.api.cumulo.org.es/cosmos/bank/v1beta1/balances/celestia1ahzs5c3g3z9u9wxwxmcmqgu6htagp4md627uas | jq .
curl -s https://api.cosmos.cumulo.com.es/cosmos/bank/v1beta1/balances/cosmos1t24lx6zfx7hqrexppk6gh2yavytv39xqfg5u9f | jq .
curl -s https://api.xrpl.cumulo.org.es/cosmos/bank/v1beta1/balances/ethm1jl3w0f8r6688ghd30he8ddjtnmtuevkvtjwj6r | jq .
curl -s https://sentry.lcd.injective.network/cosmos/bank/v1beta1/balances/inj1xr30he02u6wpkzqj7c2g74qq7fvu68vywcl5fr | jq .
```

---

## Known Issues & Fixes

### Injective address derivation

`injectived keys add` and Hermes may derive different addresses from the same mnemonic. Always use the address shown by `hermes keys list --chain injective-1` for the actual relayer wallet.

### gRPC transport error

nginx must use `http2` and `grpc_pass`:
```nginx
listen 443 ssl http2;
location / {
    grpc_pass grpc://127.0.0.1:<PORT>;
    grpc_read_timeout 600s;
    grpc_send_timeout 600s;
}
```

### XRPL EVM specific

- `trusting_period = "18hours"` - PoA chain with 1 day unbonding period
- `gas price = 600000000 axrp/gas` - minimum enforced by the node
- Binary: `exrpd`

### SEDA specific

- `gas price = 10000000000 aseda/gas`
- `account_prefix = "seda"`
- derivation: `cosmos`

---

## References

| Resource | URL |
|---|---|
| Hermes documentation | https://hermes.informal.systems |
| XRPL EVM IBC guide | https://docs.xrplevm.org/pages/users/sending-through-ibc |
| Celestia IBC Relayer Guide | https://docs.celestia.org/how-to-guides/ibc-relayer |
| cosmos/chain-registry IBC paths | https://github.com/cosmos/chain-registry/tree/master/_IBC |
| chain-registry PR #7712 | https://github.com/cosmos/chain-registry/pull/7712 |
| Hermes GitHub | https://github.com/informalsystems/hermes |
| Faucet Cosmos provider testnet | https://faucet.polypore.xyz |

---

*cumulo.me - IBC Initiative - June 2026*
