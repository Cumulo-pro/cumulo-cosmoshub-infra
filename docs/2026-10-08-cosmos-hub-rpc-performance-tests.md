# Cosmos Hub RPC — Performance & Compliance Test Report

| | |
|---|---|
| **Operator** | Cumulo ([cumulo.pro](https://cumulo.pro)) |
| **Date** | 2026-10-08 — all times in UTC |
| **Purpose** | Evidence for the ICF Ecosystem Growth Delegation RFP *"Cosmos Hub Sponsored Endpoint Provider"* ([forum post](https://forum.cosmos.network/t/targeted-rfp-cosmos-hub-sponsored-endpoint-provider/17414)) |
| **Chain** | `cosmoshub-4` |
| **Public endpoints** | `rpc.cosmos.cumulo.com.es` · `api.cosmos.cumulo.com.es` · `grpc.cosmos.cumulo.com.es` |

This report records every test run on 2026-10-08 against Cumulo's current production Cosmos Hub RPC node, with the raw tool output in the appendices. It states what was measured, what was not, and the known limitations of each test.

---

## Contents

1. [Summary against the RFP requirements](#1-summary-against-the-rfp-requirements)
2. [Test environment](#2-test-environment)
3. [Timeline](#3-timeline)
4. [Interface checks](#4-interface-checks)
5. [External multi-region monitoring](#5-external-multi-region-monitoring)
6. [Retention and indexing](#6-retention-and-indexing)
7. [Load tests — general traffic](#7-load-tests--general-traffic)
8. [Load tests — worst-case searches](#8-load-tests--worst-case-searches)
9. [Upgrade readiness](#9-upgrade-readiness)
10. [Requirements not yet tested](#10-requirements-not-yet-tested)
11. [Conclusions for the service design](#11-conclusions-for-the-service-design)
12. [Limitations](#12-limitations)
- [Appendix A — Raw load-test reports](#appendix-a--raw-load-test-reports)
- [Appendix B — Height-lag log](#appendix-b--height-lag-log)
- [Appendix C — Reproduction scripts](#appendix-c--reproduction-scripts)

---

## 1. Summary against the RFP requirements

| # | RFP requirement | Evidence (this report) | Status |
|---|---|---|---|
| R1 | Public RPC + WebSocket, API and gRPC over TLS, no API key, CORS incl. `OPTIONS`, canonical gRPC codes | All interfaces answer over TLS without authentication; WebSocket upgrade `101`; preflight `204` with `Access-Control-Allow-Origin: *` on RPC and API; gRPC returns status codes (§4) | ✅ Met today |
| R2 | ≥ 2 synced nodes behind a height-aware load balancer, 99.9 % monthly availability | Single node today; 100 % reliability in external monitoring (§5) | ⚙️ Planned |
| R3 | Per-client capacity (10 req/s, burst 50; 5 searches/s; 5 WS subs; 1 tx/s) | Worst-case searches at 8 req/s, above the 5 req/s per-client limit: 100 % success, p99 694 ms (§8.3) | ✅ Capacity verified · limits ⚙️ planned |
| R4 | Shared capacity 100 req/s incl. 20 searches/s; documented limits; `429` + `Retry-After` / `RESOURCE_EXHAUSTED` | 150 req/s mixed traffic, 100 % success, p99 4.4 ms; 40 searches/s, p99 6.7 ms; worst-case searches ≈ 15 req/s per node (§7, §8). No rate limiting configured yet | ✅ Capacity verified (2 nodes) · limits ⚙️ planned |
| R5 | ≥ 15,000 blocks of blocks and tx index on every node | 150,016 blocks retained (§6) | ✅ Met today |
| R6 | `indexer = "kv"`, empty `index-events` | Confirmed in config; all events returned by `tx_search` (§6) | ✅ Met today |
| R7 | ≤ 3 blocks behind head; WS events within 10 s | Max lag of 1 block during all load tests, including worst-case search overload (§7, §8). WS event latency not yet measured | ✅ Lag met · WS latency ⏳ to test |
| R8 | Serve ≤ 300 blocks after upgrade height | Node on current release (Gaia v28.3.1); upgrades applied manually today (§9) | ⚙️ Cosmovisor planned |
| R9 | External monitoring of every interface, on-call alerting, outage notice < 12 h | RPC and REST monitored from 4 regions every 5 min (§5) | ⚙️ gRPC, WS and alerting planned |

Legend: ✅ verified · ⚙️ part of the planned build · ⏳ test defined, not yet run (§10).

All load-test capacity figures are **node-level**: requests were sent from the node's own host to its local RPC listener, without nginx, TLS or network in the path (§12).

---

## 2. Test environment

| Item | Value |
|---|---|
| Node software | Gaia `v28.3.1` (commit `b7f8b81`), Cosmos SDK `v0.53.4`, CometBFT `0.38.22` |
| Node role | Full node, non-validating (`voting_power: 0`), moniker `RPCCumulo` |
| Indexer | `indexer = "kv"`, `index-events = []`, `tx_index: on` |
| Retention at test time | Height `33,162,918` (2026-09-28 12:44) → `33,312,934` (2026-10-08 10:07) = 150,016 blocks |
| CPU | AMD Ryzen 9 9950X — 16 cores / 32 threads, 4.3 GHz base |
| Memory | 192 GB DDR5 |
| Storage | NVMe SSDs in software RAID |
| Reverse proxy | nginx 1.30.5, TLS on all public interfaces |
| Load generator | [vegeta](https://github.com/tsenart/vegeta) v12, run on the same host |

**Shared host.** The machine is a production server that also runs nodes of other networks. Results are therefore conservative compared with the dedicated nodes planned for the sponsored service.

**Baseline (10:11–10:29, before any load):**

| Metric | Value |
|---|---|
| Load average (1 / 5 / 15 min) | 2.13 / 1.92 / 1.60 at 10:11; 2.3–3.0 at 10:28–10:29 |
| CPU | 88.6–91.1 % idle; iowait 0.02–0.20 % |
| RAM | 63 GiB used, 122 GiB available (most of the remainder is page cache) |
| Disk | RAID utilisation 0.9–7.8 % |
| RPC node process | ~11 GB resident memory |
| Height lag vs. independent reference RPC | 0 blocks (10:28:38–10:30:56) |

---

## 3. Timeline

| Time (UTC) | Test | Section |
|---|---|---|
| 10:06 | External multi-region monitoring capture | §5 |
| 10:07 | `/status`, retention and `tx_search` event check | §6 |
| 10:11 | RPC CORS preflight; indexer config; host baseline | §2, §4, §6 |
| 10:21–10:24 | Node process / config review; nginx review; API and gRPC checks; WebSocket direct and via nginx; API CORS preflight | §4 |
| 10:25–10:26 | Version and upgrade-mechanism check | §9 |
| 10:28–10:31 | Height-lag baseline | §2, App. B |
| 10:32–10:42 | **Run 1** — full stepped run (mixed 25/50/100/150 req/s, heavy 10/20/40 req/s) with height-lag watcher | §7.3, App. B |
| ~10:43–10:46 | **Run 2** — mixed 100 and 150 req/s, reports saved | §7.2 |
| ~10:48–10:51 | **Run 3** — heavy 20 and 40 req/s, reports saved | §7.2 |
| ~10:53–10:55 | **Run 4** — worst-case searches, unbounded, 5 req/s | §8.2 |
| 10:56 | Recovery check | §8.2 |
| ~13:41 | Single-query latency of worst-case searches | §8.1 |
| 14:00 | Host baseline check before Run 5 | §8.3 |
| ~14:01–14:10 | **Run 5** — worst-case searches, bounded, stepped run 8/11/14/20 req/s | §8.3 |
| 14:12 | Post-test lag and load check | §8.3 |

---

## 4. Interface checks

### 4.1 Public interfaces

| Check | Request | Result |
|---|---|---|
| RPC status | `GET https://rpc.cosmos.cumulo.com.es/status` | `network: cosmoshub-4`, `catching_up: false`, latest block time equal to query time |
| RPC CORS preflight | `OPTIONS https://rpc.cosmos.cumulo.com.es/status` with `Origin` and `Access-Control-Request-Method: POST` | `HTTP/2 204`; `access-control-allow-origin: *`; `access-control-allow-methods: GET, POST, OPTIONS`; `access-control-allow-headers: Content-Type, Authorization, Accept, Origin, User-Agent, X-Requested-With` |
| WebSocket, direct to node | `GET $LOCAL_RPC/websocket` with `Upgrade: websocket` handshake | `HTTP/1.1 101 Switching Protocols` |
| WebSocket via nginx | `GET https://rpc.cosmos.cumulo.com.es/websocket` with `Upgrade: websocket` handshake | `HTTP/1.1 101 Switching Protocols` |
| REST API | `GET https://api.cosmos.cumulo.com.es/cosmos/base/tendermint/v1beta1/blocks/latest` | `chain_id: cosmoshub-4`, `height: 33313090`, `time: 2026-10-08T10:22:33Z` |
| API CORS preflight | `OPTIONS https://api.cosmos.cumulo.com.es/cosmos/base/tendermint/v1beta1/blocks/latest` with `Origin` and `Access-Control-Request-Method: GET` | `HTTP/2 204`; same CORS headers as RPC |
| gRPC, valid call | `grpcurl grpc.cosmos.cumulo.com.es:443 cosmos.base.tendermint.v1beta1.Service/GetLatestBlock` | `33313090` (current height) |
| gRPC, invalid call | `GetBlockByHeight {"height":"1"}` | `Code: Unknown` — `height 1 is not available, lowest height is 33162918` |
| No API key | All requests above | No authentication required |

**gRPC status codes.** gRPC returns proper status codes rather than HTTP errors. The `Unknown` code for a pruned height is produced by the Cosmos SDK itself, not by the proxy, so any Gaia-based provider returns the same code for this case.

### 4.2 Proxy configuration review

- The RPC virtual host forwards WebSocket upgrades (`proxy_http_version 1.1`, `Upgrade` and `Connection: upgrade` headers).
- CORS headers are added to every response (`always`), and `OPTIONS` is answered with `204` directly by nginx.
- gRPC is proxied with `grpc_pass` behind TLS.
- **No `limit_req` / `limit_conn` rules are configured.** Rate limiting, `429` + `Retry-After` and `RESOURCE_EXHAUSTED` are part of the planned build (§11).

---

## 5. External multi-region monitoring

Cumulo operates [EndPoint Scan](https://cumulo.pro/services/cosmos/rpcscan.php) ([API view](https://cumulo.pro/services/cosmos/apiscan.php)), a public health check of validator-run Cosmos Hub endpoints from four probe locations (US, EU, CA, AS). The endpoint list is maintained publicly on GitHub and any provider can request inclusion.

Dashboard capture, last scan 10:06:44:

| Endpoint | Reliability | CORS | TLS | EU | US | CA | AS | Avg | EMA |
|---|---|---|---|---|---|---|---|---|---|
| `rpc.cosmos.cumulo.com.es` | **100 %** | ✅ | ✅ | 58 ms | 534 ms | 598 ms | 1,359 ms | 661 ms | 683 ms |
| Dashboard average (15 of 18 endpoints healthy) | — | — | — | 117 ms | 484 ms | 520 ms | 739 ms | 460 ms | — |

Probe latency includes network distance to each region; the current node is located in Europe. The planned architecture places the second node in a different location (§11).

---

## 6. Retention and indexing

| Check | Result |
|---|---|
| `/status` earliest block | `33,162,918` — 2026-09-28T12:44:17Z |
| `/status` latest block | `33,312,934` — 2026-10-08T10:07:43Z |
| Blocks retained | **150,016** (10× the required 15,000) |
| `config.toml` | `indexer = "kv"` |
| `app.toml` | `index-events = []` |
| `tx_search?query="tx.height>=33312900"&per_page=1` | `total_count: 41`. The returned transaction (an authz `MsgExec`) carries its full event set: `tx` (`acc_seq`, `signature`), `coin_spent`, `coin_received`, `transfer`, `message`, `withdraw_rewards`, `delegate`, `fee_pay`, `tip_pay`, with `authz_msg_index` / `msg_index` attributes, all with `index: true` |

---

## 7. Load tests — general traffic

### 7.1 Method

- **Targets:** 3,000 requests per file, each block-specific request at a random height within the last 15,000 blocks, to avoid serving everything from cache.
- **Mixed profile** (search share matches the RFP's 20 of 100 req/s):

  | Endpoint | Share |
  |---|---|
  | `/status` | 30 % |
  | `/block?height=h` | 30 % |
  | `/commit?height=h` | 20 % |
  | `/tx_search?query="tx.height=h"&per_page=30` | 10 % |
  | `/block_results?height=h` | 10 % |

- **Heavy profile** (search calls only): 50 % `tx_search` by height, 50 % `block_results`.
- **Execution:** constant rate, 60 s per step, 20 s pause between steps, 10 s client timeout.
- **Lag watcher:** every 5 s, the node's height was compared with an independent public Cosmos Hub RPC, and the host load average was recorded.

### 7.2 Results

**Mixed profile (Run 2)**

| Rate | Requests | Throughput | Success | Min | Mean | p50 | p90 | p95 | p99 | Max |
|---|---|---|---|---|---|---|---|---|---|---|
| 100 req/s | 6,000 | 100.02/s | **100 %** | 0.09 ms | 0.92 ms | 0.66 ms | 3.11 ms | 3.71 ms | 4.53 ms | 12.0 ms |
| 150 req/s | 9,000 | 150.02/s | **100 %** | 0.08 ms | 0.88 ms | 0.63 ms | 2.97 ms | 3.65 ms | 4.36 ms | 12.7 ms |

All responses `200`; mean response size 35.4 KB.

**Heavy profile (Run 3)**

| Rate | Requests | Throughput | Success | Min | Mean | p50 | p90 | p95 | p99 | Max |
|---|---|---|---|---|---|---|---|---|---|---|
| 20 req/s | 1,200 | 20.02/s | **100 %** | 0.13 ms | 2.30 ms | 2.08 ms | 4.22 ms | 4.67 ms | 7.49 ms | 12.1 ms |
| 40 req/s | 2,400 | 40.01/s | **100 %** | 0.11 ms | 2.20 ms | 2.11 ms | 4.10 ms | 4.50 ms | 6.72 ms | 16.6 ms |

All responses `200`; mean response size 57.5 KB.

### 7.3 Height lag during load (Run 1)

Run 1 executed the full stepped sequence (mixed 25 → 150 req/s, then heavy 10 → 40 req/s, i.e. the same rates as Runs 2 and 3 plus lower warm-up steps) with the lag watcher active from 10:32:35 to 10:42:14. Its vegeta reports were not saved, which is why Runs 2 and 3 were repeated for the latency tables.

| Metric | Value |
|---|---|
| Samples | 115 (every 5 s) |
| Lag range | −1 to +1 block (negative: the reference node was one block behind) |
| **Maximum lag** | **1 block** (RFP limit: 3) |
| Peak load average | 8.40 at 10:33:46 |

Full log in Appendix B.

---

## 8. Load tests — worst-case searches

The cost of a `tx_search` grows with the number of index entries that match the query. To measure the worst case, we used a **very active address**: an authz auto-compounding (REStake) bot with thousands of transactions in the retained window.
Query: `message.sender='cosmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd88lyufl'`, `per_page=30`, random page 1–3.

### 8.1 Single-query latency (one request at a time, no other load)

| Query | HTTP | Time |
|---|---|---|
| Very active address, **unbounded** (whole ~150,000-block index) | 200 | 4.589 s |
| Same address, **bounded** to the last 15,000 blocks (`AND tx.height>=H-15000`) | 200 | 0.303 s |
| Low-activity address (`delegate.delegator='cosmos1cg4n…'`) | 200 | 0.015 s |

**Finding:** the size of the retained index drives the worst-case cost. Bounding the search to 15,000 blocks reduces it about 15×. The bounded query may have benefited partly from cache warmed by the unbounded one run just before.

### 8.2 Unbounded worst case (Run 4)

| Rate | Duration | Requests | Success | Latency |
|---|---|---|---|---|
| 5 req/s | 60 s | 300 | **0 %** | all requests hit the 10 s client timeout |

At ~4.6 s per query, 5 req/s accumulates more than 20 concurrent searches, each slowing the others. CometBFT keeps executing a search after the client disconnects: one minute after the run, load average was 6.34 / 5.45 / 3.56 (1 / 5 / 15 min), showing abandoned searches still in progress. **Height lag remained 0**: search overload delayed search responses but did not delay block processing.

### 8.3 Bounded worst case (Run 5)

Run with the host at baseline (load 3.87, RPC node ~20 % CPU at 14:00:41). The target file was generated with the current height. Each step ran for 20 s with a 60 s pause, and the run stopped automatically at the first step below 99 % success.

| Rate | Requests | Throughput | Success | Min | Mean | p50 | p90 | p95 | p99 | Max |
|---|---|---|---|---|---|---|---|---|---|---|
| 8 req/s | 160 | 7.89/s | **100 %** | 231 ms | 376 ms | 352 ms | 522 ms | 567 ms | 694 ms | 741 ms |
| 11 req/s | 220 | 10.90/s | **100 %** | 247 ms | 384 ms | 378 ms | 509 ms | 536 ms | 560 ms | 613 ms |
| 14 req/s | 280 | 13.80/s | **100 %** | 283 ms | 529 ms | 515 ms | 687 ms | 778 ms | 944 ms | 959 ms |
| 20 req/s | 400 | 5.92/s | 41.5 % | 520 ms | 8.50 s | 10 s | 10 s | 10 s | 10 s | 10 s |

At 20 req/s: 166 responses `200`, 234 client timeouts. Mean response size for the successful steps: ~188 KB.
Post-test at 14:12:03: height lag 0, load average 2.90, back at baseline.

**Findings**

- One node sustains about **15 req/s of worst-case searches**. Latency starts to rise at 14 req/s (p50 0.35 s → 0.51 s).
- Above the ceiling, the node degrades abruptly rather than gradually: at 20 req/s it completed only ~6 req/s as requests queued.
- Height lag stayed at 0 before and after the run: search overload delays search responses, not block processing.

---

## 9. Upgrade readiness

| Check | Result |
|---|---|
| Running version | Gaia `v28.3.1`, Cosmos SDK `v0.53.4`, the current release line on `cosmoshub-4` at test time |
| Upgrade mechanism | Manual binary swap at the upgrade height; Cosmovisor not in use |
| Historical upgrade timing | Not available from local logs (journal retention does not cover the last network upgrade); to be reconstructed from external monitoring history |

The sponsored service will use Cosmovisor with the upgrade binary staged in advance on every node (§11), so nodes switch versions at the upgrade height without manual intervention.

---

## 10. Requirements not yet tested

The items below cannot be verified on the current single-node setup, or have not been run yet. Each has a defined test that will be executed once the service is deployed.

| Requirement | Planned test |
|---|---|
| Height-aware load balancer removes nodes > 3 blocks behind | Stop one node's consensus (or delay it) and verify it leaves rotation before it is 4 blocks behind; verify it returns once caught up |
| 99.9 % monthly availability | Monthly availability per interface from multi-region monitoring at 1-minute resolution |
| Client affinity | Repeated requests from one client IP land on the same backend (response header or node ID) |
| Per-client limits: 10 req/s sustained, burst 50 | Single-IP load test at 10 req/s (expect 100 % success) and at 15 req/s (expect `429` + `Retry-After`); burst of 50 |
| Per-client 5 searches/s | Single-IP search load at 5 req/s (100 % success) and 8 req/s (`429`) |
| 5 WebSocket subscriptions per client | Open 5 subscriptions (all accepted) and a 6th (rejected) |
| 1 tx broadcast/s per client | `broadcast_tx_sync` at 1/s (accepted) and 3/s (limited) with invalid tx bytes |
| `RESOURCE_EXHAUSTED` for gRPC | gRPC load from one client above the limit; verify the status code |
| Shared 100 req/s via the public endpoint | Repeat the mixed 100 req/s test through `https://rpc.cosmos.cumulo.com.es` from several external hosts |
| WebSocket events within 10 s | Subscribe to `NewBlock` and `Tx`; measure arrival time minus `block.time` over 24 h |
| Upgrade within 300 blocks | Measure on the next network upgrade from monitoring history |
| External monitoring of gRPC and WebSocket, on-call alerting | Add to EndPoint Scan; trigger a test alert |

---

## 11. Conclusions for the service design

1. **General capacity is not the bottleneck.** One node served 150 req/s of mixed traffic with p99 < 5 ms; the RFP asks for 100 req/s across the whole service.
2. **Typical searches are cheap.** 40 req/s of `tx_search` by height and `block_results` at p99 < 7 ms.
3. **Worst-case searches set the real limit.** This is a property of the CometBFT `kv` indexer and applies to any provider. Planned mitigations:
   - Retention of ~20,000 blocks (above the 15,000 required), pruning the transaction index as well as blocks.
   - Two nodes behind a height-aware load balancer → ~30 req/s of worst-case search capacity against the 20 req/s required.
   - Per-node admission control on search routes (~6 concurrent, ~12/s) returning `429` + `Retry-After` immediately instead of queueing.
   - A proxy read timeout on search routes.
   - The RFP per-client limit of 5 searches/s on top of the above.
4. **Consensus stays healthy under overload.** The node never fell more than 1 block behind during any test, including search overload (§7, §8).
5. **Dedicated nodes in separate locations.** The sponsored service will run on dedicated machines, so its capacity is not shared with other workloads, in two locations to avoid a single point of failure.
6. **Automated upgrades.** Cosmovisor with pre-staged binaries on every node.

---

## 12. Limitations

- **Node-level load tests.** Requests were sent from the node's own host to its local RPC listener, so nginx, TLS and network latency are not included. Public-endpoint tests from external hosts are planned (§10).
- **Load generator on the same host.** vegeta shared CPU with the node; its overhead is small but non-zero.
- **Cache effects.** Each target file of 3,000 requests is replayed in a loop, so after the first pass part of the data comes from the OS page cache. Heights are randomised within the last 15,000 blocks to limit this.
- **Shared host.** Other networks' nodes ran on the same machine during all tests.
- **Excluded run.** One worst-case search run that overlapped an unrelated scheduled job on the host was discarded; the run on a baseline host (Run 5) is the one reported.
- **Lost reports.** The vegeta reports of Run 1 were not saved. Its lag data is reported (§7.3), and its 100/150 and 20/40 req/s steps were repeated as Runs 2 and 3.
- **Single worst-case address.** Other high-activity addresses may produce different figures.
- **Monitoring capture.** The EndPoint Scan figures are a single dashboard capture. At capture time, the per-endpoint checks shown were about one hour old, while the dashboard header showed a 10:06:44 scan. They are not a long-term availability report.

---

## Appendix A — Raw load-test reports

<details>
<summary>Run 2 — mixed profile, 100 and 150 req/s</summary>

```
== mixed 100 req/s ==
Requests      [total, rate, throughput]         6000, 100.02, 100.02
Duration      [total, attack, wait]             59.99s, 59.99s, 276.785µs
Latencies     [min, mean, 50, 90, 95, 99, max]  90.472µs, 915.489µs, 655.441µs, 3.107ms, 3.714ms, 4.528ms, 12.019ms
Bytes In      [total, mean]                     212247498, 35374.58
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:6000
Error Set:
== mixed 150 req/s ==
Requests      [total, rate, throughput]         9000, 150.02, 150.02
Duration      [total, attack, wait]             59.993s, 59.993s, 182.115µs
Latencies     [min, mean, 50, 90, 95, 99, max]  78.85µs, 877.03µs, 628.394µs, 2.973ms, 3.649ms, 4.362ms, 12.745ms
Bytes In      [total, mean]                     318371247, 35374.58
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:9000
Error Set:
```
</details>

<details>
<summary>Run 3 — heavy profile, 20 and 40 req/s</summary>

```
== heavy 20 req/s ==
Requests      [total, rate, throughput]         1200, 20.02, 20.02
Duration      [total, attack, wait]             59.95s, 59.95s, 424.985µs
Latencies     [min, mean, 50, 90, 95, 99, max]  134.625µs, 2.299ms, 2.08ms, 4.224ms, 4.672ms, 7.485ms, 12.056ms
Bytes In      [total, mean]                     69027632, 57523.03
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:1200
Error Set:
== heavy 40 req/s ==
Requests      [total, rate, throughput]         2400, 40.02, 40.01
Duration      [total, attack, wait]             59.979s, 59.976s, 3.577ms
Latencies     [min, mean, 50, 90, 95, 99, max]  107.904µs, 2.199ms, 2.113ms, 4.097ms, 4.498ms, 6.724ms, 16.569ms
Bytes In      [total, mean]                     137418662, 57257.78
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:2400
Error Set:
```
</details>

<details>
<summary>Run 4 — worst-case searches, unbounded, 5 req/s</summary>

```
Requests      [total, rate, throughput]         300, 5.02, 0.00
Duration      [total, attack, wait]             1m10s, 59.8s, 10.001s
Latencies     [min, mean, 50, 90, 95, 99, max]  10s, 10.001s, 10.001s, 10.001s, 10.002s, 10.003s, 10.005s
Bytes In      [total, mean]                     0, 0.00
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           0.00%
Status Codes  [code:count]                      0:300
Error Set:
Get ".../tx_search?query=%22message.sender%3D%27cosmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd88lyufl%27%22&per_page=30&page=N": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
```
</details>

<details>
<summary>§8.1 — single-query latency</summary>

```
# 1. Very active address, unbounded
200
real    0m4.589s
# 2. Same address, bounded to last 15,000 blocks
200
real    0m0.303s
# 3. Low-activity address
200
real    0m0.015s
```
</details>

<details>
<summary>Run 5 — worst-case searches, bounded, stepped run</summary>

```
== costly bounded 8 req/s ==
Requests      [total, rate, throughput]         160, 8.05, 7.89
Duration      [total, attack, wait]             20.273s, 19.876s, 397.483ms
Latencies     [min, mean, 50, 90, 95, 99, max]  230.696ms, 376.498ms, 352.337ms, 522.288ms, 567.217ms, 693.661ms, 740.631ms
Bytes In      [total, mean]                     30181292, 188633.08
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:160
== costly bounded 11 req/s ==
Requests      [total, rate, throughput]         220, 11.05, 10.90
Duration      [total, attack, wait]             20.186s, 19.909s, 276.616ms
Latencies     [min, mean, 50, 90, 95, 99, max]  247.044ms, 383.829ms, 377.773ms, 508.764ms, 535.88ms, 560.489ms, 612.912ms
Bytes In      [total, mean]                     41447852, 188399.33
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:220
== costly bounded 14 req/s ==
Requests      [total, rate, throughput]         280, 14.05, 13.80
Duration      [total, attack, wait]             20.294s, 19.928s, 365.635ms
Latencies     [min, mean, 50, 90, 95, 99, max]  282.684ms, 528.581ms, 514.607ms, 687.472ms, 778.014ms, 944.202ms, 959.409ms
Bytes In      [total, mean]                     52646900, 188024.64
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           100.00%
Status Codes  [code:count]                      200:280
== costly bounded 20 req/s ==
Requests      [total, rate, throughput]         400, 20.05, 5.92
Duration      [total, attack, wait]             28.026s, 19.952s, 8.074s
Latencies     [min, mean, 50, 90, 95, 99, max]  519.971ms, 8.499s, 10s, 10.001s, 10.001s, 10.002s, 10.003s
Bytes In      [total, mean]                     31376232, 78440.58
Bytes Out     [total, mean]                     0, 0.00
Success       [ratio]                           41.50%
Status Codes  [code:count]                      0:234  200:166
>> Fails at 20 req/s, stopping here
```
</details>

---

## Appendix B — Height-lag log

`lag` = reference height − our height; `load` = 1-minute load average on 32 threads.

<details>
<summary>Baseline, 10:28:38–10:30:56 (before load)</summary>

```
10:28:38 lag=0 load=3.05
10:28:43 lag=0 load=2.89
10:28:48 lag=0 load=2.81
10:28:54 lag=0 load=2.67
10:28:59 lag=0 load=2.78
10:29:04 lag=0 load=2.63
10:29:09 lag=0 load=2.50
10:29:14 lag=0 load=2.46
10:29:19 lag=0 load=2.34
10:29:24 lag=0 load=2.88
10:30:15 lag=0 load=4.91
10:30:21 lag=0 load=4.75
10:30:26 lag=0 load=4.69
10:30:31 lag=0 load=4.48
10:30:36 lag=0 load=4.28
10:30:41 lag=0 load=4.02
10:30:46 lag=0 load=3.85
10:30:51 lag=0 load=3.70
10:30:56 lag=0 load=3.44
```
The load rise at 10:30 coincides with the compilation of the load-test tool.
</details>

<details>
<summary>Run 1, 10:32:35–10:42:14 (full stepped load run)</summary>

```
10:32:35 lag=0 load=2.58
10:32:40 lag=0 load=2.53
10:32:45 lag=0 load=2.81
10:32:50 lag=0 load=2.75
10:32:55 lag=0 load=3.41
10:33:01 lag=0 load=3.30
10:33:06 lag=0 load=3.19
10:33:11 lag=0 load=3.18
10:33:16 lag=0 load=3.08
10:33:21 lag=0 load=3.24
10:33:26 lag=0 load=3.14
10:33:31 lag=0 load=5.69
10:33:36 lag=0 load=5.39
10:33:41 lag=0 load=5.68
10:33:46 lag=0 load=8.40
10:33:51 lag=0 load=7.89
10:33:56 lag=0 load=7.41
10:34:01 lag=0 load=6.98
10:34:06 lag=0 load=6.50
10:34:12 lag=0 load=6.06
10:34:17 lag=0 load=5.98
10:34:22 lag=0 load=5.74
10:34:27 lag=0 load=5.44
10:34:32 lag=0 load=5.08
10:34:37 lag=0 load=4.76
10:34:42 lag=0 load=4.45
10:34:47 lag=0 load=4.58
10:34:52 lag=-1 load=4.29
10:34:57 lag=0 load=4.03
10:35:02 lag=0 load=4.19
10:35:07 lag=0 load=4.65
10:35:13 lag=0 load=4.36
10:35:18 lag=0 load=4.17
10:35:23 lag=0 load=4.32
10:35:28 lag=0 load=4.13
10:35:33 lag=0 load=3.96
10:35:38 lag=0 load=3.80
10:35:43 lag=0 load=3.66
10:35:48 lag=0 load=3.37
10:35:53 lag=0 load=3.26
10:35:58 lag=0 load=3.48
10:36:03 lag=0 load=3.28
10:36:08 lag=-1 load=3.82
10:36:13 lag=0 load=3.83
10:36:19 lag=0 load=3.68
10:36:24 lag=0 load=3.47
10:36:29 lag=0 load=3.51
10:36:34 lag=0 load=3.31
10:36:39 lag=0 load=3.21
10:36:44 lag=0 load=3.03
10:36:49 lag=0 load=3.03
10:36:54 lag=0 load=4.47
10:36:59 lag=0 load=4.27
10:37:04 lag=0 load=4.01
10:37:09 lag=0 load=3.77
10:37:14 lag=0 load=3.62
10:37:19 lag=0 load=3.49
10:37:25 lag=0 load=3.45
10:37:30 lag=0 load=3.34
10:37:35 lag=0 load=3.15
10:37:40 lag=0 load=2.98
10:37:45 lag=0 load=2.82
10:37:50 lag=0 load=2.67
10:37:55 lag=0 load=2.54
10:38:00 lag=0 load=2.74
10:38:05 lag=0 load=2.68
10:38:10 lag=0 load=2.54
10:38:15 lag=0 load=2.42
10:38:20 lag=0 load=2.30
10:38:25 lag=0 load=2.20
10:38:31 lag=0 load=2.10
10:38:36 lag=0 load=1.93
10:38:41 lag=0 load=1.78
10:38:46 lag=0 load=1.72
10:38:51 lag=0 load=1.66
10:38:56 lag=0 load=1.61
10:39:01 lag=0 load=1.48
10:39:06 lag=0 load=1.52
10:39:11 lag=0 load=1.40
10:39:16 lag=0 load=1.92
10:39:21 lag=0 load=1.85
10:39:26 lag=0 load=2.42
10:39:31 lag=0 load=2.22
10:39:36 lag=0 load=2.05
10:39:42 lag=0 load=2.12
10:39:47 lag=0 load=2.11
10:39:52 lag=1 load=2.02
10:39:57 lag=0 load=1.94
10:40:02 lag=0 load=1.87
10:40:07 lag=0 load=1.80
10:40:12 lag=0 load=1.89
10:40:17 lag=0 load=1.82
10:40:22 lag=0 load=1.68
10:40:27 lag=0 load=1.54
10:40:32 lag=0 load=1.66
10:40:37 lag=-1 load=1.53
10:40:42 lag=0 load=1.40
10:40:48 lag=0 load=1.37
10:40:53 lag=0 load=1.66
10:40:58 lag=0 load=1.53
10:41:03 lag=0 load=1.41
10:41:08 lag=0 load=1.29
10:41:13 lag=0 load=1.19
10:41:18 lag=0 load=1.09
10:41:23 lag=0 load=1.01
10:41:28 lag=0 load=1.17
10:41:33 lag=0 load=1.07
10:41:38 lag=0 load=1.15
10:41:43 lag=0 load=1.05
10:41:48 lag=0 load=0.97
10:41:53 lag=0 load=1.45
10:41:59 lag=0 load=1.34
10:42:04 lag=0 load=1.23
10:42:09 lag=0 load=1.13
10:42:14 lag=0 load=1.12
```
</details>

<details>
<summary>Spot checks</summary>

```
10:56:16  lag=0  load 6.34 / 5.45 / 3.56   (1 min after Run 4)
14:00:41  —      load 3.87, RPC node 20 % CPU (baseline before Run 5)
14:12:03  lag=0  load 2.90                 (after Run 5)
```
</details>

---

## Appendix C — Reproduction scripts

**Install and variables**

```bash
go install github.com/tsenart/vegeta/v12@latest
mkdir -p ~/loadtest && cd ~/loadtest
LOCAL_RPC=http://127.0.0.1:<rpc-port>       # the node's local RPC listener
REF_RPC=https://<independent-cosmoshub-rpc> # any independent public RPC used as height reference
```

**Interface checks**

```bash
# CORS preflight (RPC and API)
curl -si -X OPTIONS https://rpc.cosmos.cumulo.com.es/status \
  -H "Origin: https://example.com" -H "Access-Control-Request-Method: POST"
curl -si -X OPTIONS https://api.cosmos.cumulo.com.es/cosmos/base/tendermint/v1beta1/blocks/latest \
  -H "Origin: https://example.com" -H "Access-Control-Request-Method: GET"

# WebSocket handshake (expect 101)
curl -si --http1.1 -N --max-time 5 https://rpc.cosmos.cumulo.com.es/websocket \
  -H "Connection: Upgrade" -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ=="

# REST and gRPC
curl -s https://api.cosmos.cumulo.com.es/cosmos/base/tendermint/v1beta1/blocks/latest | jq '.block.header | {chain_id, height, time}'
grpcurl grpc.cosmos.cumulo.com.es:443 cosmos.base.tendermint.v1beta1.Service/GetLatestBlock | jq '.block.header.height'
grpcurl -d '{"height":"1"}' grpc.cosmos.cumulo.com.es:443 cosmos.base.tendermint.v1beta1.Service/GetBlockByHeight
```

**Target generation (mixed / heavy)**

```bash
H=$(curl -s $LOCAL_RPC/status | jq -r .result.sync_info.latest_block_height)
BASE=$LOCAL_RPC
gen() {
  : > "$1"
  for i in $(seq 1 3000); do
    h=$((H - RANDOM % 15000)); r=$((RANDOM % 10))
    [ "$2" = heavy ] && r=$((8 + RANDOM % 2))
    case $r in
      0|1|2) echo "GET $BASE/status" ;;
      3|4|5) echo "GET $BASE/block?height=$h" ;;
      6|7)   echo "GET $BASE/commit?height=$h" ;;
      8)     echo "GET $BASE/tx_search?query=%22tx.height%3D$h%22&per_page=30" ;;
      9)     echo "GET $BASE/block_results?height=$h" ;;
    esac
    echo
  done >> "$1"
}
gen mixed.txt mixed; gen heavy.txt heavy
```

**Height-lag watcher**

```bash
while true; do
  L=$(curl -s $LOCAL_RPC/status | jq -r .result.sync_info.latest_block_height)
  R=$(curl -s $REF_RPC/status | jq -r .result.sync_info.latest_block_height)
  echo "$(date +%T) lag=$((R-L)) load=$(cut -d' ' -f1 /proc/loadavg)"
  sleep 5
done > lag.log
```

**Runs 1–3 (general traffic)**

```bash
for rate in 25 50 100 150; do
  vegeta attack -targets=mixed.txt -rate=$rate -duration=60s -timeout=10s | vegeta report
  sleep 20
done
for rate in 10 20 40; do
  vegeta attack -targets=heavy.txt -rate=$rate -duration=60s -timeout=10s | vegeta report
  sleep 20
done
```

**§8.1 and Run 4 (worst-case searches, unbounded)**

```bash
ADDR=cosmos1jv65s3grqf6v6jl3dp4t6c9t9rk99cd88lyufl
time curl -s -o /dev/null -w "%{http_code}\n" --max-time 180 \
  "$LOCAL_RPC/tx_search?query=%22message.sender%3D%27$ADDR%27%22&per_page=30"
time curl -s -o /dev/null -w "%{http_code}\n" --max-time 180 \
  "$LOCAL_RPC/tx_search?query=%22message.sender%3D%27$ADDR%27%20AND%20tx.height%3E%3D$((H-15000))%22&per_page=30"

for i in $(seq 1 500); do
  echo "GET $LOCAL_RPC/tx_search?query=%22message.sender%3D%27$ADDR%27%22&per_page=30&page=$((1 + RANDOM % 3))"
  echo
done > costly.txt
vegeta attack -targets=costly.txt -rate=5 -duration=60s -timeout=10s | vegeta report
```

**Run 5 (worst-case searches, bounded, stepped with auto-stop)**

```bash
H=$(curl -s $LOCAL_RPC/status | jq -r .result.sync_info.latest_block_height)
for i in $(seq 1 500); do
  echo "GET $LOCAL_RPC/tx_search?query=%22message.sender%3D%27$ADDR%27%20AND%20tx.height%3E%3D$((H-15000))%22&per_page=30&page=$((1 + RANDOM % 3))"
  echo
done > costly_bounded.txt

for rate in 8 11 14 20; do
  vegeta attack -targets=costly_bounded.txt -rate=$rate -duration=20s -timeout=10s > r_$rate.bin
  vegeta report r_$rate.bin
  S=$(vegeta report -type=json r_$rate.bin | jq .success)
  awk "BEGIN{exit !($S < 0.99)}" && break
  sleep 60
done
```
