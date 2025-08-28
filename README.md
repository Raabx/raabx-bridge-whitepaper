# RAABX zk-SPV Bridge (BTC ⇄ cbBTC)

**Version:** v1.0 (Production Demo)  
**Network:** Base mainnet (cbBTC) + Bitcoin mainnet  
**Status:** zk-SPV verifier live • Strict Gate live • HTLC/PTLC withdrawals live • Relayer/Prover live • UI integration live

> **Public note:** Contract addresses, keys, and operational endpoints are intentionally withheld in this public README until audits and launch communications are complete.

---

## 1) Executive Summary

The RAABX Bridge enables **trust-minimized value transfer** between **Bitcoin (BTC)** and **cbBTC on Base**. Deposits from Bitcoin are materialized on Base only when a **zero-knowledge SPV (zk-SPV)** verifier proves on-chain that the BTC transaction is included in a valid, sufficiently deep Bitcoin header chain. Withdrawals from Base to Bitcoin use **hash-time locked contracts (HTLC/PTLC)** for script-enforced release on Bitcoin.

**Key properties**
- **True zk-SPV:** Proof checks Bitcoin **PoW**, **retarget rules**, **Merkle inclusion**, and **k-deep** anti-reorg depth — all inside a succinct SNARK.
- **Strict Gate:** The Gate only credits when an authorized verifier (via **VerifierHub**) returns `true`.
- **Script-enforced withdrawals:** HTLC/PTLC guarantees Bitcoin payouts strictly according to the on-chain intent on Base.
- **Production liquidity controls:** Live per-side reserves, per-tx caps, queueing, and back-pressure are enforced end-to-end.

---

## 2) Architecture (high-level)

Bitcoin (headers + tx) zk-SPV proof (SNARK)

│ ▲

│ observe tx, gather headers │ (Merkle inclusion + PoW + retarget + depth k)

▼ │

Relayer / Prover ─────────────────────┘

│ submit {to, amount, digest, publicInputs, proof}

▼

VerifierHub ──▶ ZkSpvVerifier (selected via verifierKey)

│

▼

DepositProofGateStrict ── on success ──▶ cbBTC transfer (Base)

Withdraw (cbBTC → BTC)
User burn/intent on Base ─▶ HTLC/PTLC details ─▶ Bitcoin output locked
User reveals preimage (or timeout path) ─▶ BTC release per script

---

**Upgradability:** Verifier is **pluggable** via `VerifierHub`. New circuit versions can be rolled without redeploying the Gate.

---

## 3) Components

| Component                  | Purpose                                                                                                      |
|---------------------------|--------------------------------------------------------------------------------------------------------------|
| **VerifierHub**           | Registry `bytes32 verifierKey → IVerifier`.                                                   |
| **DepositProofGateStrict**| Stateless Gate. Loads verifier from Hub, **reverts if none**, and transfers cbBTC upon successful verify.     |
| **ZkSpvVerifier**         | On-chain verifier for SNARK proof validating Bitcoin header chain and tx inclusion with k-deep policy.        |
| **Relayer / Prover**      | Observes Bitcoin, assembles witness, **generates zk-SPV proof**, submits to Gate via Hub.                     |
| **Withdrawal scripts**    | **HTLC/PTLC** on Bitcoin for non-custodial, script-enforced release according to Base-side burn/intent.       |
| **Liquidity manager**     | Enforces reserves, caps, rate limits, queueing, and SLOs for both directions.                                 |

---

## 4) Protocols

### A) BTC → cbBTC (credit on Base, zk-SPV)
1. User sends BTC to a unique address derived from the public account (zpub/xpub).
2. Relayer tracks headers and constructs the witness (tx merkle path, header chain segment).
3. Prover produces a **SNARK** attesting:  
   - tx is included under a valid header,  
   - header chain segment satisfies Bitcoin **PoW** and **retarget**,  
   - the segment head is ≥ **k** blocks past the deposit block (anti-reorg),  
   - public inputs bind `(recipient, amount, Gate, Base chainId, nonce/digest)`.
4. Relayer calls `Gate.verifyAndCredit(to, amountWei, digest, publicInputs, proof)`.
5. Gate queries `VerifierHub.getVerifier(verifierKey)` and invokes `verify(...)`.
6. If `true`, Gate transfers cbBTC to `to` and emits `Credited`.

### B) cbBTC → BTC (script-enforced withdrawal)
1. User signs a **withdraw intent** and burns cbBTC (or escrow-locks) on Base.
2. Relayer posts a **Bitcoin HTLC/PTLC** output to the user’s BTC address (hash-lock + CLTV time-lock).
3. User (or agent) publishes the **preimage** to claim BTC before timeout; otherwise the timeout path refunds according to policy.
4. UI tracks on-chain status; proofs/receipts are persisted.

---

## 5) Liquidity & Limits (implemented)

- **Live reserves on both sides.**  
  - **Base side:** Gate maintains a cbBTC reserve.  
  - **Bitcoin side:** Managed UTXO set for HTLC/PTLC payouts.
- **Caps & rate limits:** Enforced by API and contracts to prevent over-commitment.
- **Fair queueing:** If reserves are insufficient, requests enter an ordered queue with ETA and cancel/refresh options.
- **UI truth:** UI shows **available liquidity** and **maximum per-tx** in real time; it blocks attempts > available.

---

## 6) Fees & Quotes (implemented)

- **Network fees:** Passed through exactly (BTC miners + Base gas). No markup.
- **Platform fee:** Flat **$1 (USD) in BTC per swap** for the public demo. Configurable at runtime; audited changes are announced.
- **Quotes:**  
  - `displayed_total = network_fee + flat_fee (+ slippage buffer if a DEX hop is required)`  
  - Quotes are **binding** for a short TTL. UI auto-refreshes if expired.  
  - Price API feeds are monitored with “stale” badges and hard TTL cutoffs.

**Production band guidance:** Keep the flat fee between **$0.75 – $2.00** depending on load and operations cost; never blend platform fee into network fees.

---

## 7) Security & Correctness

- **Strict Gate:** No verifier → **revert**. Eliminates “unset verifier passes.”
- **zk-SPV soundness:**  
  - Validates PoW and **difficulty retarget** across epochs,  
  - Enforces **k-deep** confirmations,  
  - Ensures **Merkle inclusion** of the deposit tx,  
  - Binds `(to, amount, Gate, chainId, nonce)` as public inputs to prevent redirection/replay.
- **HTLC/PTLC withdrawals:** Script-enforced; no operator discretion on release.
- **Key hygiene:**  
  - Hub/Gate ownership on **cold key**; deployer has no privileges.  
  - Relayer/Prover key is proof-only; cannot move reserves.  
- **Observability:** Metrics, structured logs, append-only receipts, anomaly alerts.

**SNARK setup notes**
- **Groth16**: circuit-specific Phase 2 CRS; ceremony artifacts are versioned and published.  
- **PLONK**: universal SRS option; verification key hash pinned on-chain/off-chain for auditability.

---

## 8) Operations (implemented runbook)

- **Liquidity targets:**  
  - Maintain Gate cbBTC ≥ target X; alert if below.  
  - Maintain BTC UTXO reserve ≥ target Y; auto-sweep change; consolidate when fees are low.
- **Rotations & upgrades:**  
  - Rotate verifier by updating `VerifierHub.setVerifier(key, newVerifier)` under the cold owner.  
  - Post verification key hash and circuit version to the public registry.
- **Emergency controls:**  
  - Pause credits by setting verifier to `0x0` (Strict Gate reverts).  
  - HTLC/PTLC timeouts guarantee safe refund paths on Bitcoin.

---

## 9) API (public surface)

- `GET /liquidity` → `{ cbBTC_available, BTC_available, max_per_tx, queue_depth }`  
- `POST /quote/deposit` → `{ deposit_addr, min_confs, fees: {network, flat}, ttl }`  
- `POST /quote/withdraw` → `{ htlc_terms, fees: {network, flat}, ttl }`  
- `POST /submit/deposit-proof` → internal; relayer/prover submits `(publicInputs, proof)`  
- `POST /withdraw/intent` → user intent & destination BTC address; returns HTLC/PTLC details and status handle

All responses include **chain/network badges**, **TTL countdowns**, and **idempotency keys**.

---

## 10) UI Contract (implemented)

- **Base-only guard** for cbBTC actions (hard chainId check).
- Real-time **liquidity** and **max per-tx** display.
- **Quote TTL** countdown with auto-refresh.
- Clear **fee line items**: network vs. flat.
- **Queue UX** with ETA, cancel, and refresh.

---

## 11) Monitoring & SLOs

- **KPIs:** proof generation time, verifier gas cost, header lag, queue wait time, fulfillment rate, failure rate.
- **Alerts:** stale headers, proof verifier reverts, reserve under-target, price feed stale, queue > threshold.

---

## 12) Upgradability & Governance

- Verifiers are hot-swappable via **VerifierHub**; Gate remains immutable.
- Circuit, CRS, verification key, and version metadata are published with change logs.
- Config knobs (k-deep, fee band, caps) are managed by multisig/cold owner, with timelock announcements for changes.

---

## 13) Privacy & Compliance

- No PII collected; BTC addresses and Base addresses are treated as pseudonymous identifiers.
- Optional zk-KYC lanes can be introduced without changing the Gate/Verifier interfaces.
- Append-only receipts simplify external audits.

---

## 14) Disclosures

This is a **production-grade demo** with caps and strict policies.  
Do not treat Gate reserves as long-term custody. Use at your own risk.

---

