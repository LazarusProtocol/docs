# Architecture & Protocol Mechanics

Lazarus Protocol coordinates on-chain escrow custody, off-chain storage key indexing, cryptographic attestation signing, and autonomous keeper execution across four distinct components.

* **Smart Contracts:** [`contracts/`](../contracts) — Non-custodial SAC escrow and voucher settlement on Soroban.
* **Indexer:** [`indexer/`](../indexer) — 4-stage hybrid key discovery pipeline and REST API.
* **Attestation Service:** [`attestation/`](../attestation) — Enclave and 2-of-3 FROST MPC signature engine (`attestation`).
* **Keeper Node:** [`keeper/`](../keeper) — Autonomous polling, footprint packing, and batch redemption loop (`keeper`).
* **Web Dashboard:** [`web/`](../web) — Developer diagnostics, visual TTL heatmap, vault escrow portal, and keeper operator dashboard.

---

## 1. System Overview

1. **Protocol Enrollment:** A protocol connects their wallet on the web console ([https://lazarusprotocol.online/](https://lazarusprotocol.online/)) and deposits SAC USDC into `LazarusVault` on Stellar.
2. **Key Discovery:** The indexer automatically maps the contract's persistent storage keys, instance keys, and compiled WASM bytecodes via Hubble BigQuery and RPC catch-up replay.
3. **Phase 1 (Extend):** Autonomous keeper nodes poll expiring keys, pack up to 100 entries into an `ExtendFootprintTTLOp` transaction, and submit it to Stellar.
4. **Phase 2 (Attest):** The keeper submits the transaction hash to the Attestation Engine. The engine verifies RPC consensus finality and signs a domain-separated `BumpVoucher` (`LAZARUS_VOUCHER_V1`).
5. **Phase 3 (Settle):** The keeper buffers up to 10 vouchers and redeems them on-chain via `record_bump_batch`, claiming USDC bounties.

---

## 2. Core Protocol Challenges & Solutions

### Challenge 1: The One-Soroban-Operation Invariant
In Stellar Core, a transaction envelope containing Soroban operations may only contain **one** operation. `ExtendFootprintTTLOp` (which extends ledger TTL) and `InvokeHostFunctionOp` (which executes contract logic to disburse tokens) cannot share an envelope.

**Solution:**
Lazarus decouples state warming from payment:
* Keepers broadcast unbundled `ExtendFootprintTTLOp` transactions directly to Stellar validators, paying standard base fees in XLM.
* Keepers claim compensation separately by presenting cryptographic proof of the extension to `LazarusVault` in a batched `record_bump_batch` call.

---

### Challenge 2: Complete Coverage of Contract Storage
Extending a contract's persistent data entries is useless if the contract's `Instance` storage or its compiled `ContractCode` WASM bytecode expires. If the instance expires, invocations trap at the host VM level.

**Solution:**
During enrollment, the indexer automatically injects two synthetic entries into `StorageKeyIndex`:
* **Contract Instance Key:** A `LedgerKey::ContractData` entry with `SCValType.SCV_LEDGER_KEY_CONTRACT_INSTANCE`.
* **Contract Code WASM Key:** A `LedgerKey::ContractCode` entry referencing the contract's 32-byte WASM hash.

Keepers treat persistent entries, instance keys, and WASM bytecodes identically when packing footprints, ensuring that executable code never expires.

---

### Challenge 3: Replay Prevention & Out-of-Order Execution
If vouchers used sequential nonces (`nonce == prev_nonce + 1`), then if voucher #3 targeted a contract whose owner called `emergency_withdraw`, voucher #3 would fail with `ContractInactive` or `InsufficientEscrow`, permanently deadlocking vouchers #4 through #10.

**Solution:**
* **Domain Separation:** Every voucher payload prepends `"LAZARUS_VOUCHER_V1"` and the vault contract address before hashing, preventing cross-contract or cross-network replay.
* **Hole-Punching Nonce Tracking:** `LazarusVault` tracks redemptions per `(keeper, nonce)` pair in `DataKey::RedeemedVoucher`. If an intermediate voucher cannot be settled, the keeper can skip it and redeem subsequent vouchers without failure.

---

### Challenge 4: Rate Limiting & Griefing Protection
If an oracle key were compromised or a rogue contract registered with an excessive bounty, an attacker could attempt to exhaust the vault's escrow reserves.

**Solution:**
* **Bounty Registration Ceiling:** In `register_contract`, a single `fixed_bounty` is capped at 20% (`MaxHourlyOutflow / 5`) of the global hourly outflow.
* **Per-Contract Hourly Cap:** Each contract is limited to `max_contract_hourly_outflow = max(MaxHourlyOutflow / 2, fixed_bounty * 2)`.
* **Global Hourly Cap:** All payouts across the vault in `current_hour = timestamp / 3600` are tracked in `DataKey::HourlyOutflowAccumulator` and capped at `MaxHourlyOutflow`.
* **2-of-3 FROST MPC Quorum:** The production attestation service distributes shares across three independent operators, requiring at least two nodes to co-sign every voucher.
