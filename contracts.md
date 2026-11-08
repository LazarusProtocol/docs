# Smart Contracts (`lazarus_vault`) — Architecture & API Reference

The Lazarus settlement and escrow layer resides in `contracts/lazarus_vault/`. It is implemented in Rust using `soroban-sdk` v27, compiles to `#![no_std]` WebAssembly (`wasm32v1-none`), and handles non-custodial protocol escrow, cryptographic voucher verification, fee collection, and dual-tier rate limiting.

* **GitHub Repository:** [https://github.com/LazarusProtocol/contracts](https://github.com/LazarusProtocol/contracts)
* **Web Console:** [https://lazarusprotocol.online/](https://lazarusprotocol.online/)
* **Indexer API:** [https://api.lazarusprotocol.online/](https://api.lazarusprotocol.online/)

---

## 1. Live Stellar Testnet Deployments

| Parameter | Identifier / Address |
| :--- | :--- |
| **Network** | Stellar Testnet (`Test SDF Network ; September 2015`) |
| **Soroban RPC** | `https://soroban-testnet.stellar.org` |
| **Vault Contract ID** | `CBKFA2F7RUDGLJNE3K5X2XEXVXRJQ5AMB2NSPREDTATE2ZHDALFKPGUR` |
| **WASM Hash** | `38ad59f3c5ce3f8b6e5e943865b8bfa473c24d116e137bc47b0249c3cc0f757b` |
| **Escrow SAC Token (USDC)** | `CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC` |
| **Oracle / FROST PubKey** | `197f6b23e16c8532c6abc838facd5ea789be0c76b2920334039bfa8b3d368d61` |
| **Explorer** | [stellar.expert/explorer/testnet/contract/CBKFA2...](https://stellar.expert/explorer/testnet/contract/CBKFA2F7RUDGLJNE3K5X2XEXVXRJQ5AMB2NSPREDTATE2ZHDALFKPGUR) |

---

## 2. Data Types & Storage Architecture (`src/types.rs`)

All token balances, bounties, protocol fees, and circuit breaker accumulators use signed 128-bit integers (`i128`) in stroops (`10^-7` units) to comply with Stellar Asset Contract (SAC) and SEP-41 token standards.

### Constants
```rust
pub const DAY_IN_LEDGERS: u32 = 17_280;
pub const BUMP_THRESHOLD: u32 = 30 * DAY_IN_LEDGERS; // 518,400 ledgers (~30 days)
pub const BUMP_TO: u32 = 120 * DAY_IN_LEDGERS;       // 2,073,600 ledgers (~120 days)
pub const MAX_BATCH_SIZE: u32 = 10;
pub const PROTOCOL_FEE_BPS: i128 = 500;              // 5.00% (500 basis points)
pub const BPS_DENOMINATOR: i128 = 10_000;
```

### Structures & Enums

#### `VaultConfig`
Stores configuration, active escrow balance, and rate limits for an enrolled target contract:
```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct VaultConfig {
    pub client: Address,                   // Protocol Admin / Depositor
    pub balance: i128,                     // Active escrow balance in stroops
    pub policy: BountyPolicy,              // BountyPolicy::Fixed(i128)
    pub min_ledger_threshold: u32,         // Minimum trigger window (e.g. 345,600 ledgers)
    pub bump_target: u32,                  // Target TTL extension (e.g. 2,073,600 ledgers)
    pub max_contract_hourly_outflow: i128, // Per-contract hourly circuit breaker cap
    pub is_active: bool,                   // Active warming status flag
}
```

#### `BumpVoucher`
The cryptographic voucher payload signed by the Attestation Oracle:
```rust
#[contracttype]
#[derive(Clone, Debug, Eq, PartialEq)]
pub struct BumpVoucher {
    pub keeper: Address,          // Recipient keeper node operator
    pub target_contract: Address, // Enrolled contract whose storage was bumped
    pub ledger_bumped: u32,       // Target ledger sequence achieved
    pub bounty_amount: i128,      // Deterministic payout amount (in stroops)
    pub nonce: u64,               // Per-keeper monotonic voucher nonce
}
```

#### `DataKey` Storage Schema
```rust
#[contracttype]
#[derive(Clone)]
pub enum DataKey {
    Admin,                               // [Instance] Administrator address
    OraclePubKey,                        // [Instance] 32-byte Ed25519 / FROST group public key
    Token,                               // [Instance] SAC Token Address (e.g. USDC)
    MaxHourlyOutflow,                    // [Instance] Global economic circuit breaker ceiling
    CurrentHour,                         // [Instance] Current hour timestamp (u64)
    HourlyOutflowAccumulator,            // [Instance] Outflow accumulated in current hour (i128)
    ContractHourlyOutflow(Address, u64), // [Persistent] (target, hour) -> i128 outflow
    AccumulatedFees,                     // [Instance] Protocol fee treasury accumulator (i128)
    Registration(Address),               // [Persistent] Target Contract Address -> VaultConfig
    KeeperNonce(Address),                // [Persistent] Keeper Address -> u64 Highest Nonce
    RedeemedVoucher(Address, u64),       // [Persistent] (Keeper, Nonce) -> bool marker
}
```

### Self-Preserving Storage TTL Strategy
On every contract invocation, `lazarus_vault` extends its own instance and persistent storage keys:
```rust
e.storage().instance().extend_ttl(BUMP_THRESHOLD, BUMP_TO);
e.storage().persistent().extend_ttl(&DataKey::Registration(target), BUMP_THRESHOLD, BUMP_TO);
```
This guarantees that the vault contract itself never expires as long as active transactions occur.

---

## 3. Public Method Interface Reference

### `initialize`
Initializes administrative keys, oracle public key, payment token, and the global hourly circuit breaker.
```rust
pub fn initialize(
    e: Env,
    admin: Address,
    oracle_pubkey: BytesN<32>,
    token: Address,
    max_hourly_outflow: i128,
) -> Result<(), VaultError>
```
* **Auth:** None (can only be executed once upon contract deployment).
* **Errors:** `VaultError::AlreadyInitialized`, `VaultError::BountyOutOfBounds`.

---

### `register_contract`
Registers a client contract with an initial USDC escrow deposit and configures warming policies.
```rust
pub fn register_contract(
    e: Env,
    client: Address,
    target: Address,
    initial_deposit: i128,
    fixed_bounty: i128,
    min_threshold: u32,
    bump_target: u32,
) -> Result<(), VaultError>
```
* **Auth:** Requires `client` authorization (`client.require_auth()`).
* **Protections:**
  * Self-registration guard: `target != e.current_contract_address()`.
  * Existence guard: Reverts with `VaultError::AlreadyRegistered` if `target` already has an active vault or balance > 0, preventing hostile registration overwrites.
  * Griefing cap: `fixed_bounty` cannot exceed 20% of `MaxHourlyOutflow` (`max_hourly / 5`).
  * Per-contract hourly cap: Set to `max(max_hourly / 2, fixed_bounty * 2)`.
* **Token Transfer:** Pulls `initial_deposit` from `client` into `e.current_contract_address()`.

---

### `deposit_funds`
Deposits additional escrow funds for an existing registered contract.
```rust
pub fn deposit_funds(
    e: Env,
    client: Address,
    target: Address,
    amount: i128,
) -> Result<(), VaultError>
```
* **Auth:** Requires `client` authorization (`client.require_auth()`) matching `config.client`.
* **Anti-Trapping & Reactivation:** Only the registered `client` who owns the vault escrow may deposit funds (`VaultError::Unauthorized` for third-party callers). If the contract was previously deactivated (e.g. via `emergency_withdraw`), deposits by the original `client` automatically reactivate monitoring (`config.is_active = true`).

---

### `record_bump`
Redeems a single attestation voucher and dispatches the bounty payout.
```rust
pub fn record_bump(
    e: Env,
    voucher: BumpVoucher,
    signature: BytesN<64>,
) -> Result<(), VaultError>
```
* **Auth:** Requires `voucher.keeper.require_auth()` plus a valid Ed25519 oracle signature over the canonical `LAZARUS_VOUCHER_V1` digest.

---

### `record_bump_batch`
Redeems a batch of up to 10 vouchers in a single transaction, disbursing USDC to keepers in bulk and amortizing Soroban invocation fees.
```rust
pub fn record_bump_batch(
    e: Env,
    vouchers: Vec<BumpVoucher>,
    signatures: Vec<BytesN<64>>,
) -> Result<(), VaultError>
```
* **Auth:** Requires `keeper.require_auth()` for each distinct keeper in the batch plus valid Ed25519 oracle signatures.
* **Batch Constraints:** `1 <= vouchers.len() <= MAX_BATCH_SIZE (10)` and `vouchers.len() == signatures.len()`.
* **Out-of-Order Execution:** If an intermediate voucher cannot be redeemed (e.g., target contract called `emergency_withdraw`), subsequent vouchers can still be processed without bricking the keeper's nonce sequence.

---

### `emergency_withdraw`
Enables the contract depositor to immediately retrieve 100% of unspent escrow balance.
```rust
pub fn emergency_withdraw(
    e: Env,
    client: Address,
    target: Address,
) -> Result<(), VaultError>
```
* **Auth:** Requires `client` authorization (`client.require_auth()`). Must match `config.client`.
* **State Effect:** Refunds remaining balance to `client`, resets `config.balance = 0`, and sets `config.is_active = false`.
* **Zero-Refund Guard:** Bypasses token transfer calls if balance is already 0.

---

### `withdraw_protocol_fees`
Withdraws accumulated 5% protocol fees to a designated treasury address.
```rust
pub fn withdraw_protocol_fees(
    e: Env,
    admin: Address,
    to: Address,
    amount: i128,
) -> Result<(), VaultError>
```
* **Auth:** Requires `admin` authorization (`admin.require_auth()`).

---

### `update_oracle_pubkey`
Rotates the oracle public key (e.g. migrating from Nitro Enclave to 2-of-3 FROST MPC).
```rust
pub fn update_oracle_pubkey(
    e: Env,
    admin: Address,
    new_oracle_pubkey: BytesN<32>,
) -> Result<(), VaultError>
```
* **Auth:** Requires `admin` authorization (`admin.require_auth()`).

---

### `update_max_hourly_outflow`
Updates the global hourly circuit breaker limit.
```rust
pub fn update_max_hourly_outflow(
    e: Env,
    admin: Address,
    new_max_outflow: i128,
) -> Result<(), VaultError>
```
* **Auth:** Requires `admin` authorization (`admin.require_auth()`).

---

## 4. Read-Only Query Methods

| Method | Return Type | Description |
| :--- | :--- | :--- |
| `get_registration(target)` | `Result<VaultConfig, VaultError>` | Fetches configuration and balance for an enrolled contract |
| `get_keeper_nonce(keeper)` | `u64` | Returns highest redeemed watermark nonce for a keeper |
| `is_voucher_redeemed(keeper, nonce)` | `bool` | Checks if a specific voucher nonce has been redeemed |
| `get_accumulated_fees()` | `i128` | Returns accumulated 5% protocol fees in treasury |
| `get_hourly_outflow()` | `(u64, i128, i128)` | Returns `(current_hour, hourly_accumulator, max_hourly_outflow)` |
| `get_oracle_pubkey()` | `Result<BytesN<32>, VaultError>` | Returns the active 32-byte Ed25519 oracle public key |

---

## 5. Security Architecture & Invariants

When settling vouchers in `record_bump` or `record_bump_batch`, the contract evaluates:

1. **Nonce Hole-Punch Check:** Confirms `DataKey::RedeemedVoucher(keeper, nonce)` is false.
2. **Active Target Contract Check:** Verifies the target contract exists and `is_active == true`.
3. **Strict Bounty Policy Check:** Ensures `voucher.bounty_amount` equals the contract's fixed policy.
4. **Domain-Separated Host Signature Verification:** Computes the SHA-256 digest of the payload and verifies the Ed25519 signature via `env.crypto().ed25519_verify()`.
5. **95/5 Conservation Split:** Calculates `protocol_fee = bounty * 500 / 10000` and validates that `balance >= bounty + protocol_fee`.
6. **Dual Hourly Circuit Breakers:** Ensures both global and per-contract hourly outflow limits are respected.
7. **State Commit & Settlement:** Deducts balance, marks the voucher as redeemed, and transfers USDC to the keeper.

### The 4 On-Chain Security Locks
1. **Cryptographic Envelope:** Domain header `"LAZARUS_VOUCHER_V1"` binds the vault address, keeper address, target contract, bumped ledger, bounty amount, and nonce. Verified via `env.crypto().ed25519_verify()`.
2. **Nonce Hole-Punching Replay Guard:** Dual tracking via `DataKey::RedeemedVoucher(keeper, nonce)` and `DataKey::KeeperNonce(keeper)` allows out-of-order execution without sequential deadlocks.
3. **95/5 Conservation Payout Split:** Keepers receive 100% of the declared bounty. An additional 5% protocol fee is credited to `DataKey::AccumulatedFees`.
4. **Dual Hourly Outflow Circuit Breakers:** Global ceiling (`MaxHourlyOutflow`) and per-contract ceiling (`max_contract_hourly_outflow`) prevent draining attacks even in the event of oracle compromise.

---

## 6. Complete Error Code Reference (`VaultError`)

*Note: Error code 9 (`InvalidSignature`) is intentionally absent. In Soroban, `env.crypto().ed25519_verify()` traps directly at the host VM level on signature failure.*

| Code | Variant | Trigger Condition |
| :--- | :--- | :--- |
| `1` | `NotInitialized` | Vault storage is missing `Admin`, `OraclePubKey`, `Token`, or `MaxHourlyOutflow` |
| `2` | `AlreadyInitialized` | `initialize` called when contract is already initialized |
| `3` | `ContractInactive` | Target contract is not registered or `VaultConfig.is_active == false` |
| `4` | `InsufficientEscrow` | Contract escrow balance is less than `bounty_amount + protocol_fee` |
| `5` | `InvalidNonce` | `voucher.nonce == 0` or voucher has already been redeemed |
| `6` | `BountyMismatch` | `voucher.bounty_amount` differs from enrolled `BountyPolicy::Fixed` |
| `7` | `BountyOutOfBounds` | Parameter `<= 0` or `fixed_bounty > MaxHourlyOutflow / 5` |
| `8` | `ArithmeticError` | Checked arithmetic overflow or underflow occurred |
| `10` | `BatchSizeExceeded` | `vouchers.len() > MAX_BATCH_SIZE (10)` in `record_bump_batch` |
| `11` | `VectorMismatch` | `vouchers.len() != signatures.len()` in `record_bump_batch` |
| `12` | `Unauthorized` | Caller does not match stored `client` or `Admin` address |
| `13` | `HourlyOutflowExceeded` | Global hourly ceiling or per-contract hourly ceiling exceeded |
| `14` | `InvalidTarget` | Target address matches the vault's own contract address |
| `15` | `AlreadyRegistered` | Target contract already has an active vault or balance > 0 |

---

## 7. Testing & Verification

The contract includes a comprehensive test suite (`contracts/lazarus_vault/src/test.rs`) covering 11 integration and unit tests:

```bash
cargo test --manifest-path contracts/Cargo.toml
```

### Verified Test Cases
1. `test_initialize_and_double_init_rejected`: Prevents double initialization.
2. `test_register_contract_and_overwrite_guard`: Verifies existence guard and 20% bounty ceiling.
3. `test_deposit_funds_increases_escrow`: Verifies escrow top-ups.
4. `test_record_bump_single_settlement_and_5_percent_fee`: Verifies single voucher redemption and 5% fee math.
5. `test_replay_protection_out_of_order_batch_tolerance_and_strict_billing`: Verifies nonce hole-punching and out-of-order execution tolerance.
6. `test_record_bump_batch_and_limits`: Verifies batch size limit (`MAX_BATCH_SIZE = 10`) and vector length matching.
7. `test_hourly_outflow_circuit_breaker_and_hour_rollover`: Verifies timestamp hourly rollover and circuit breaker enforcement.
8. `test_emergency_withdraw_and_protocol_fee_withdrawal`: Verifies 100% balance refund and administrative fee collection.
9. `test_admin_update_oracle_pubkey_and_max_hourly_outflow`: Verifies key rotation and limit updates.
10. `test_reactivate_after_emergency_withdraw_and_third_party_rejected`: Verifies deposit reactivation by client and rejection of third-party deposits.
11. `test_threshold_bounds_and_empty_batch_rejected`: Verifies threshold/bump bounds (`MIN_THRESHOLD_LEDGERS`, `MAX_TTL_LEDGERS`) and rejects empty batches.

---

## 8. Soroban CLI Cheat Sheet

### Compile to Wasm
```bash
cargo build --target wasm32v1-none --release --manifest-path contracts/Cargo.toml
```

### Register Contract with Escrow
```bash
stellar contract invoke \
  --id CBKFA2F7RUDGLJNE3K5X2XEXVXRJQ5AMB2NSPREDTATE2ZHDALFKPGUR \
  --source-account YOUR_ADMIN_SECRET \
  --network testnet \
  -- register_contract \
  --client YOUR_ADMIN_ADDRESS \
  --target YOUR_CONTRACT_ID \
  --initial_deposit 100000000 \
  --fixed_bounty 1000000 \
  --min_threshold 345600 \
  --bump_target 2073600
```

### Emergency Withdrawal
```bash
stellar contract invoke \
  --id CBKFA2F7RUDGLJNE3K5X2XEXVXRJQ5AMB2NSPREDTATE2ZHDALFKPGUR \
  --source-account YOUR_ADMIN_SECRET \
  --network testnet \
  -- emergency_withdraw \
  --client YOUR_ADMIN_ADDRESS \
  --target YOUR_CONTRACT_ID
```
