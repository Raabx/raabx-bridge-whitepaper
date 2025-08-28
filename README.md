# RAABX zk-Assisted SPV Bridge (BTC ⇄ cbBTC) — Public Overview

**Version:** MVP v0.1 (Base mainnet)  
**Status:** Contracts deployed privately; relayer online; UI integration in progress  
**Scope:** Bidirectional demo bridge for **BTC → cbBTC** (credit on Base) and **cbBTC → BTC** (settle to Bitcoin) with a strict on-chain Gate and a verifier that will be upgraded to full **zk-SPV**.

> **Note:** Contract addresses and operational keys are **withheld** in this public doc until audits and launch comms are complete.

---

## 1) Abstract

This MVP turns a confirmed Bitcoin deposit into **cbBTC** on Base and, in the reverse direction, returns **BTC** when a user burns cbBTC on Base. Security is **zk-assisted SPV**: an on-chain **Gate** accepts a deposit only if a verifier registered in the **VerifierHub** returns `true`. Today the verifier is **signature-attested** (binding BTC header state + tx digest + recipient + amount + expected Gate + chainId). The design cleanly upgrades to a true **zk-SPV** circuit by swapping the verifier in the Hub.

The bridge aligns with the RAABX L2/L3 roadmap (Bitcoin-cadence timing, explicit sealing windows, Taproot anchoring, and zk-KYC lanes). When RAABX L2/L3 are live, receipts anchor to Bitcoin and exits can be verified using covenant scripts once available.

---

## 2) Components (no addresses in public version)

- **VerifierHub** — Registry: `bytes32 verifierKey → IVerifier`. Owner on cold key.  
- **DepositProofGateStrict (“Gate”)** — Loads verifier from Hub; **reverts** if missing; on success transfers cbBTC from Gate balance to user.  
- **Verifier (MVP: SigVerifierAttested)** — Validates an attestation `(to, amountWei, gate, chainId, nonce, btcDigest, …)`. Drop-in replaceable with a **zk-SPV** verifier.

> **Liquidity model:** Gate must be **pre-funded** with cbBTC; BTC withdrawals come from operator-managed reserves (moving to HTLC/PTLC scripts next).

---

## 3) Architecture (high-level)

BTC → Relayer/Attester → VerifierHub → DepositProofGateStrict → cbBTC (Base)
^ | | |
| └— builds proof/attestation └— transfers cbBTC on success
└────────── withdraw: burn on Base; relayer settles BTC

yaml
Copy code

---

## 4) Protocol Flows

### A) BTC → cbBTC
1. User deposits BTC to a derived address.  
2. Relayer tracks headers and inclusion proof.  
3. MVP: Relayer signs an attestation binding recipient, amount, digest, gate, chainId, nonce.  
4. Relayer calls `verifyAndCredit(...)`. Gate loads verifier from Hub and checks it.  
5. On success, Gate transfers **cbBTC** to the user and emits `Credited`.

**Upgrade path:** Replace the attested verifier with a **zk-SPV circuit** that checks header work, difficulty, anti-reorg depth, and Merkle inclusion inside a proof.

### B) cbBTC → BTC
- **MVP (attested):** User burns cbBTC (or signs a withdraw intent); relayer validates and pays out BTC to user’s address from reserves.  
- **Next:** HTLC/PTLC script path to remove operator trust.

---

## 5) Liquidity, Limits & Concurrency

- UI shows **available cbBTC** at the Gate and **available BTC** reserve; users can’t request more than available.  
- Withdrawals may queue if BTC reserve is low (clear messaging + ETA).  
- Per-tx caps and rate limits prevent depletion; idempotent digests/nonce prevent double-credit.

---

## 6) Fees (for demo; subject to change)

- **Network fees:** Passed through 1:1 (BTC and Base gas).  
- **Platform fee:** **Flat \$1 (USD) in BTC** per swap during the demo (may change post-demo). The UI converts \$→sats using a reputable price API; if the quote is stale beyond a TTL, the UI blocks submission until refreshed.  
- **Displayed quote = network fee + flat fee (+ slippage buffer if a DEX hop is needed).** Quotes are binding for a short TTL and must be refreshed after expiry.

**Production guidance:** Keep a flat fee in a narrow band (e.g., **\$0.75–\$2.00**) and revisit periodically; keep it separate from network fees to avoid user confusion.

---

## 7) Security Posture (MVP)

- **Strict Gate:** Reverts if no verifier is set (eliminates “unset verifier passes”).  
- **Owner rotation to cold EOA**; deployer has no privileges.  
- **Single-purpose attester** (not a treasury key).  
- **Observability:** Metrics + append-only receipts for every action.

> **Known limitations:** Verifier is attested (not zk-SPV yet); withdraws are operator-assisted; liquidity must be pre-funded.

---

## 8) Roadmap

1. Swap in **zk-SPV Verifier** (Groth16/PLONK).  
2. **HTLC/PTLC** withdraws.  
3. **RAABX integration** (anchors, exits, zk-KYC lanes).  
4. **BTC-only covenant rollup** path when covenant opcodes become available.

---

## 9) Disclosures
This is an MVP demo. Funds and per-tx caps are limited; withdrawals may be delayed; verifier is signature-based today. Do not store large balances in the Gate. Use at your own risk.

---

## 10) Contact
team@raabx.com

---

## Public updates will be posted after audits and launch readiness checks.

---

