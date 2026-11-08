# Getting Started with Lazarus Protocol

This guide walks through getting started with Lazarus Protocol, both as a smart contract developer protecting a protocol from storage archival, and as an autonomous keeper node operator earning USDC bounties.

---

## 1. For Protocol Integrators

Protocols enroll their contracts in Lazarus to automate persistent state maintenance. All funds deposited remain non-custodial.

### Option A: Web App Enrollment (Recommended)

1. Open the Lazarus Console at [https://lazarusprotocol.online/vaults](https://lazarusprotocol.online/vaults).
2. Connect your Stellar wallet (Freighter, Albedo, Lobstr, or xBull).
3. Enter your Soroban Contract ID (`C...`, 56 characters).
4. Specify your initial escrow deposit (in USDC) and your fixed bounty reward per extended batch (e.g., `0.05` to `0.10` USDC).
5. Confirm the transaction in your wallet.

Once registered, the Lazarus Hybrid Discovery Pipeline automatically indexes all persistent storage keys, contract instances, and WASM bytecodes for your contract.

### Option B: Programmatic Enrollment via Stellar CLI

You can register a contract directly on-chain using the `stellar` CLI:

```bash
# 1. Authorize SAC token allowance for the Lazarus Vault
stellar contract invoke \
  --id CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC \
  --source-account YOUR_ADMIN_SECRET \
  --network testnet \
  -- approve \
  --from YOUR_ADMIN_ADDRESS \
  --spender CCWAH7BVPGYOEBPRFWPZIONBYUCCQH7OARGZLOD6ZV66SOAFBVA2DHD5 \
  --amount 1000000000 \
  --live_until_ledger_seq 3000000

# 2. Register your contract in LazarusVault
stellar contract invoke \
  --id CCWAH7BVPGYOEBPRFWPZIONBYUCCQH7OARGZLOD6ZV66SOAFBVA2DHD5 \
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

*Note: All token amounts and bounties use 7 decimal places (stroops). `1000000` = 0.10 USDC.*

### Topping Up Escrow

When your escrow reserve runs low, the registered `client` can deposit additional funds or reactivate an inactive contract:

```bash
stellar contract invoke \
  --id CCWAH7BVPGYOEBPRFWPZIONBYUCCQH7OARGZLOD6ZV66SOAFBVA2DHD5 \
  --source-account YOUR_ADMIN_SECRET \
  --network testnet \
  -- deposit_funds \
  --client YOUR_ADMIN_ADDRESS \
  --target YOUR_CONTRACT_ID \
  --amount 500000000
```

### Emergency Withdrawal

The protocol administrator retains 100% sovereign ownership over deposited escrow funds. To withdraw all unspent escrow tokens immediately:

```bash
stellar contract invoke \
  --id CCWAH7BVPGYOEBPRFWPZIONBYUCCQH7OARGZLOD6ZV66SOAFBVA2DHD5 \
  --source-account YOUR_ADMIN_SECRET \
  --network testnet \
  -- emergency_withdraw \
  --client YOUR_ADMIN_ADDRESS \
  --target YOUR_CONTRACT_ID
```

This immediately transfers the full remaining balance back to `client` and safely sets `is_active = false`.

---

## 2. For Keeper Node Operators

Keepers monitor expiring storage keys, submit physical footprint extension transactions (`ExtendFootprintTTLOp`), request signed vouchers, and redeem them on-chain for USDC bounties.

### Quick Start with Docker

Pre-built container packages are available directly from the GitHub Container Registry under [https://github.com/LazarusProtocol](https://github.com/LazarusProtocol):

```bash
# Pull the latest keeper image
docker pull ghcr.io/lazarusprotocol/keeper-daemon:latest

# Run the keeper daemon connected to testnet
docker run -d \
  --name lazarus-keeper \
  --restart unless-stopped \
  -e LAZARUS_KEEPER_SECRET="S...YOUR_STELLAR_SECRET_KEY" \
  -e LAZARUS_INDEXER_API_URL="https://api.lazarusprotocol.online" \
  -e LAZARUS_RPC_URL="https://soroban-testnet.stellar.org" \
  -e LAZARUS_VAULT_CONTRACT_ID="CCWAH7BVPGYOEBPRFWPZIONBYUCCQH7OARGZLOD6ZV66SOAFBVA2DHD5" \
  ghcr.io/lazarusprotocol/keeper-daemon:latest
```

### Inspecting Node Logs

```bash
docker logs -f lazarus-keeper
```

Example output:
```text
2026-11-05 06:12:00 | INFO | lazarus.keeper.daemon | Initialized keeper GAK... target_vault=CCWA...
2026-11-05 06:12:05 | INFO | lazarus.keeper.daemon | Polled 42 expiring keys across 3 contracts
2026-11-05 06:12:07 | INFO | lazarus.keeper.daemon | Broadcasted ExtendFootprintTTLOp (100 keys) tx=a8f2...
2026-11-05 06:12:12 | INFO | lazarus.keeper.daemon | Received signed BumpVoucher nonce=44 from attestation oracle
2026-11-05 06:12:15 | INFO | lazarus.keeper.daemon | Settled batch of 10 vouchers in record_bump_batch payout=1.00 USDC
```

