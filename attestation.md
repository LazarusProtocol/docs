# Attestation Engine & Cryptographic Vouchers

The Lazarus Attestation Engine bridges off-chain proof of physical storage extension with on-chain Soroban escrow settlement. It verifies Stellar RPC transaction consensus, manages monotonic nonces, and signs `BumpVoucher` structures using Ed25519 and 2-of-3 FROST Multi-Party Computation (MPC).

* **GitHub Repository:** [https://github.com/LazarusProtocol/attestation](https://github.com/LazarusProtocol/attestation)
* **Container Package:** `ghcr.io/lazarusprotocol/attestation-enclave:latest`

---

## 1. The Decoupled Voucher Pattern

Stellar Core strictly mandates that a single transaction envelope may contain only one Soroban operation. `ExtendFootprintTTLOp` and `InvokeHostFunctionOp` cannot share an envelope.

Lazarus resolves this constraint by decoupling physical state warming from economic payment into three distinct phases:

1. **Phase 1: Physical TTL Extension (`ExtendFootprintTTLOp`)**
   - The keeper packs up to 100 expiring keys into a transaction footprint.
   - Submits directly to Stellar validators (pays a minimal base fee in XLM).

2. **Phase 2: RPC Consensus Attestation**
   - The keeper sends the transaction hash to the Attestation Engine.
   - The engine queries Soroban RPC `getTransaction` to confirm consensus, ledger finality, and updated TTL.
   - Generates a signed `BumpVoucher` (`LAZARUS_VOUCHER_V1`).

3. **Phase 3: Batched On-Chain Settlement (`record_bump_batch`)**
   - The keeper submits up to 10 vouchers in a single `InvokeHostFunctionOp`.
   - `LazarusVault` verifies the Ed25519 signature on-chain and dispatches USDC rewards.

---

## 2. On-Chain RPC Consensus Verification

To prevent attackers from generating valid vouchers using arbitrary or fabricated transaction hashes, the attestation engine enforces strict on-chain consensus checks before signing (`indexer/indexer/services.py`):

1. **RPC Query:** Queries Soroban RPC `getTransaction(tx_hash)`.
2. **Status Check:** Confirms `status == "SUCCESS"`.
3. **Ledger Inclusion:** Validates that the transaction was included in a finalized ledger sequence.
4. **Footprint Proof:** Inspects transaction metadata to verify that the target contract's keys were actually extended to or beyond `bump_target`.

Only transactions that meet all four requirements receive a signed voucher.

---

## 3. Cryptographic Domain Separation

Every voucher signature is calculated over a deterministic binary payload with explicit domain separation:

```text
payload = 
    "LAZARUS_VOUCHER_V1"            (Domain Header)
  + vault_contract_address          (On-chain Vault ID)
  + keeper_stellar_address          (Keeper Recipient Public Key)
  + target_contract_address         (Enrolled Contract ID)
  + ledger_bumped                   (u32 Target Ledger Sequence)
  + bounty_amount                   (i128 Payout in Stroops)
  + nonce                           (u64 Per-Keeper Monotonic Nonce)
```

The payload is SHA-256 hashed and signed by the oracle's private key. 

On-chain, `LazarusVault` reconstructs the exact same payload in Soroban and invokes:
```rust
let digest: BytesN<32> = e.crypto().sha256(&payload_bytes).into();
e.crypto().ed25519_verify(&oracle_pubkey, &digest.into(), signature);
```

### Security Guarantees
* **Cross-Contract Replay Immunity:** Binding `vault_contract_address` ensures a voucher cannot be redeemed against another vault.
* **Network Isolation:** Binding contract IDs (which include the Stellar network passphrase hash) prevents vouchers signed on Testnet from executing on Mainnet.
* **Mempool Sniping Immunity:** Binding `keeper_stellar_address` ensures that even if a rival node intercepts a voucher in the public transaction queue, they cannot redeem the payout to their own wallet.

---

## 4. Signing Infrastructure: Enclave & 2-of-3 FROST MPC

The protocol supports two isolated signing modes (`attestation/`):

### Mode 1: AWS Nitro Enclave Isolated Signer (`enclave_signer.py`)
In standalone deployments, the oracle private key is stored within a hardware-isolated AWS Nitro Enclave with memory encryption and cryptographic attestation documents.

### Mode 2: 2-of-3 FROST MPC Threshold Signer (`frost_coordinator.py`)
In decentralized production environments, the oracle key is partitioned among three independent node operators using the Flexible Round-Optimized Schnorr Threshold (FROST) protocol:
* Any 2 of the 3 nodes must independently verify the RPC consensus proof before co-signing the voucher.
* No single node possesses the master private key, eliminating single-point-of-compromise risks.

---

## 5. Nonce Hole-Punching Replay Protection

To prevent replay attacks while allowing out-of-order execution, `LazarusVault` tracks redemptions per voucher:

```rust
// contracts/lazarus_vault/src/contract.rs
let redeemed_key = DataKey::RedeemedVoucher(voucher.keeper.clone(), voucher.nonce);
if e.storage().persistent().get(&redeemed_key).unwrap_or(false) {
    return Err(VaultError::InvalidNonce);
}
e.storage().persistent().set(&redeemed_key, &true);
```

If voucher #3 targets a contract whose balance is exhausted or which has called `emergency_withdraw`, the keeper can skip voucher #3 and submit vouchers #4 through #10 without causing the batch to fail.

