# Empirical Study: `x/tokenfactory` CosmWasm Bindings and Admin Lifecycle in Gaia v26

Validator: `cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m`  
Chain: `provider` (Cosmos Hub ICS Testnet)  
Reference: [cosmos/testnets — testnet-tuesdays/demoday25](https://github.com/cosmos/testnets/tree/master/testnet-tuesdays/demoday25)  
6 real on-chain transactions · Operations tested: instantiate, create_denom, mint, change_admin, modify-metadata, burn  
**Updated May 2026** — extended with orphan denom scenario and enterprise compliance investigation  
Date: April–May 2026 · Testnet Tuesday participation event

---

## Table of Contents

1. [Background](#1-background)
2. [The x/tokenfactory Module](#2-the-xtokenfactory-module)
3. [CosmWasm Bindings](#3-cosmwasm-bindings)
4. [Methodology](#4-methodology)
5. [Task Execution](#5-task-execution)
6. [Admin Lifecycle Investigation](#6-admin-lifecycle-investigation)
7. [Orphan Denom — Compound Risk Scenario](#7-orphan-denom--compound-risk-scenario)
8. [Enterprise Compliance — Capability Boundaries](#8-enterprise-compliance--capability-boundaries)
9. [Extended Operations Reference](#9-extended-operations-reference)
10. [Key Findings](#10-key-findings)
11. [Sources & Verification](#11-sources--verification)

---

## 1. Background

Gaia v26 introduced the `x/tokenfactory` module to the Cosmos Hub, enabling any account to create native tokens in a permissionless manner - no governance proposal required. This study documents the full operational lifecycle of a token created through a CosmWasm contract, including the admin control chain, metadata registration, and token supply management.

The token used in this study is **CUM** (Cumulo Validator Community Token), a limited-supply token created to explore the module's capabilities in a real testnet environment.

This document was updated in May 2026 following community feedback on the [Cosmos Hub Forum](https://forum.cosmos.network/t/empirical-study-x-tokenfactory-cosmwasm-bindings-and-admin-lifecycle-in-gaia-v26/16919), which identified two additional scenarios worth investigating empirically: the orphan denom compound risk and the enterprise compliance capability boundaries.

---

## 2. The x/tokenfactory Module

The `x/tokenfactory` module allows any account to create a new token with the name:

```
factory/{creator_address}/{subdenom}
```

Because tokens are namespaced by creator address, this design avoids name collisions and makes creation fully permissionless. A single account can create multiple denoms by providing a unique subdenom for each.

Once a denom is created, the original creator is granted **admin privileges** over the asset. These privileges include:

| Privilege | Description | Available in Gaia v26 |
|-----------|-------------|----------------------|
| **Mint** | Issue new tokens to any address | ✅ |
| **Burn** | Destroy tokens, reducing total supply permanently | ✅ |
| **Burn-from** | Burn tokens held by another address | ❌ Not registered |
| **Force-transfer** | Move tokens between two addresses without their consent | ❌ Not registered |
| **Change admin** | Transfer admin rights to another address | ✅ |
| **Renounce admin** | Set admin to `""`, making the token permanently immutable | ✅ |
| **Modify metadata** | Set name, symbol, description, and decimal precision | ✅ |

> **Note:** `burn-from` and `force-transfer` are available in the `gaiad` CLI but are not registered in the Cosmos Hub message router. Attempting to execute them returns `this capability is not enabled on chain` at the baseapp level (`cosmos-sdk@v0.53.6/baseapp/baseapp.go`). See [Section 8](#8-enterprise-compliance--capability-boundaries) for the full investigation.

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

> ⚠️ **Production footgun:** The contract's `get_denom` query constructs the denom string from the `creator_address` argument provided — it does **not** read from chain state. If you pass your wallet address, you get back a denom that does not exist on-chain. The real denom uses the **contract's address** as creator, not the user wallet's. Testing locally with your wallet address will appear to work correctly, but the denom will be wrong in production. Always verify against `gaiad q tokenfactory denoms-from-admin <contract_address>`.

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

## 7. Orphan Denom — Compound Risk Scenario

This section documents a compound failure mode identified by the community following the original study: the combination of `--no-admin` contract deployment and admin renouncement can produce a denom that is permanently inoperable by anyone.

### 7.1 The compound risk

Two independently documented behaviors combine to create an irreversible dead end:

- **`--no-admin` on contract instantiation** — the contract cannot be upgraded or migrated. Any logic not present at deployment is permanently absent.
- **Admin renouncement** — transferring the denom admin to an empty address (`""`) or a burn address removes all admin control forever.

When both conditions are met simultaneously, the result is an **orphan denom**: a token whose supply is permanently frozen, with no contract able to operate it (immutable) and no wallet able to control it (admin renounced).

> This scenario may be **intentional** for a community token designed around credible supply scarcity. It becomes a **footgun** for any more complex token design that may require future admin operations.

### 7.2 On-chain demonstration

We reproduced the orphan denom scenario on the provider testnet using a new contract and denom specifically created for this purpose.

**Step 1 — Instantiate a new immutable contract**

```bash
gaiad tx wasm instantiate 360 "{}" \
  --label "Orphan Denom Test" \
  --no-admin \
  --from cumwallet \
  --gas-prices 0.005uatom \
  --gas auto \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

**TX:** [B8767EBB6738480A6D5C088F922B7FF6DCADF949A2A66EAEDFF34638BD7E0AA3](https://cumulo.pro/services/cosmos_testnet/search?q=B8767EBB6738480A6D5C088F922B7FF6DCADF949A2A66EAEDFF34638BD7E0AA3)

Contract address: `cosmos14wqld0v6a4a4wr8ucya58melfyutg6s7u6cs0t9l8e59rpfhrz7qzqd35x`

**Step 2 — Create denom ORPHAN**

```bash
gaiad tx wasm execute cosmos14wqld0v6a4a4wr8ucya58melfyutg6s7u6cs0t9l8e59rpfhrz7qzqd35x \
  '{ "create_denom": { "subdenom": "ORPHAN" } }' \
  --from cumwallet \
  --amount 10000000uatom \
  --gas auto \
  --gas-prices 0.005uatom \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

**TX:** [5672001AC091BA0079C75C5555E6F6163F8B3123EB8445F375BAF3763D6006E4](https://cumulo.pro/services/cosmos_testnet/search?q=5672001AC091BA0079C75C5555E6F6163F8B3123EB8445F375BAF3763D6006E4)

Admin confirmed as contract:

```json
{
  "authority_metadata": {
    "admin": "cosmos14wqld0v6a4a4wr8ucya58melfyutg6s7u6cs0t9l8e59rpfhrz7qzqd35x"
  }
}
```

**Step 3 — Attempt to renounce admin directly from the contract**

The demo contract rejects an empty `new_admin_address`:

```
Error: failed to execute message; message index: 0: Generic error: addr_validate errored: 
Input is empty: execute wasm contract failed
```

> This is an additional finding: the CosmWasm demo contract validates that `new_admin_address` is non-empty, blocking direct renouncement through the contract interface. Renouncement requires transferring admin to a wallet first, then executing the renouncement from the wallet.

**Step 4 — Transfer admin to wallet**

```bash
gaiad tx wasm execute cosmos14wqld0v6a4a4wr8ucya58melfyutg6s7u6cs0t9l8e59rpfhrz7qzqd35x \
  '{"change_admin": {
      "denom": "factory/cosmos14wqld0v6a4a4wr8ucya58melfyutg6s7u6cs0t9l8e59rpfhrz7qzqd35x/ORPHAN",
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

**TX:** [4217029DF12C8F28597A37AE881F532DA5BD1F669C8B8A85022AB355A485FCA1](https://cumulo.pro/services/cosmos_testnet/search?q=4217029DF12C8F28597A37AE881F532DA5BD1F669C8B8A85022AB355A485FCA1)

**Step 5 — Renounce admin to burn address (IRREVERSIBLE)**

> `gaiad tx tokenfactory change-admin <denom> ""` is rejected by the CLI with `Invalid address (empty address string is not allowed)`. The standard method for permanent renouncement on the Cosmos Hub is to transfer admin to the canonical burn address.

```bash
gaiad tx tokenfactory change-admin \
  "factory/cosmos14wqld0v6a4a4wr8ucya58melfyutg6s7u6cs0t9l8e59rpfhrz7qzqd35x/ORPHAN" \
  "cosmos1qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqnrql8a" \
  --from cumwallet \
  --gas auto \
  --gas-prices 0.005uatom \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

**TX:** [3C7338C9DAA46C6EFB4618A6CEFC2657721E0E37F9238AB3FF0CFD3EA7B49CAB](https://cumulo.pro/services/cosmos_testnet/search?q=3C7338C9DAA46C6EFB4618A6CEFC2657721E0E37F9238AB3FF0CFD3EA7B49CAB)

Admin confirmed as burn address:

```json
{
  "authority_metadata": {
    "admin": "cosmos1qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqnrql8a"
  }
}
```

**Step 6 — Attempt to operate the orphan denom**

Any attempt to mint, burn, or transfer the ORPHAN denom now fails:

```
Error: failed to execute message; message index: 0: unauthorized account
```

The denom is permanently inoperable. Neither the contract (immutable, `--no-admin`) nor any wallet (admin transferred to burn address) can execute privileged operations on it.

---

## 8. Enterprise Compliance — Capability Boundaries

This section investigates which tokenfactory operations relevant to regulated asset issuance are available in Gaia v26, following the enterprise compliance discussion on the Cosmos Hub Forum.

### 8.1 Context

Community members noted that `force-transfer`, `burn-from`, and admin controls are the compliance primitives required by regulated tokenized asset issuers for use cases such as controlled redemption, regulatory freeze (OFAC-style), and admin-gated burn. This section empirically tests which of these primitives are operational on Hub testnet today.

### 8.2 Module account permissions

Querying the tokenfactory module account on-chain:

```bash
gaiad q auth module-account tokenfactory \
  --node https://cosmos.rpc.testnet.cumulo.me \
  -o json | jq
```

```json
{
  "account": {
    "type": "/cosmos.auth.v1beta1.ModuleAccount",
    "value": {
      "name": "tokenfactory",
      "permissions": ["minter", "burner"]
    }
  }
}
```

The module has `minter` and `burner` permissions but no additional permissions that would enable third-party account operations.

### 8.3 Testing burn-from

Sends a burn operation targeting tokens held by a second address (`cosmos1vt9tcer3wny2m893nuq45tdyzs48gnh5rxfd2g`, which holds 100 CUM after a transfer from the admin wallet):

```bash
gaiad tx tokenfactory burn-from \
  cosmos1vt9tcer3wny2m893nuq45tdyzs48gnh5rxfd2g \
  "100factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM" \
  --from cumwallet \
  --gas auto \
  --gas-prices 0.005uatom \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

Result:

```
Error: this capability is not enabled on chain
[cosmos/cosmos-sdk@v0.53.6/baseapp/baseapp.go:1051]
```

### 8.4 Testing force-transfer

```bash
gaiad tx tokenfactory force-transfer \
  "50factory/cosmos1ujttrfpkslrgcml078hpeh9l6f3ssn3u8m44vyzyaahkxeecpw6s23fxvx/CUM" \
  cosmos1vt9tcer3wny2m893nuq45tdyzs48gnh5rxfd2g \
  cosmos1cdekug2t0rzjjw96yaytw4ryt4u0mzwyaskz3m \
  --from cumwallet \
  --gas auto \
  --gas-prices 0.005uatom \
  --gas-adjustment 1.5 \
  --node https://cosmos.rpc.testnet.cumulo.me \
  --chain-id provider \
  -y
```

Result:

```
Error: this capability is not enabled on chain
[cosmos/cosmos-sdk@v0.53.6/baseapp/baseapp.go:1051]
```

### 8.5 Analysis

The error `this capability is not enabled on chain` originates at `baseapp.go` — the Cosmos SDK message router layer — not at the tokenfactory keeper. This means the messages are not registered in the Hub's message router, regardless of whether the CLI supports them or the module implements them.

| Operation | CLI available | On-chain (Gaia v26) | Error source |
|-----------|--------------|---------------------|--------------|
| `mint` | ✅ | ✅ | — |
| `burn` | ✅ | ✅ | — |
| `change-admin` | ✅ | ✅ | — |
| `modify-metadata` | ✅ | ✅ | — |
| `burn-from` | ✅ | ❌ | `baseapp.go:1051` |
| `force-transfer` | ✅ | ❌ | `baseapp.go:1051` |

The tokenfactory module parameters (queried via `gaiad q tokenfactory params`) expose only `denom_creation_fee` and `denom_creation_gas_consume` — no parameter controls the availability of these messages. Enabling `burn-from` and `force-transfer` on the Cosmos Hub would require a software upgrade that registers these message types in the baseapp router.

### 8.6 Implication for enterprise use cases

The Cosmos Hub's current tokenfactory implementation supports the foundational lifecycle of a token (create, mint, burn, transfer admin, metadata) but does not yet expose the compliance primitives required for regulated asset issuance. The primitives exist in the broader tokenfactory protocol and are available in other Cosmos chains, but were not included in the Gaia v26 integration.

This is an open question for Hub governance: enabling these capabilities would meaningfully expand the Hub's suitability for tokenized real-world assets (RWA) and regulated financial products.

---

## 9. Extended Operations Reference

### 9.1 Via wasm contract execute messages

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

> **Note:** The demo contract does not accept an empty `new_admin_address`. Admin renouncement must be performed from a wallet that holds admin rights, using `gaiad tx tokenfactory change-admin`.

### 9.2 Via gaiad tokenfactory (direct, wallet must be admin)

```bash
# Mint to your own wallet
gaiad tx tokenfactory mint "<amount><denom>" --from <key>

# Mint to any address
gaiad tx tokenfactory mint-to "<amount><denom>" <recipient_address> --from <key>

# Burn tokens from your wallet (permanent)
gaiad tx tokenfactory burn "<amount><denom>" --from <key>

# Burn tokens held by another address — NOT ENABLED in Gaia v26
# gaiad tx tokenfactory burn-from "<amount><denom>" <address> --from <key>

# Move tokens between two addresses without their consent — NOT ENABLED in Gaia v26
# gaiad tx tokenfactory force-transfer "<amount><denom>" <from_addr> <to_addr> --from <key>

# Transfer admin to another wallet
gaiad tx tokenfactory change-admin <denom> <new_admin_address> --from <key>

# Renounce admin permanently — transfer to burn address (IRREVERSIBLE)
# Note: passing "" as new admin is rejected by both the CLI and the demo contract
gaiad tx tokenfactory change-admin <denom> cosmos1qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqnrql8a --from <key>

# Update token name, symbol, description and decimal precision
gaiad tx tokenfactory modify-metadata <denom> <symbol> "<description>" <exponent> --from <key>
```

### 9.3 Useful queries

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

# Query tokenfactory module parameters
gaiad q tokenfactory params

# Query tokenfactory module account permissions
gaiad q auth module-account tokenfactory
```

---

## 10. Key Findings

**1. The contract is the denom admin, not the user wallet.**
When a denom is created through a CosmWasm contract, the contract address becomes the `admin` of the denom in `x/tokenfactory`. Any privileged operation must be routed through the contract unless admin is explicitly transferred to the wallet.

**2. The minimal demo contract does not expose all tokenfactory operations.**
The sample contract (code ID 360) only implements `create_denom`, `mint_tokens`, and `change_admin`. Operations like `burn`, `force-transfer`, or `modify-metadata` are not available as contract messages and must be called directly once admin is transferred.

**3. Admin transfer is a one-way door if renounced.**
Calling `change_admin` with an empty string (`""`) permanently removes all admin control. The token supply becomes fixed forever — no minting, burning, or metadata updates are possible. This can be a desirable property for a community token seeking credible supply scarcity.

**4. `get_denom` is a local computation, not an on-chain lookup.**
> ⚠️ **Production footgun.** The query constructs the denom string from the `creator_address` argument provided, not from chain state. If you pass your wallet address when testing, you get a denom that looks valid but does not exist on-chain — the real denom uses the contract's address as creator. Always verify the actual denom via `gaiad q tokenfactory denoms-from-admin <contract_address>`.

**5. subdenom is case-sensitive.**
`CUM` and `cum` would be registered as two distinct denoms.

**6. `--no-admin` + renouncement = orphan denom (compound risk).**
Deploying a contract with `--no-admin` makes it permanently immutable. If the denom admin is subsequently renounced (transferred to a burn address), the resulting denom has no entity capable of operating it — the contract cannot be upgraded to add the logic, and no wallet holds admin rights. This state is irreversible. Additionally, the demo contract itself rejects empty `new_admin_address` values, so renouncement requires a two-step process: transfer admin to a wallet first, then renounce from the wallet to the burn address `cosmos1qqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqqnrql8a`.

**7. `burn-from` and `force-transfer` are not registered in Gaia v26.**
Both commands exist in the `gaiad` CLI and are implemented in the `cosmos/tokenfactory` module, but the Cosmos Hub's Gaia v26 integration does not register these message types in the baseapp message router. Attempts to execute them return `this capability is not enabled on chain` at `cosmos-sdk@v0.53.6/baseapp/baseapp.go:1051`. These operations are not controllable via governance parameters — enabling them would require a software upgrade.

---

## 11. Sources & Verification

| Resource | Link |
|----------|------|
| Task specification | [cosmos/testnets — demoday25](https://github.com/cosmos/testnets/tree/master/testnet-tuesdays/demoday25) |
| Sample contract source | [cosmos/tokenfactory — wasm-demo](https://github.com/cosmos/tokenfactory/tree/main/wasm-demo/contracts/tokenfactory) |
| CosmWasm token-factory bindings | [CosmWasm/token-factory](https://github.com/CosmWasm/token-factory) |
| Osmosis tokenfactory module docs | [docs.osmosis.zone/tokenfactory](https://docs.osmosis.zone/overview/features/tokenfactory/) |
| Cosmos Developer Documentation | [docs.cosmos.network](https://docs.cosmos.network/) |
| Gaia v26.0.0 upgrade details | [forum.cosmos.network/t/gaia-v26-0-0-software-upgrade](https://forum.cosmos.network/t/gaia-v26-0-0-software-upgrade/16653) |
| Cumulo Explorer | [cumulo.pro/services/cosmos_testnet](https://cumulo.pro/services/cosmos_testnet) |

### On-chain transaction log

**Original study — April 2026**

| Operation | TX Hash | Explorer |
|-----------|---------|---------|
| Instantiate contract (code 360) | `C613BBE9...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=C613BBE90941D92A161994D1B8CB0E8180A2DB662F754A8282551DC4873F9051) |
| Create denom CUM | `819BA821...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=819BA821971DFB913143A335DFA82877EC9FB55EC243371A7E670FFCA84E49F9) |
| Mint 1000 CUM | `75DB7FC4...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=75DB7FC432481C0F3FF619C0CE534139B0871607A9EA4AE84BBE3CAC87E69A6E) |
| Transfer admin: contract → wallet | `D92F7AA5...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=D92F7AA54C02B3401B4027A8DEAD0B6259A0CCF5DC204A34DA50457B29B929DC) |
| Set token metadata | `7CF4C6B2...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=7CF4C6B245C4089954A29BA2CA28B8FBC95F870A71DBEE505FCE7E1E2DC5BC67) |
| Burn 100 CUM | `E080A82B...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=E080A82B56EB0A5BEE5EB282889C5F9BD24174FAFE661C1CF0319B8B8186CFDC) |

**Extended study — May 2026**

| Operation | TX Hash | Explorer |
|-----------|---------|---------|
| Instantiate orphan contract | `B8767EBB...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=B8767EBB6738480A6D5C088F922B7FF6DCADF949A2A66EAEDFF34638BD7E0AA3) |
| Create denom ORPHAN | `5672001A...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=5672001AC091BA0079C75C5555E6F6163F8B3123EB8445F375BAF3763D6006E4) |
| Transfer admin: contract → wallet | `4217029D...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=4217029DF12C8F28597A37AE881F532DA5BD1F669C8B8A85022AB355A485FCA1) |
| Renounce admin to burn address | `3C7338C9...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=3C7338C9DAA46C6EFB4618A6CEFC2657721E0E37F9238AB3FF0CFD3EA7B49CAB) |
| Send 100 CUM to second wallet | `6120CBF5...` | [view](https://cumulo.pro/services/cosmos_testnet/search?q=6120CBF57ED500B6BD219202A5BBE7AD44BDDD917D25C942DC9A6AFCB078B057) |

*Cumulo · Cosmos Hub Validator · Testnet Tuesday April–May 2026*
