# Analytical Storage Key Indexer — Architecture & API Reference

The Lazarus Indexer (`indexer/`) is a high-throughput Django REST Framework, Celery, and Redis service that continuously monitors Stellar ledger state, indexes Soroban storage keys, computes expiration schedules, and serves keeper telemetry.

* **Indexer Repository:** [https://github.com/LazarusProtocol/indexer](https://github.com/LazarusProtocol/indexer)
* **Production API Base URL:** `https://api.lazarusprotocol.online/`

---

## 1. The Soroban Key Enumeration Problem

Soroban JSON-RPC does not provide a method to list all storage keys belonging to a contract (there is no `getContractKeys` RPC endpoint). The RPC only allows querying entries when callers supply the exact Base64 XDR `LedgerKey` (`getLedgerEntries(keys[])`).

Without a comprehensive key discovery service:
* Protocols cannot identify which user accounts or positions are approaching archival.
* Keepers cannot assemble `ExtendFootprintTTLOp` footprints.

---

## 2. The 4-Stage Hybrid Key Discovery Pipeline

To reconstruct 100% of a contract's storage footprint, Lazarus implements a 4-stage hybrid discovery pipeline (`indexer/indexer/services.py`):

1. **Stage A (Hubble BigQuery):** Analytical bulk export from Stellar Foundation's Hubble dataset.
2. **Stage B (RPC Catch-Up Replay):** Paginates `getLedgers` from the BigQuery checkpoint to the current ledger head.
3. **Stage C (Protocol Adapters):** Deterministic storage-tree walkers (e.g. Blend, Phoenix) for compound keys.
4. **Stage D (Live Ledger Poller):** Celery background tasks refresh `live_until_ledger_seq` as new blocks close.

### Stage A: Hubble BigQuery Snapshot
Queries the Stellar Foundation's public BigQuery dataset (`crypto-stellar.crypto_stellar_dbt.contract_data_current` joined with `ttl_current`) for all active persistent and instance keys:

```sql
SELECT 
    cd.contract_id,
    cd.ledger_key_hash,
    cd.key_xdr,
    cd.durability,
    ttl.live_until_ledger_seq
FROM `crypto-stellar.crypto_stellar_dbt.contract_data_current` cd
LEFT JOIN `crypto-stellar.crypto_stellar_dbt.ttl_current` ttl
    ON cd.ledger_key_hash = ttl.key_hash
WHERE cd.contract_id = @contract_id;
```

**Synthetic Bytecode Entries:** In addition to contract data keys, Lazarus injects synthetic entries for the contract's `Instance` and `ContractCode` (`LedgerKeyContractCode`), ensuring that the underlying compiled WASM bytecode is tracked and kept warm.

### Stage B: RPC Catch-Up Replay
To bridge the ingestion delay between BigQuery's analytical snapshots and the live ledger head, the pipeline queries Soroban RPC `getLedgers` in batches of 100 ledgers, parsing transaction meta for newly created storage keys.

### Stage C: Protocol Adapters
Certain DeFi protocols (e.g., Blend lending pools) store user data in compound storage keys constructed from user public keys and reserve symbols. Lazarus utilizes deterministic protocol adapters (`BlendProtocolAdapter`) that inspect pool configurations and derive all underlying reserve and pool keys.

### Stage D: Live Ledger Poller
Background Celery workers query Soroban RPC `getLedgerEntries` every 30 seconds for all monitored keys, maintaining up-to-the-ledger expiration records in PostgreSQL with Redis caching.

---

## 3. Database Schema

The core PostgreSQL models (`indexer/indexer/models.py`) include:

* **`MonitoredContract`:** Represents an enrolled contract, tracking client address, escrow balance, fixed bounty rate, minimum TTL threshold, bump target, and protocol adapter type (`generic`, `blend`, `soroswap`).
* **`StorageKeyIndex`:** Individual storage entry, recording `key_hash`, `key_xdr`, `durability`, `live_until_ledger_seq`, and `last_extended_ledger`.
* **`KeeperRegistry`:** Tracks registered keeper operator addresses, monotonic `current_nonce`, and `total_bounties_claimed`.
* **`VoucherLog`:** Historical ledger of signed vouchers, tracking keeper address, `tx_hash`, `nonce`, `ledger_bumped`, `bounty_amount`, Ed25519 `signature`, and on-chain `redeemed` status.

---

## 4. REST API Reference

All endpoints are hosted at `https://api.lazarusprotocol.online/`.

### `GET /api/v1/contracts`
Lists all enrolled contracts, active escrow balances, indexed key counts, and registered policies.

### `POST /api/v1/contracts`
Registers a new contract for automated monitoring and triggers the 4-stage hybrid discovery pipeline.
```json
{
  "contract_id": "CA3D5KRYM6CB7OWQ6TWYRR3Z4T7GNZLKERYNZGGA5SOAOPIFY6YQGAXE",
  "client_address": "GDEJF7VHNX4KINR7EJGWXLJYWVG2V6BC7ZVJ4TZOY7C33F4L42SJCZFC",
  "escrow_balance": "100000000",
  "fixed_bounty": "1000000",
  "min_ledger_threshold": 345600,
  "bump_target": 2073600,
  "protocol_type": "generic"
}
```

### `GET /api/v1/keys/expiring`
Polled by keeper nodes to discover keys requiring TTL extension.
* **Query Parameters:**
  * `current_ledger`: Current Stellar ledger sequence number (default: `500000`).
* **Response:**
  ```json
  {
    "current_ledger": 1284500,
    "count": 42,
    "results": [
      {
        "id": 1,
        "contract_id": "CCWAH7...",
        "key_hash": "a8f3b...",
        "key_xdr": "AAAAFA...",
        "durability": "persistent",
        "live_until_ledger_seq": 1300000,
        "remaining_ledgers": 15500,
        "min_ledger_threshold": 20000,
        "bump_target": 1784500,
        "bump_ledgers_delta": 500000,
        "fixed_bounty": 1000000,
        "is_emergency": false
      }
    ]
  }
  ```

### `GET /api/v1/diagnostics/heatmap`
Returns categorized TTL health distribution (`Healthy`, `Warming`, `Critical`) for the web diagnostics console.

### `POST /api/v1/attestation/voucher`
Keeper submission endpoint to obtain a cryptographically signed `BumpVoucher` after broadcasting an on-chain `ExtendFootprintTTLOp`. See [attestation.md](attestation.md) for details.

### `GET /api/v1/health`
Liveness probe returning PostgreSQL, Redis, and Celery connectivity status.

