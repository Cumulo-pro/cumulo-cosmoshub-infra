# Wallet & Key Management

This document describes how wallets and keys are handled for the Cosmos Hub validator.

---

## Wallet Creation

A dedicated wallet is used exclusively for validator operations.

```bash
WALLET="validator-wallet"
gaiad keys add "$WALLET"
```

> Store the mnemonic **offline only** (hardware wallet, encrypted vault, or secure cold storage).

---

## Key Storage Principles

- Mnemonics are never stored on disk in plaintext
- No keys are committed to Git repositories
- Validator keys are isolated from monitoring or automation scripts
- Access to signing keys is restricted to authorized operators

---

## Backup Policy

Critical files to back up securely:
- Wallet mnemonic (offline)
- `$HOME/.gaia/config/priv_validator_state.json`
- `$HOME/.gaia/config/priv_validator_key.json` (if applicable)

Backups are encrypted and stored offline.

---

## Key Rotation

If key compromise is suspected:
- Validator is immediately jailed (if required)
- Keys are rotated following Cosmos Hub procedures
- Incident is documented internally
