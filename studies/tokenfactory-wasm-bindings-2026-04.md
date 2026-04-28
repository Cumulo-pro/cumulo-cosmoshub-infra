# Empirical Study: `x/tokenfactory` CosmWasm Bindings and Admin Lifecycle in Gaia v26

Validator: `cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m`  
Chain: `provider` (Cosmos Hub ICS Testnet)  
Reference: [cosmos/testnets — testnet-tuesdays/demoday25](https://github.com/cosmos/testnets/tree/master/testnet-tuesdays/demoday25)  
6 real on-chain transactions · Operations tested: instantiate, create_denom, mint, change_admin, modify-metadata, burn
Date: April 2026 · Testnet Tuesday participation event  

---

## Table of Contents

1. [Background](#1-background)
2. [The x/tokenfactory Module](#2-the-xtokenfactory-module)
3. [CosmWasm Bindings](#3-cosmwasm-bindings)
4. [Methodology](#4-methodology)
5. [Task Execution](#5-task-execution)
6. [Admin Lifecycle Investigation](#6-admin-lifecycle-investigation)
7. [Extended Operations Reference](#7-extended-operations-reference)
8. [Key Findings](#8-key-findings)
9. [Sources & Verification](#9-sources--verification)

---

## 1. Background

Gaia v26 introduced the `x/tokenfactory` module to the Cosmos Hub, enabling any account to create native tokens in a permissionless manner - no governance proposal required. This study documents the full operational lifecycle of a token created through a CosmWasm contract, including the admin control chain, metadata registration, and token supply management.

The token used in this study is **CUM** (Cumulo Validator Community Token), a limited-supply token created to explore the module's capabilities in a real testnet environment. 

---

## 2. The x/tokenfactory Module

The `x/tokenfactory` module allows any account to create a new token with the name:

```
factory/{creator_address}/{subdenom}
```

Because tokens are namespaced by creator address, this design avoids name collisions and makes creation fully permissionless. A single account can create multiple denoms by providing a unique subdenom for each.

Once a denom is created, the original creator is granted **admin privileges** over the asset. These privileges include:

| Privilege | Description |
|-----------|-------------|
| **Mint** | Issue new tokens to any address |
| **Burn** | Destroy tokens, reducing total supply permanently |
| **Burn-from** | Burn tokens held by another address |
| **Force-transfer** | Move tokens between two addresses without their consent |
| **Change admin** | Transfer admin rights to another address |
| **Renounce admin** | Set admin to `""`, making the token permanently immutable |
| **Modify metadata** | Set name, symbol, description, and decimal precision |

Admin privileges can also be shared with other accounts using the `x/authz` module.

---

## 3. CosmWasm Bindings

The `x/tokenfactory` module exposes bindings to CosmWasm, allowing smart contracts to interact with the module on behalf of their users. This enables more complex token management logic to be encoded in contract code rather than executed directly by end users.

In this study, we use a minimal sample contract (code ID `360`) uploaded to the provider testnet. The contract exposes the following interface:

**Execute messages:**
- `create_denom` — registers a new subdenom via the tokenfactory module
- `mint_tokens` — mints tokens to a specified address
- `change_admin` — transfers the denom admin to a new address

**Query messages:**
- `get_denom` — returns the computed denom for a given creator and subdenom

When a denom is created through the contract, the **contract itself becomes the admin** of the denom — not the user's wallet. This is a critical distinction explored in detail in Section 6.

---

## 4. Methodology

All transactions were signed by the validator's self-delegation address:
`cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m`

Infrastructure used:
- **Binary:** `gaiad` (Gaia v26)
- **RPC:** `https://cosmos.rpc.testnet.cumulo.me` (own validator node)
- **Chain ID:** `provider`
- **Explorer:** [cumulo.pro/services/cosmos_testnet](https://cumulo.pro/services/cosmos_testnet)

> **Note:** The local node had transaction indexing disabled (`transaction indexing is disabled` error). All tx queries were routed through the validator's own RPC endpoint.

---

## 5. Task Execution

### 5.1 Instantiate the contract

The code `360` is a template already stored on the testnet. Instantiating it creates a personal copy of the contract, owned by the caller's wallet.

```bash
gaiad tx wasm instantiate 360 "{}" \
  --label "CUM Token Factory" \
  --no-admin \
  --from cumwallet \
  --gas-prices 0.005uatom \
  --gas auto \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

> `--no-admin` prevents anyone from migrating or upgrading the contract after deployment. `"{}"` is the empty instantiation message — this contract requires no initialization parameters.

**TX:** [C613BBE90941D92A161994D1B8CB0E8180A2DB662F754A8282551DC4873F9051](https://cumulo.pro/services/cosmos_testnet/search?q=C613BBE90941D92A161994D1B8CB0E8180A2DB662F754A8282551DC4873F9051)

Retrieving the contract address (via own RPC due to disabled local indexer):

```bash
gaiad q tx C613BBE90941D92A161994D1B8CB0E8180A2DB662F754A8282551DC4873F9051 \
  -o json \
  --node https://cosmos.rpc.testnet.cumulo.me | \
  jq -r '.events[] | select(.type=="wasm").attributes[] | select(.key=="_contract_address").value'
```

```
cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx
```

---

### 5.2 Create denom

The contract calls the tokenfactory module on behalf of the user and registers `CUM` as a subdenom. A **10 ATOM fee** (`10000000uatom`) is required by the module for denom creation.

```bash
gaiad tx wasm execute cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx \
  '{ "create_denom": { "subdenom": "CUM" } }' \
  --from cumwallet \
  --amount 10000000uatom \
  --gas auto \
  --gas-prices 0.005uatom \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

**TX:** [819BA821971DFB913143A335DFA82877EC9FB55EC243371A7E670FFCA84E49F9](https://cumulo.pro/services/cosmos_testnet/search?q=819BA821971DFB913143A335DFA82877EC9FB55EC243371A7E670FFCA84E49F9)

Verification:

```bash
gaiad q tokenfactory denoms-from-admin \
  cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx \
  --node https://cosmos.rpc.testnet.cumulo.me \
  -o json | jq
```

```
factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM
```

---

### 5.3 Mint tokens

1000 units minted directly to the validator's self-delegation address.

```bash
gaiad tx wasm execute cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx \
  '{ "mint_tokens": {
      "amount": "1000",
      "denom": "factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM",
      "mint_to_address": "cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m"
    }}' \
  --from cumwallet \
  --gas auto \
  --gas-prices 0.005uatom \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

**TX:** [75DB7FC432481C0F3FF619C0CE534139B0871607A9EA4AE84BBE3CAC87E69A6E](https://cumulo.pro/services/cosmos_testnet/search?q=75DB7FC432481C0F3FF619C0CE534139B0871607A9EA4AE84BBE3CAC87E69A6E)

Balance verification:

```bash
gaiad q bank balance cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m \
  "factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM" \
  --node https://cosmos.rpc.testnet.cumulo.me \
  -o json | jq
```

```json
{
  "balance": {
    "denom": "factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM",
    "amount": "1000"
  }
}
```

---

## 6. Admin Lifecycle Investigation

### 6.1 Identifying the real admin

After completing the three tasks, we investigated the actual on-chain admin of the denom:

```bash
gaiad q tokenfactory denom-authority-metadata \
  "factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM" \
  --node https://cosmos.rpc.testnet.cumulo.me \
  -o json | jq
```

```json
{
  "authority_metadata": {
    "admin": "cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx"
  }
}
```

The admin is the **contract**, not the user's wallet. The control hierarchy is:

```
User wallet (cumwallet)
    └── owner of the wasm contract
            └── admin of the CUM denom in x/tokenfactory
```

This means any privileged tokenfactory operation (mint, burn, modify-metadata) must go through the contract, not be called directly by the wallet.

### 6.2 Querying contract state

We explored the contract's internal state to understand what information it stores:

```bash
gaiad q wasm contract-state smart \
  cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx \
  '{"get_denom": {
      "creator_address": "cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m",
      "subdenom": "CUM"
    }}' \
  --node https://cosmos.rpc.testnet.cumulo.me \
  -o json | jq
```

```json
{
  "data": {
    "denom": "factory/cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m/CUM"
  }
}
```

> **Note:** The contract's `get_denom` query returns the *computed* denom based on the creator address provided — it constructs the denom string locally. The *actual* on-chain denom uses the contract's address as creator, not the user wallet's. These are two different things.

### 6.3 Transferring admin to the wallet — change_admin

To operate directly on the denom without routing through the contract, we transferred the admin from the contract to our wallet. The contract implements `change_admin` as an execute message:

```bash
gaiad tx wasm execute cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx \
  '{"change_admin": {
      "denom": "factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM",
      "new_admin_address": "cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m"
    }}' \
  --from cumwallet \
  --gas auto \
  --gas-prices 0.005uatom \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

**TX:** [D92F7AA54C02B3401B4027A8DEAD0B6259A0CCF5DC204A34DA50457B29B929DC](https://cumulo.pro/services/cosmos_testnet/search?q=D92F7AA54C02B3401B4027A8DEAD0B6259A0CCF5DC204A34DA50457B29B929DC)

Admin after the transfer:

```json
{
  "authority_metadata": {
    "admin": "cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m"
  }
}
```

### 6.4 Registering token metadata - modify-metadata

With the wallet now as direct admin, we registered the token's metadata so it appears correctly in explorers and wallets:

```bash
gaiad tx tokenfactory modify-metadata \
  "factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM" \
  "CUM" \
  "CUM is the community token of Cumulo validator. Limited supply of 1000 units used to reward and engage delegators on the Cosmos Hub testnet." \
  6 \
  --from cumwallet \
  --gas auto \
  --gas-prices 0.005uatom \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

> The `6` exponent defines that 1 CUM = 1,000,000 base units — the same precision as ATOM.

**TX:** [7CF4C6B245C4089954A29BA2CA28B8FBC95F870A71DBEE505FCE7E1E2DC5BC67](https://cumulo.pro/services/cosmos_testnet/search?q=7CF4C6B245C4089954A29BA2CA28B8FBC95F870A71DBEE505FCE7E1E2DC5BC67)

On-chain metadata result:

```json
{
  "metadata": {
    "description": "CUM is the community token of Cumulo validator. Limited supply of 1000 units used to reward and engage delegators on the Cosmos Hub testnet.",
    "denom_units": [
      {
        "denom": "factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM",
        "aliases": ["CUM"]
      },
      {
        "denom": "CUM",
        "exponent": 6,
        "aliases": ["factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM"]
      }
    ],
    "base": "factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM",
    "display": "CUM",
    "symbol": "CUM"
  }
}
```

### 6.5 Burning tokens — deflationary supply management

We burned 100 units to demonstrate permanent supply reduction:

```bash
gaiad tx tokenfactory burn \
  "100factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM" \
  --from cumwallet \
  --gas auto \
  --gas-prices 0.005uatom \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

**TX:** [E080A82B56EB0A5BEE5EB282889C5F9BD24174FAFE661C1CF0319B8B8186CFDC](https://cumulo.pro/services/cosmos_testnet/search?q=E080A82B56EB0A5BEE5EB282889C5F9BD24174FAFE661C1CF0319B8B8186CFDC)

| Before burn | After burn |
|-------------|------------|
| 1000 CUM | 900 CUM |

---

## 7. Extended Operations Reference

### 7.1 Via wasm contract execute messages

These operations are available through the demo contract (code ID 360):

```bash
# Create a new denom (10 ATOM fee required)
gaiad tx wasm execute <contract> \
  '{"create_denom": {"subdenom": "<name>"}}' \
  --amount 10000000uatom --from <key> ...

# Mint tokens to any address
gaiad tx wasm execute <contract> \
  '{"mint_tokens": {"amount": "<n>", "denom": "<denom>", "mint_to_address": "<addr>"}}' \
  --from <key> ...

# Transfer denom admin to another address
gaiad tx wasm execute <contract> \
  '{"change_admin": {"denom": "<denom>", "new_admin_address": "<addr>"}}' \
  --from <key> ...
```

### 7.2 Via gaiad tokenfactory (direct, wallet must be admin)

```bash
# Mint to your own wallet
gaiad tx tokenfactory mint "<amount><denom>" --from <key>

# Mint to any address
gaiad tx tokenfactory mint-to "<amount><denom>" <recipient_address> --from <key>

# Burn tokens from your wallet (permanent)
gaiad tx tokenfactory burn "<amount><denom>" --from <key>

# Burn tokens held by another address
gaiad tx tokenfactory burn-from "<amount><denom>" <address> --from <key>

# Move tokens between two addresses without their consent (admin only)
gaiad tx tokenfactory force-transfer "<amount><denom>" <from_addr> <to_addr> --from <key>

# Transfer admin to another wallet
gaiad tx tokenfactory change-admin <denom> <new_admin_address> --from <key>

# Renounce admin permanently — token becomes immutable (IRREVERSIBLE)
gaiad tx tokenfactory change-admin <denom> "" --from <key>

# Update token name, symbol, description and decimal precision
gaiad tx tokenfactory modify-metadata <denom> <symbol> "<description>" <exponent> --from <key>
```

### 7.3 Useful queries

```bash
# List all denoms administered by an address or contract
gaiad q tokenfactory denoms-from-admin <address>

# Check who is the admin of a denom
gaiad q tokenfactory denom-authority-metadata <denom>

# Read token metadata registered in the bank module
gaiad q bank denom-metadata <denom>

# Check token balance in a wallet
gaiad q bank balance <address> <denom>

# List all contracts instantiated by an address
gaiad q wasm list-contracts-by-creator <address>

# Read contract info (code_id, creator, admin, label)
gaiad q wasm contract <contract_address>

# Query internal contract state
gaiad q wasm contract-state smart <contract_address> '<query_json>'
```

---

## 8. Key Findings

**1. The contract is the denom admin, not the user wallet.**
When a denom is created through a CosmWasm contract, the contract address becomes the `admin` of the denom in `x/tokenfactory`. Any privileged operation must be routed through the contract unless admin is explicitly transferred to the wallet.

**2. The minimal demo contract does not expose all tokenfactory operations.**
The sample contract (code ID 360) only implements `create_denom`, `mint_tokens`, and `change_admin`. Operations like `burn`, `force-transfer`, or `modify-metadata` are not available as contract messages and must be called directly once admin is transferred.

**3. Admin transfer is a one-way door if renounced.**
Calling `change_admin` with an empty string (`""`) permanently removes all admin control. The token supply becomes fixed forever — no minting, burning, or metadata updates are possible. This can be a desirable property for a community token seeking credible supply scarcity.

**4. The `get_denom` contract query is a local computation, not an on-chain lookup.**
The query constructs the denom string from the inputs provided rather than reading from chain state. It uses the `creator_address` argument, not the contract's own address, which can be misleading.

**5. subdenom is case-sensitive.**
`CUM` and `cum` would be registered as two distinct denoms.

---

## 9. Sources & Verification

| Resource | Link |
|----------|------|
| Task specification | [cosmos/testnets — demoday25](https://github.com/cosmos/testnets/tree/master/testnet-tuesdays/demoday25) |
| Sample contract source | [cosmos/tokenfactory — wasm-demo](https://github.com/cosmos/tokenfactory/tree/main/wasm-demo/contracts/tokenfactory) |
| CosmWasm token-factory bindings | [CosmWasm/token-factory](https://github.com/CosmWasm/token-factory) |
| Osmosis tokenfactory module docs | [docs.osmosis.zone/tokenfactory](https://docs.osmosis.zone/overview/features/tokenfactory/) |
| Cosmos Developer Documentation | [docs.cosmos.network](https://docs.cosmos.network/) |
| Cumulo Explorer | [cumulo.pro/services/cosmos_testnet](https://cumulo.pro/services/cosmos_testnet) |

### On-chain transaction log

| Operation | TX Hash | Explorer |
|-----------|---------|---------|
| Instantiate contract (code 360) | `C613BBE9...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=C613BBE90941D92A161994D1B8CB0E8180A2DB662F754A8282551DC4873F9051) |
| Create denom CUM | `819BA821...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=819BA821971DFB913143A335DFA82877EC9FB55EC243371A7E670FFCA84E49F9) |
| Mint 1000 CUM | `75DB7FC4...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=75DB7FC432481C0F3FF619C0CE534139B0871607A9EA4AE84BBE3CAC87E69A6E) |
| Transfer admin: contract → wallet | `D92F7AA5...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=D92F7AA54C02B3401B4027A8DEAD0B6259A0CCF5DC204A34DA50457B29B929DC) |
| Set token metadata | `7CF4C6B2...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=7CF4C6B245C4089954A29BA2CA28B8FBC95F870A71DBEE505FCE7E1E2DC5BC67) |
| Burn 100 CUM | `E080A82B...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=E080A82B56EB0A5BEE5EB282889C5F9BD24174FAFE661C1CF0319B8B8186CFDC) |
