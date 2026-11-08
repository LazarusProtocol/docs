# Lazarus Protocol — Introduction

Lazarus Protocol is a decentralized, non-custodial state persistence and proactive Time-To-Live (TTL) warming network for Stellar Soroban smart contracts. It monitors storage entries, contract instances, and compiled WASM bytecodes, extending their ledger TTL before archival eviction occurs.

* **Web Console:** [https://lazarusprotocol.online/](https://lazarusprotocol.online/)
* **Indexer API & Attestation:** [https://api.lazarusprotocol.online/](https://api.lazarusprotocol.online/)
* **GitHub Organization:** [https://github.com/LazarusProtocol](https://github.com/LazarusProtocol)

---

## 1. The Problem: Soroban State Rent & Storage Archival

Unlike traditional blockchains where contract state persists indefinitely without ongoing maintenance, Stellar Soroban implements an active state rent model governed by **CAP-0053**, **CAP-0062**, and **CAP-0066**.

Every ledger entry carries a Time-To-Live counter (`live_until_ledger_seq`) stored in a dedicated `TTLEntry`:

* **`Persistent` (`ContractDataDurabilityPersistent`):** User balances, vault positions, liquidity pools, and orderbooks.
* **`Instance` (`ContractDataDurabilityInstance`):** Contract configuration, admin keys, and reference to the executable WASM hash.
* **`ContractCode` (`LedgerEntryType::CONTRACT_CODE`):** Compiled WebAssembly bytecode deployed on the ledger.
* **`Temporary` (`ContractDataDurabilityTemporary`):** Short-lived authentication signatures and flash data.

### The Consequences of Archival

When `current_ledger > live_until_ledger_seq`, the entry is evicted from active validator RAM into cold storage:

1. **Host Execution Abort:** Any contract call or cross-contract invocation (`env.invoke_contract`) attempting to access an archived entry immediately traps at the host VM level with an unrecoverable `HostError`.
2. **10× Restoration Cost Penalty:** Restoring an archived entry requires locating its cryptographic Merkle inclusion proof and broadcasting `RestoreFootprintOp`. This incurs disk-write base reserves and transaction gas up to **10× the cost** of an unbundled proactive `ExtendFootprintTTLOp`.
3. **Orphaned Contract States:** If a contract's `Instance` or `ContractCode` entry expires while user `Persistent` storage entries are still live, the entire contract becomes uncallable until the instance is restored.

---

## 2. Why Naive Keeper Solutions Fail on Stellar

Building an automated maintenance network on Stellar requires overcoming protocol-level architectural constraints:

### A. The One-Soroban-Operation-Per-Envelope Invariant
Stellar Core strictly mandates that a single transaction envelope may only contain one Soroban operation. `ExtendFootprintTTLOp` (which extends state storage) and `InvokeHostFunctionOp` (which executes smart contract code to disburse rewards) **cannot share a transaction envelope**. An on-chain contract cannot atomically extend arbitrary foreign contract storage footprints and pay a keeper in the same transaction.

### B. The RPC Key Enumeration Blindspot
Soroban JSON-RPC exposes `getLedgerEntries(keys[])`, which requires callers to know the exact 32-byte Base64 XDR `LedgerKey` ahead of time. There is no `getContractKeys(contract_id)` endpoint to enumerate all storage entries belonging to a contract.

### C. Multi-Keeper Race Collisions
If multiple independent keeper daemons observe an expiring entry and broadcast `ExtendFootprintTTLOp` in the same ledger, competing keepers waste network transaction fees without receiving compensation.

---

## 3. The Lazarus Solution

Lazarus resolves these constraints through four interlocking subsystems:

1. **The 3-Phase Decoupled Voucher Pattern:** Separates physical ledger footprint extensions (`ExtendFootprintTTLOp`, Phase 1) from cryptographic verification (`BumpVoucher`, Phase 2) and batched on-chain settlement (`record_bump_batch`, Phase 3).
2. **4-Stage Hybrid Key Discovery Pipeline:** Synthesizes historical Hubble BigQuery exports, RPC ledger catch-up replay, and DeFi storage-tree adapters (e.g. Blend) to comprehensively map all storage keys.
3. **Deterministic Modulo Cohort Sharding:** Partitions monitored keys across keeper nodes using deterministic hashing (`hash(contract_id + ledger_key) % 5`), eliminating race conditions and mempool collisions during normal warming windows.
4. **Non-Custodial Escrow & Dual Circuit Breakers:** Protocol treasuries deposit SAC USDC reserves into `LazarusVault`. Funds are locked against unauthorized withdrawals, rate-limited by global and per-contract hourly outflow caps, and redeemable at any moment via `emergency_withdraw`.

---

## 4. Live Stellar Testnet Deployments

| Resource | Identifier / Address |
| :--- | :--- |
| **Network** | Stellar Testnet (`Test SDF Network ; September 2015`) |
| **Soroban RPC** | `https://soroban-testnet.stellar.org` |
| **LazarusVault Contract ID** | `CCWAH7BVPGYOEBPRFWPZIONBYUCCQH7OARGZLOD6ZV66SOAFBVA2DHD5` |
| **Escrow SAC Token (USDC)** | `CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC` |
| **Oracle / FROST PubKey** | `197f6b23e16c8532c6abc838facd5ea789be0c76b2920334039bfa8b3d368d61` |
| **Web Portal** | `https://lazarusprotocol.online/` |
| **Indexer REST API** | `https://api.lazarusprotocol.online/` |

