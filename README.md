# Lazarus Protocol — Technical Documentation

Lazarus Protocol is a decentralized, non-custodial state-rent and proactive TTL-warming keeper network for Stellar (Soroban). It monitors persistent storage keys, contract instances, and executable WASM bytecodes across enrolled Soroban smart contracts and extends their Time-To-Live (TTL) before state archival occurs.

* **Web Console:** [https://lazarusprotocol.online/](https://lazarusprotocol.online/)
* **Indexer API & Telemetry:** [https://api.lazarusprotocol.online/](https://api.lazarusprotocol.online/)
* **GitHub Organization:** [https://github.com/LazarusProtocol](https://github.com/LazarusProtocol)

---

## Documentation Guide

| Document | Description |
| :--- | :--- |
| **[introduction.md](./introduction.md)** | Protocol overview, CAP-0053/0062/0066 state rent lifecycle, 10× restoration penalty, and core solutions. |
| **[get-started.md](./get-started.md)** | Step-by-step onboarding guide for protocol integrators and autonomous keeper node operators. |
| **[keeper.md](./keeper.md)** | Keeper daemon guide, GitHub container packages, deterministic modulo cohort sharding, and economics. |
| **[indexer.md](./indexer.md)** | Analytical storage key indexer, 4-stage hybrid discovery pipeline, Hubble BigQuery, and REST API. |
| **[attestation.md](./attestation.md)** | Decoupled voucher engine, RPC consensus verification, domain separation, and 2-of-3 FROST MPC. |
| **[contracts.md](./contracts.md)** | Soroban Rust implementation walkthrough (`types.rs`, `contract.rs`), circuit breakers, and error codes. |
| **[architecture.md](./architecture.md)** | System topology, protocol invariants, and challenge/solution breakdowns. |

---

## Live Stellar Testnet Deployment

| Parameter | Value |
| :--- | :--- |
| **Network** | Stellar Testnet (`Test SDF Network ; September 2015`) |
| **Soroban RPC** | `https://soroban-testnet.stellar.org` |
| **Vault Contract ID** | `CBKFA2F7RUDGLJNE3K5X2XEXVXRJQ5AMB2NSPREDTATE2ZHDALFKPGUR` |
| **WASM Hash** | `38ad59f3c5ce3f8b6e5e943865b8bfa473c24d116e137bc47b0249c3cc0f757b` |
| **Escrow Token (SAC USDC)** | `CDLZFC3SYJYDZT7K67VZ75HPJVIEUVNIXF47ZG2FB2RMQQVU2HHGCYSC` |
| **Oracle / FROST PubKey** | `197f6b23e16c8532c6abc838facd5ea789be0c76b2920334039bfa8b3d368d61` |
| **Web Portal** | `https://lazarusprotocol.online/` |
| **API Endpoint** | `https://api.lazarusprotocol.online/` |
