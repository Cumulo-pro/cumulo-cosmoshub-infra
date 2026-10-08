# Cumulo: Cosmos Hub Infrastructure

This repository documents the infrastructure, configuration, and operational practices used by **Cumulo** to run and maintain a **Cosmos Hub (cosmoshub-4)** node and validator.

The goal of this repository is to provide **transparent, auditable, and continuously updated documentation** of our real-world operations as a professional validator and infrastructure provider.

This is not a generic tutorial.  
Everything documented here reflects **how we actually operate the node in production**.

---

## 🧭 Scope

This repository covers:

- Deployment of a Cosmos Hub full node and validator
- Node configuration and pruning strategy
- Systemd-based service management
- Validator setup and operational policies
- Monitoring and observability
- Upgrade and maintenance procedures
- Security and key-management practices
- Ongoing operational changes and upgrades
- Incident reports, performance tests and on-chain studies

---

## 🌐 Network

- **Network:** Cosmos Hub  
- **Chain ID:** `cosmoshub-4`  
- **Client:** `gaiad`  
- **Current version:** `v28.3.1`  

---

## 🔌 Public Endpoints

| Interface | Endpoint |
|---|---|
| RPC (incl. WebSocket) | `https://rpc.cosmos.cumulo.com.es` |
| REST API | `https://api.cosmos.cumulo.com.es` |
| gRPC | `grpc.cosmos.cumulo.com.es:443` |

All endpoints are served over TLS, without an API key, with CORS enabled for browser applications.

---

## 📡 EndPoint Scan

Cumulo runs **EndPoint Scan**, a public multi-region health check (US · EU · CA · AS) of validator-run Cosmos Hub endpoints. It reports latency, a reliability score, CORS and TLS certificate status, refreshed every 5 minutes.

- RPC Mainnet: <https://cumulo.pro/services/cosmos/rpcscan>
- API Mainnet: <https://cumulo.pro/services/cosmos/apiscan>
- RPC Testnet: <https://cumulo.pro/services/cosmos_testnet/rpcscan>
- API Testnet: <https://cumulo.pro/services/cosmos_testnet/apiscan>

The list of monitored endpoints is maintained in [`data/validators.json`](data/validators.json) (mainnet) and [`data/validators_testnet.json`](data/validators_testnet.json) (testnet). Any provider can request inclusion by opening a pull request against these files.

---

## 📝 Reports

Operational reports, newest first:

| Date | Report |
|---|---|
| 2026-10-08 | [Cosmos Hub RPC — Performance & Compliance Test Report](docs/2026-10-08-cosmos-hub-rpc-performance-tests.md) |
| 2026-09-05 | [Cosmos Hub mainnet app-hash event: v28.1.0 emergency patch](docs/2026-09-05-cosmos-hub-v28.1.0-incident-report.md) |

---

## 🔬 Studies

On-chain research published by Cumulo:

- [Tokenfactory and Wasm bindings (v2)](studies/tokenfactory-wasm-bindings-2026-04-v2.md) · [first version](studies/tokenfactory-wasm-bindings-2026-04.md)
- [Gas surcharge on multisend transactions](studies/gas-surcharge-multisend-2026-03/README.md)

---

## 🧑‍🚀 Cumulo as an Operator

Cumulo is a multi-chain infrastructure operator running production-grade validators and nodes across multiple ecosystems.

Our focus as an operator is:
- Security-first infrastructure
- High availability and operational reliability
- Transparent processes
- Long-term commitment to the networks we support
- Public documentation of real operational practices

This repository is part of that commitment.

---

## 📁 Repository Structure

```text
.
├─ README.md
├─ cosmoshub-4/                 # Node and validator documentation
│  ├─ install.md                # Node installation and configuration
│  ├─ cosmoshub_testnet_fullnode_validator_setup.md
│  ├─ COSMOS_CLI_COMMAND_REFERENCE.md
│  ├─ Cosmos_CometBFT_Metrics.md
│  ├─ dashboard-*.json          # Grafana dashboard
│  ├─ horcrux/                  # Threshold signing (Horcrux)
│  └─ validator/                # Validator setup, wallet and policies
├─ data/                        # Endpoint lists used by EndPoint Scan
│  ├─ validators.json
│  └─ validators_testnet.json
├─ docs/                        # Incident reports and test reports
├─ ibc/                         # IBC relaying
│  ├─ collector/
│  └─ hermes/                   # Hermes config and channel definitions
└─ studies/                     # On-chain research
```
