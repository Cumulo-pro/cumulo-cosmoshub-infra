# IBC Relayer Activity Collector

Node.js service that collects the full IBC transaction history for each Cumulo relayer wallet and generates a cumulative JSON file consumed by the Activity Tracker dashboard.

---

## Server Location

| Resource | Path |
|---|---|
| Script | `/opt/ibc-relayer-collector/collector-relayers.js` |
| Output JSON | `/var/lib/cumulo-ibc-relayer/activity.json` |
| systemd service | `ibc-relayer-collector` |
| Public URL | `https://data.relayers.cumulo.com.es/activity.json` |

---

## Configured Wallets

| Key | Chain | Network | Address |
|---|---|---|---|
| `celestia_mainnet` | celestia | mainnet | `celestia1ahzs5c3g3z9u9wxwxmcmqgu6htagp4md627uas` |
| `cosmoshub_mainnet` | cosmoshub-4 | mainnet | `cosmos1t24lx6zfx7hqrexppk6gh2yavytv39xqfg5u9f` |
| `xrplevm_mainnet` | xrplevm_1440000-1 | mainnet | `ethm1jl3w0f8r6688ghd30he8ddjtnmtuevkvtjwj6r` |
| `injective_mainnet` | injective-1 | mainnet | `inj1jxr2se9ut4yxqz4jemz3qn9knkaklwngvc3c3w` |
| `seda_mainnet` | seda-1 | mainnet | `seda10l7d6fe3l68zghf6fyrpj5xqpj748sfg6jszcv` |
| `osmosis_mainnet` | osmosis-1 | mainnet | `osmo1djcckm2zw4xyz2lt6yg6euhgnrwp29k38n2l86` |
| `dymension_mainnet` | dymension_1100-1 | mainnet | `dym1qqev5krh9ge9tctc4gkfs6l54anuunzjngwkcl` |
| `celestia_testnet` | mocha-4 | testnet | `celestia1mlf0e4cx657a2mkt9t9clcw78zhqut4629v3s7` |
| `cosmoshub_testnet` | provider | testnet | `cosmos1qqsrv08cvqpxcctmr7pyy3c5g3ukvurktejy2q` |

---

## Adding a New Wallet

Edit the `WALLETS` array in the script:

```bash
nano /opt/ibc-relayer-collector/collector-relayers.js
```

Add an entry with this format:

```javascript
{
  key:          "chain_mainnet",         // unique identifier, snake_case
  chain:        "chain-id",              // exact chain_id
  network:      "mainnet",               // "mainnet" or "testnet"
  label:        "Chain Mainnet",         // display name in the dashboard
  address:      "addr1...",              // relayer wallet address
  rest:         "https://api.chain.com", // REST endpoint (prefer Cumulo's own)
  explorer_tx:  "https://mintscan.io/chain/tx/", // null if no explorer
  denom:        "utoken",                // native fee denom
  denom_display:"TOKEN",                 // symbol shown in the dashboard
  exp:          1e6,                     // decimal exponent: 1e6 or 1e18
},
```

**Decimals by chain:**
- `1e6` → Celestia (TIA), Cosmos Hub (ATOM), Osmosis (OSMO)
- `1e18` → Injective (INJ), SEDA (SEDA), XRPL EVM (XRP), Dymension (DYM)

Restart the service after saving:

```bash
sudo systemctl restart ibc-relayer-collector
systemctl status ibc-relayer-collector
```

Verify the new wallet appears in the logs:

```bash
journalctl -u ibc-relayer-collector -n 50
```

---

## Useful Commands

```bash
# Check service status
systemctl status ibc-relayer-collector

# Run once manually (without affecting the service)
node /opt/ibc-relayer-collector/collector-relayers.js

# Stream logs in real time
journalctl -u ibc-relayer-collector -f

# Inspect the last bytes of the generated JSON
tail -c 500 /var/lib/cumulo-ibc-relayer/activity.json

# Restart
sudo systemctl restart ibc-relayer-collector
```

---

## How It Works

- Runs in `--watch` mode, executing every **1 hour**
- Each run fetches only new transactions since the last known block height (incremental)
- Writes atomically (`activity.json.tmp` → `activity.json`) to prevent partial reads
- The dashboard consumes it via `activity_proxy.php` with a 5-minute client cache

## Collected Transaction Types

| Type | Category |
|---|---|
| MsgRecvPacket | relay |
| MsgAcknowledgement | relay |
| MsgTimeout / MsgTimeoutOnClose | relay |
| MsgUpdateClient | maintenance |
| MsgCreateClient | maintenance |
