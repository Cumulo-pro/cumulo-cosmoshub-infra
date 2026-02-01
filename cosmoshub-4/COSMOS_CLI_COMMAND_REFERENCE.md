# Cosmos Hub Node CLI Command Reference
*(Gaia / gaiad — cosmoshub-4)*

This document is a **practical operator command reference** for running a **Cosmos Hub** node and validator.
It is intentionally concise, structured, and focused on **day-to-day operations**.

All commands assume:
- Chain ID: `cosmoshub-4`
- Binary: `gaiad`
- Default home: `$HOME/.gaia`

---

## ⚙️ systemd — Service Operations

### Reload systemd units
```bash
sudo systemctl daemon-reload
```

### Start service
```bash
sudo systemctl start gaiad
```

### Stop service
```bash
sudo systemctl stop gaiad
```

### Restart service
```bash
sudo systemctl restart gaiad
```

### Enable service at boot
```bash
sudo systemctl enable gaiad
```

### Disable service
```bash
sudo systemctl disable gaiad
```

### Check service status
```bash
sudo systemctl status gaiad
```

---

## 📜 Logs

### Follow logs (raw output)
```bash
sudo journalctl -u gaiad -f -o cat
```

### Show last 200 lines
```bash
sudo journalctl -u gaiad -n 200 --no-hostname -o cat
```

---

## 📡 Node Status & Sync

### Latest block height
```bash
curl -s localhost:26657/status | jq .result.sync_info.latest_block_height
```

### Syncing status
```bash
curl -s localhost:26657/status | jq .result.sync_info.catching_up
```

### Full node status (JSON)
```bash
gaiad status 2>&1 | jq
```

---

## 🌐 Peers

### Number of connected peers
```bash
curl -s localhost:26657/net_info | jq '.result.n_peers'
```

### List connected peers
```bash
curl -s localhost:26657/net_info | jq -r '.result.peers[] | "\(.node_info.id)@\(.remote_ip):26656"'
```

### Your node peer string
```bash
NODE_ID=$(gaiad tendermint show-node-id)
PUBLIC_IP=$(curl -s ifconfig.me)
echo "${NODE_ID}@${PUBLIC_IP}:26656"
```

---

## 🔑 Keys & Wallet Management

### Add new wallet
```bash
gaiad keys add $WALLET
```

### Restore wallet from mnemonic
```bash
gaiad keys add $WALLET --recover
```

### List wallets
```bash
gaiad keys list
```

### Delete wallet
```bash
gaiad keys delete $WALLET
```

### Export wallet key (encrypted)
```bash
gaiad keys export $WALLET
```

### Import wallet key
```bash
gaiad keys import $WALLET wallet.backup
```

### Export EVM private key (⚠️ unsafe)
```bash
gaiad keys unsafe-export-eth-key $WALLET
```

---

## 💰 Balances & Transfers

### Check wallet balance
```bash
gaiad query bank balances $WALLET_ADDRESS
```

### Transfer funds
```bash
gaiad tx bank send $WALLET_ADDRESS <TO_WALLET_ADDRESS> 1000000uatom   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

---

## 🪙 Staking & Rewards

### Withdraw all rewards
```bash
gaiad tx distribution withdraw-all-rewards   --from $WALLET   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom
```

### Withdraw validator rewards + commission
```bash
gaiad tx distribution withdraw-rewards $VALOPER_ADDRESS   --from $WALLET   --commission   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

### Delegate to self
```bash
gaiad tx staking delegate $(gaiad keys show $WALLET --bech val -a) 1000000uatom   --from $WALLET   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

### Delegate to another validator
```bash
gaiad tx staking delegate <TO_VALOPER_ADDRESS> 1000000uatom   --from $WALLET   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

### Redelegate stake
```bash
gaiad tx staking redelegate $VALOPER_ADDRESS <TO_VALOPER_ADDRESS> 1000000uatom   --from $WALLET   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

### Unbond stake
```bash
gaiad tx staking unbond $(gaiad keys show $WALLET --bech val -a) 1000000uatom   --from $WALLET   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

---

## 🧑‍⚖️ Validator Operations

### Create validator
```bash
gaiad tx staking create-validator   --amount 1000000uatom   --from $WALLET   --commission-rate 0.1   --commission-max-rate 0.2   --commission-max-change-rate 0.01   --min-self-delegation 1   --pubkey $(gaiad tendermint show-validator)   --moniker "$MONIKER"   --identity ""   --details "I love blockchain ❤️"   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

### Edit validator metadata / commission
```bash
gaiad tx staking edit-validator   --commission-rate 0.1   --new-moniker "$MONIKER"   --identity ""   --details "I love blockchain ❤️"   --from $WALLET   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

### Validator details
```bash
gaiad query staking validator $(gaiad keys show $WALLET --bech val -a)
```

### Signing info (jailing / missed blocks)
```bash
gaiad query slashing signing-info $(gaiad tendermint show-validator)
```

### Slashing parameters
```bash
gaiad query slashing params
```

### Unjail validator
```bash
gaiad tx slashing unjail   --from $WALLET   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

---

## 🏛 Governance

### List proposals
```bash
gaiad query gov proposals
```

### View proposal
```bash
gaiad query gov proposal <PROPOSAL_ID>
```

### Submit text proposal
```bash
gaiad tx gov submit-proposal   --title ""   --description ""   --deposit 1000000uatom   --type Text   --from $WALLET   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

### Vote on proposal
```bash
gaiad tx gov vote <PROPOSAL_ID> yes   --from $WALLET   --chain-id cosmoshub-4   --gas auto --gas-adjustment 1.5 --fees 3000uatom -y
```

---

## 🧹 Cleanup (Destructive)

### Stop service
```bash
sudo systemctl stop gaiad
```

### Remove systemd service
```bash
sudo rm /etc/systemd/system/gaiad.service
sudo systemctl daemon-reload
```

### Remove node data
```bash
rm -rf $HOME/.gaia
rm -rf $HOME/cosmos
```

---

**End of Cosmos Hub CLI Command Reference**
