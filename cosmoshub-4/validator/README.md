# Validator Setup & Policies — Cosmos Hub (cosmoshub-4)

This section documents how Cumulo configures, creates, and operates a **Cosmos Hub validator**, including
key management principles and on-chain policies.

This documentation reflects **production practices** and is designed to be auditable and delegation‑program ready.

---

## Scope

- Wallet and key management principles
- Validator metadata and configuration
- Validator creation procedure
- Operational policies (commission, self‑delegation, uptime)
- Day‑to‑day validator operations

> ⚠️ No private keys, mnemonics, or sensitive files are ever stored in this repository.

---

## Contents

- `wallet.md` — Wallet and key management
- `validator.json.example` — Validator definition template
- `create-validator.md` — Validator creation transaction
- `operations.md` — Validator operational policies

---

## Network

- Chain ID: `cosmoshub-4`
- Client: `gaiad`
- Validator operated by: **Cumulo**
