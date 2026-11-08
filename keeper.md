# Autonomous Keeper Daemon — Architecture & Deployment

The Lazarus Keeper Node (`keeper/`, standalone repo `keeper`) is an autonomous asynchronous daemon that monitors Soroban contract storage entries nearing expiration, broadcasts batched `ExtendFootprintTTLOp` transactions, obtains cryptographic `BumpVoucher` attestations, and settles rewards on-chain.

* **GitHub Repository:** [https://github.com/LazarusProtocol/keeper](https://github.com/LazarusProtocol/keeper)
* **Container Package:** `ghcr.io/lazarusprotocol/keeper-daemon:latest`

---

## 1. Keeper Execution Lifecycle

The keeper daemon executes a continuous asynchronous event loop (`keeper/daemon.py`):

1. **Poll Expiring Keys:** Calls `GET /api/v1/keys/expiring` on the indexer and filters keys based on cohort eligibility.
2. **Assemble & Pack Footprint:** Groups up to 100 `LedgerKey` entries into a single `ExtendFootprintTTLOp` transaction, signs, and broadcasts it to the network.
3. **Attestation Request:** Submits the extension transaction hash to the Attestation Engine. The engine verifies RPC consensus and returns a signed `BumpVoucher`.
4. **Buffer & Batch Settlement:** Buffers up to 10 signed vouchers and submits them in a single `record_bump_batch` call on `LazarusVault` to claim USDC bounties.

---

## 2. Deterministic Modulo Cohort Sharding

To prevent multiple keeper nodes from competing over the same expiring keys and burning redundant XLM fees, the keeper network employs deterministic sharding:

### The Sharding Algorithm
During normal warming windows (`2,000 < remaining_ledgers <= min_ledger_threshold`), each keeper determines eligibility via:

```python
# keeper/keeper/daemon.py
def is_eligible_for_key(self, key_hash: str, current_ledger: int, remaining_ledgers: int) -> bool:
    if remaining_ledgers <= EMERGENCY_THRESHOLD:
        return True
    epoch = current_ledger // 100
    seed = f"{self.keypair.public_key}:{key_hash}:{epoch}".encode("utf-8")
    cohort = int(hashlib.sha256(seed).hexdigest(), 16) % COHORT_MODULO
    return cohort == 0
```

* Monitored keys are partitioned across 5 distinct cohorts.
* Key assignments rotate every 100 ledgers (~8.3 minutes), ensuring fair distribution across all active keepers.

### Emergency Threshold Bypass
When an entry reaches `remaining_ledgers <= 2,000` (~2.7 hours before archival eviction), the sharding algorithm bypasses cohort restrictions:

```python
if remaining_ledgers <= EMERGENCY_THRESHOLD:
    return True  # Any keeper can extend immediately
```

This guarantees state preservation even if a specific keeper node is offline or experiencing network downtime.

---

## 3. Footprint Packing & Gas Amortization

Stellar Core permits up to 100 keys in the `read_only` footprint of an `ExtendFootprintTTLOp`. 

* Individual extension: 1 key per transaction = 100 base fee invocations.
* Lazarus batched footprint: 100 keys per transaction = 1 base fee invocation (~100 to 500 stroops of XLM).

By amortizing transaction fees across 100 keys simultaneously, keeper gas costs are reduced by over 95%.

---

## 4. Deployment Options

### Option A: Pull & Run Container from GitHub Packages

Pre-built Docker images are published to GitHub Packages under [github.com/LazarusProtocol](https://github.com/LazarusProtocol):

```bash
docker pull ghcr.io/lazarusprotocol/keeper-daemon:latest

docker run -d \
  --name lazarus-keeper \
  --restart unless-stopped \
  -e LAZARUS_KEEPER_SECRET="S...YOUR_STELLAR_SECRET_KEY" \
  -e LAZARUS_INDEXER_API_URL="https://api.lazarusprotocol.online" \
  -e LAZARUS_RPC_URL="https://soroban-testnet.stellar.org" \
  -e LAZARUS_VAULT_CONTRACT_ID="CCWAH7BVPGYOEBPRFWPZIONBYUCCQH7OARGZLOD6ZV66SOAFBVA2DHD5" \
  -e LAZARUS_BUFFER_FLUSH_INTERVAL="600" \
  ghcr.io/lazarusprotocol/keeper-daemon:latest
```

### Option B: Docker Compose (Standalone Keeper)

Clone the standalone keeper repository and run the keeper node:

```bash
git clone https://github.com/LazarusProtocol/keeper.git
cd keeper

# Copy and edit configuration
cp .env.example .env

# Start keeper daemon in background
docker compose up -d --build
```

### Option C: Python CLI

```bash
git clone https://github.com/LazarusProtocol/keeper.git
cd keeper

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Run a single inspection and extension cycle
export LAZARUS_KEEPER_SECRET="S..."
export LAZARUS_INDEXER_API_URL="https://api.lazarusprotocol.online"
PYTHONPATH=. python3 -m keeper.cli --once
```

---

## 5. Configuration Reference

| Environment Variable | Default | Purpose |
| :--- | :--- | :--- |
| `LAZARUS_KEEPER_SECRET` | None (Required) | Stellar secret key (`S...`) used to sign extension and settlement transactions |
| `LAZARUS_INDEXER_API_URL` | `https://api.lazarusprotocol.online` | Base URL of the Lazarus analytical discovery indexer |
| `LAZARUS_RPC_URL` | `https://soroban-testnet.stellar.org` | Soroban RPC server endpoint |
| `LAZARUS_NETWORK_PASSPHRASE` | `Test SDF Network ; September 2015` | Stellar network passphrase (`Public Global Stellar Network...` for mainnet) |
| `LAZARUS_VAULT_CONTRACT_ID` | `CCWAH7...` | Address of the `LazarusVault` contract on Stellar |
| `LAZARUS_BUFFER_FLUSH_INTERVAL`| `600` | Maximum seconds before flushing buffered vouchers to `record_bump_batch` |
| `LAZARUS_DEV_STUB_MODE` | `false` | When `true`, enables offline simulation without broadcasting transactions |

---

## 6. Keeper Profitability Economics

1. **Transaction Costs:**
   - Physical footprint bump (`ExtendFootprintTTLOp` for 100 keys): ~0.00005 XLM.
   - Batch settlement invocation (`record_bump_batch` for 10 vouchers): ~0.0025 XLM.
2. **Gross Bounty Revenues:**
   - 10 vouchers × 0.10 USDC bounty = 1.00 USDC.
3. **Net Margin:**
   - Net revenue per batch: ~0.999 USDC profit, yielding strong operational margins independent of XLM market fluctuations.

