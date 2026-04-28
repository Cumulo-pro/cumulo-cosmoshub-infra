# Horcrux – Cosmos Testnet Implementation

## Overview

This document describes the implementation of [Horcrux](https://github.com/strangelove-ventures/horcrux) on the Cosmos Testnet as a distributed signing (MPC) solution to protect the validator private key. The setup follows a threshold signature scheme (TSS) with a **2-of-3 configuration**: signing requires agreement from at least 2 of the 3 signer nodes, eliminating single points of failure while maintaining high availability.

---

## Architecture

The deployment uses **2 sentry nodes** and **3 signer nodes**, distributed across different infrastructure providers and geographic regions.

```
                        ┌─────────────┐
                        │  P2P Network │
                        └──────┬──────┘
                               │ port 26656
              ┌────────────────┼────────────────┐
              │                                 │
       ┌──────▼──────┐                  ┌───────▼─────┐
       │   Sentry 1  │                  │   Sentry 2  │
       │  (port 1234)│                  │  (port 1235)│
       └──────┬──────┘                  └───────┬─────┘
              │                                 │
     ┌────────┼──────────────┬──────────────────┘
     │        │              │
┌────▼───┐ ┌──▼─────┐ ┌──────▼──┐
│Signer 1│ │Signer 2│ │Signer 3 │
│port2222│ │port2222│ │port 2222│
└────────┘ └────────┘ └─────────┘
```

### Threshold

| Parameter | Value |
|-----------|-------|
| Total signers | 3 |
| Signing threshold | 2 of 3 |
| Total sentries | 2 |

This means the validator continues signing blocks normally even if one signer node becomes unavailable.

---

## Node Distribution

### Sentry Nodes

| Node | Validator port |
|------|----------|----------------|
| Sentry 1 | 1234 |
| Sentry 2 | 1235 |

> **Important:** `priv_validator_key.json` was removed from all sentry nodes after the migration. Sentries act solely as a network interface — they hold no signing material.

### Signer Nodes

| Node | Provider |
|------|----------|
| Signer 1 | OVH |
| Signer 2 | Arsys |
| Signer 3 | DigitalOcean |

Signers are intentionally distributed across different providers and regions to reduce correlated failure risk.

---

## Files Deployed per Signer

Each signer node received the following files under `~/.horcrux/`:

| File | Description |
|------|-------------|
| `config.yaml` | Shared Horcrux configuration (sentry endpoints, cosigner peers, threshold, timeouts) |
| `provider_shard.json` | This signer's share of the validator private key (ed25519 shard) |
| `ecies_keys.json` | ECIES keypair for encrypted inter-signer communication |
| `provider_priv_validator_state.json` | Consensus state, copied from the original validator and truncated to `height/round/step` |

---

## Implementation Steps

### 1. Installation

Horcrux v3.2.3 was installed on all signer nodes:

```bash
wget https://github.com/strangelove-ventures/horcrux/releases/download/v3.2.3/horcrux_linux-amd64
mv horcrux_linux-amd64 horcrux
sudo cp horcrux /usr/local/bin/
sudo chmod +x /usr/local/bin/horcrux
```

### 2. Shared Configuration

`horcrux config init` was run with the sentry node addresses and all cosigner peers, specifying a threshold of 2:

```bash
horcrux config init \
  --node "tcp://<sentry-1>:1234" \
  --node "tcp://<sentry-2>:1235" \
  --cosigner "tcp://<signer-1>:2222" \
  --cosigner "tcp://<signer-2>:2222" \
  --cosigner "tcp://<signer-3>:2222" \
  --threshold 2 \
  --grpc-timeout 1000ms \
  --raft-timeout 1000ms
```

### 3. Key Sharding

**ECIES communication keys** — one keypair per cosigner for encrypted inter-signer communication:

```bash
horcrux create-ecies-shards --shards 3
```

**Validator key shards** — the original `priv_validator_key.json` was split into 3 shares:

```bash
horcrux create-ed25519-shards \
  --chain-id provider \
  --key-file ./priv_validator_key.json \
  --threshold 2 \
  --shards 3
```

Each signer received only its own shard. No full key material exists on any single node after this step.

### 4. Consensus State Migration

The original validator was stopped and the current `priv_validator_state.json` was read to obtain the last signed height. A truncated version (containing only `height`, `round`, and `step`, with no signature or signbytes) was written to `~/.horcrux/state/provider_priv_validator_state.json` on all three signers before starting Horcrux.

### 5. Horcrux Systemd Service

A systemd service was created on each signer node and enabled to run on boot:

```bash
sudo systemctl daemon-reload
sudo systemctl enable horcrux
sudo systemctl start horcrux
```

### 6. Sentry Node Configuration

Each sentry was configured to listen for the remote validator connection by setting `priv_validator_laddr` in `config.toml`:

```
priv_validator_laddr = "tcp://0.0.0.0:<port>"
```

The `priv_validator_key.json` file was then deleted from each sentry, and the node service was restarted.

---

## Network & Firewall

| Node type | Port | Access |
|-----------|------|--------|
| Signer | 2222 | Restricted to sentry IPs only |
| Sentry | 1234 / 1235 | Restricted to signer IPs only |
| Sentry | 26656 | Public (P2P) |
| Signer | 22 | SSH only |

---

## Verification

Once all signers and sentries were running, signing activity was confirmed via logs:

```bash
journalctl -u horcrux -f
# ✅ Successfully signed proposal at height XXXXXX
```

---

## References

- [Horcrux GitHub](https://github.com/strangelove-ventures/horcrux)
- [Cumulo Horcrux Architecture](https://github.com/CosmosContracts/docs)
