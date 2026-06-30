# IBC Relayer Infrastructure: cumulo.me

Cumulo operates an IBC relayer across 22 channels between 8 chains on Cosmos Hub mainnet and testnet.

## Components

| Directory | Description |
|---|---|
| [hermes/](hermes/) | Hermes relayer - config, channels, wallets, installation, maintenance |
| [collector/](collector/) | Activity collector - fetches on-chain tx history per wallet hourly |

## Monitoring

| Tool | URL |
|---|---|
| Channel status dashboard | https://cumulo.pro/services/ibc/ |
| On-chain activity tracker | https://cumulo.pro/services/ibc/activity |
| Prometheus metrics | https://hermes-metrics.cumulo.me/metrics |

## Active Paths

| Path | Channels |
|---|---|
| mocha-4 ↔ provider | 1 (testnet) |
| celestia ↔ cosmoshub-4 | 1 |
| celestia ↔ xrplevm_1440000-1 | 1 |
| celestia ↔ injective-1 | 1 |
| celestia ↔ osmosis-1 | 3 |
| celestia ↔ dymension_1100-1 | 1 |
| cosmoshub-4 ↔ xrplevm_1440000-1 | 1 |
| cosmoshub-4 ↔ injective-1 | 2 |
| cosmoshub-4 ↔ osmosis-1 | 8 |
| cosmoshub-4 ↔ seda-1 | 1 |
| cosmoshub-4 ↔ dymension_1100-1 | 1 |
| xrplevm_1440000-1 ↔ injective-1 | 1 |
| injective-1 ↔ osmosis-1 | 2 |
| seda-1 ↔ osmosis-1 | 1 |
| osmosis-1 ↔ dymension_1100-1 | 1 |

**Total: 22 channels · 8 chains**

## Relayer Wallets

| Chain | Network | Address |
|---|---|---|
| celestia | mainnet | `celestia1ahzs5c3g3z9u9wxwxmcmqgu6htagp4md627uas` |
| cosmoshub-4 | mainnet | `cosmos1t24lx6zfx7hqrexppk6gh2yavytv39xqfg5u9f` |
| xrplevm_1440000-1 | mainnet | `ethm1jl3w0f8r6688ghd30he8ddjtnmtuevkvtjwj6r` |
| injective-1 | mainnet | `inj1jxr2se9ut4yxqz4jemz3qn9knkaklwngvc3c3w` |
| seda-1 | mainnet | `seda10l7d6fe3l68zghf6fyrpj5xqpj748sfg6jszcv` |
| osmosis-1 | mainnet | `osmo1djcckm2zw4xyz2lt6yg6euhgnrwp29k38n2l86` |
| dymension_1100-1 | mainnet | `dym1qqev5krh9ge9tctc4gkfs6l54anuunzjngwkcl` |
| mocha-4 | testnet | `celestia1mlf0e4cx657a2mkt9t9clcw78zhqut4629v3s7` |
| provider | testnet | `cosmos1qqsrv08cvqpxcctmr7pyy3c5g3ukvurktejy2q` |
