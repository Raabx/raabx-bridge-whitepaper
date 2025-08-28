# RAABX zk-SPV Bridge (BTC ⇄ cbBTC) — Public Overview

**Version:** v1.0 (Base mainnet)  
**Status:** Contracts deployed privately; **zk-SPV verifier live**; relayer online; UI integration in progress  
**Scope:** Bidirectional bridge for **BTC → cbBTC** (credit on Base) and **cbBTC → BTC** (settle to Bitcoin) with a strict on-chain Gate and a **zero-knowledge SPV verifier**.

> **Public note:** Contract addresses and operational keys are intentionally withheld here until audits and launch communications are complete.

---

## 1) Abstract

This bridge turns a confirmed Bitcoin deposit into **cbBTC** on Base and, in the reverse direction, returns **BTC** when a user burns cbBTC on Base. Security is **true zk-SPV**: the on-chain Gate accepts a deposit only if a **ZkSpvVerifier** registered in the **VerifierHub** returns `true`.

The zk-SPV proof verifies, inside a succinct SNARK (e.g., Groth16/PLONK), that:
- a Bitcoin transaction is **Merkle-included** under a valid block header,
- a connected **header chain** segment satisfies Bitcoin’s **proof-of-work** and **retarget rules** across epochs,
- the head of that segment is at least **k blocks** deeper than the deposit block (**anti-reorg depth**, configurable, e.g., `k=6`),
- public inputs **bind** `(recipient, amount, expected Gate address, Base chainId, nonce/digest)`.

The bridge aligns with the RAABX L2/L3 roadmap (Bitcoin-cadence timing, explicit sealing windows, Taproot anchoring, and zk-KYC lanes). When RAABX L2/L3 are live, receipts anchor to Bitcoin and exits can be verified using covenant scripts once available.

---

## 2) Components 

| Component                | Role (high level)                                                                                                     |
|--------------------------|------------------------------------------------------------------------------------------------------------------------|
| **VerifierHub**          | Registry mapping `bytes32 verifierKey → IVerifier`. Owned by a cold key.                                               |
| **DepositProofGateStrict** (“Gate”) | Loads verifier from Hub; **reverts if missing**; on success **transfers cbBTC** from Gate balance to user.               |
| **ZkSpvVerifier**        | Verifies the SNARK proof for Bitcoin header chain validity + Merkle inclusion; binds `(to, amount, gate, chainId, nonce)`. |
| **Relayer / Prover**     | Observes Bitcoin, assembles witness data, **generates zk-SPV proof**, and submits `(publicInputs, proof)` to the Gate via Hub. |
| **Treasury wallets**     | Pre-fund Gate with cbBTC; maintain BTC reserves for withdrawals (moving toward HTLC/PTLC scripts next).                |

> An **attested-signature fallback** may exist behind a feature flag for ops emergencies, but it is **disabled for production**; the active path is zk-SPV.

---

## 3) Architecture (high-level, zk-SPV)

Bitcoin (headers + tx) zk-SPV proof (SNARK)
│ ▲
│ observe tx, collect headers │ (Merkle inclusion + PoW + retarget + depth)
▼ │
Relayer / Prover ─────────────────────┘
│ submit {to, amount, digest, publicInputs, proof}
▼
VerifierHub ──▶ ZkSpvVerifier (pluggable via verifierKey)
│
▼
DepositProofGateStrict ── transfers cbBTC on success ──▶ cbBTC (Base)

Withdraw (cbBTC → BTC):
User burn/intent on Base ─▶ Relayer validates ─▶ (Now) operator payout BTC
└▶ (Next) HTLC/PTLC script release

markdown
Copy code

**Why the design is future-proof**
- **Pluggable verifier** via `VerifierHub` (rotate to newer circuits without redeploying the Gate).
- **Bound public inputs** prevent replay or redirection (recipient, amount, Gate, chainId, nonce).
- **k-deep anti-reorg** parameter is on-chain policy; raise or lower as required by risk.

---

## 4) Protocol Flows

### A) BTC → cbBTC (credit on Base, zk-SPV)
1. User deposits BTC to a derived address (from our public account xpub/zpub).
2. Relayer tracks headers and builds the inclusion witness.
3. Relayer **proves** in a SNARK that the tx is validly included under a k-deep header chain with correct difficulty.
4. Relayer calls `Gate.verifyAndCredit(to, amountWei, digest, publicInputs, proof)`.
5. Gate asks `VerifierHub.getVerifier(verifierKey)` → `ZkSpvVerifier.verify(...)`.
6. If `true`, Gate transfers cbBTC to `to` and emits `Credited`.

### B) cbBTC → BTC (settle to Bitcoin)
- **Current:** User burns cbBTC (or signs a withdraw intent); relayer validates and pays BTC from reserves.  
- **Next:** **HTLC/PTLC** script path to remove operator trust on withdrawals.

---

## 5) Liquidity, Limits & Concurrency

- UI exposes **available cbBTC at the Gate** and **available BTC reserve**; users cannot request more than available on either side.
- Withdrawals may **queue** if BTC reserve is low (clear messaging + ETA).
- **Per-tx caps** and **rate limits** prevent depletion; **idempotent digest/nonce** prevents double-credit.

---

## 6) Fees (demo settings; subject to change)

- **Network fees:** Passed through 1:1 (BTC miners + Base gas); **no markup**.  
- **Platform fee:** Flat **$1 (USD) in BTC per swap** **during the demo** (may change post-demo).  
  The UI converts $→sats using a reputable price API; if the quote is stale beyond a TTL, the UI blocks submission until refreshed.  
- **Displayed quote =** network fee **+ flat fee** (+ slippage buffer if a DEX hop is needed). Quotes are **binding** for a short TTL and must be refreshed after expiry.

**Production guidance:** Keep a flat fee in a narrow band (e.g., **$0.75–$2.00**), reviewed periodically. Keep it separate from network fees to avoid user confusion.

---

## 7) Security Posture

- **Strict Gate:** Reverts if no verifier is set (eliminates “unset verifier passes”).  
- **Zk-SPV validity:** Proof checks PoW, difficulty retarget rules per Bitcoin consensus, k-deep anti-reorg, and tx inclusion.  
- **Owner rotation** to cold EOA; deployer has **no** privileges.  
- **Single-purpose relayer/prover:** Builds proofs only; not a treasury key.  
- **Observability:** Metrics + append-only receipts per action.

**Notes on SNARKs:**  
- If using **Groth16**, a circuit-specific trusted setup (Powers of Tau + circuit phase) is required; we publish ceremony artifacts before public launch.  
- If using **PLONK** with a universal SRS, a one-time universal setup applies.  
- Proof and verification key hashes are versioned in the repo for auditability.

---

## 8) Roadmap

1. **HTLC/PTLC** withdrawals (script-enforced release).  
2. zk-SPV circuit hardening: multi-epoch proofs, minimal witness size, verifier gas optimizations.  
3. **RAABX integration** (anchors, exits, zk-KYC lanes).  
4. **BTC-only covenant rollup** path when covenant opcodes become available.

---

## 9) Interfaces (sketch)

```solidity
interface IVerifierHub {
  function getVerifier(bytes32 key) external view returns (IVerifier);
}

interface IVerifier {
  function verify(bytes calldata proof, bytes calldata publicInputs) external view returns (bool);
}

interface IGate {
  function verifyAndCredit(
    address to,
    uint256 amountWei,
    bytes32 digest,
    bytes calldata publicInputs,
    bytes calldata proof
  ) external returns (bool);
}

---

## 10) Disclosures
This bridge is production-grade in design and includes a zk-SPV verifier. That said, mainnet operations will roll out with strict caps and progressive limits. Withdrawals may queue under low reserve. Do not store large balances in the Gate. Use at your own risk.

---

## 11) Contact
team@raabx.com
Public updates will be posted after audits and launch readiness checks.

---

## Appendix — Incident Prevention (public)
Prominent “Network must be Base” indicators for all cbBTC actions.

Address checksum and chain-ID guards in UI/API.

On-chain cap checks against Gate balance and BTC reserve telemetry to prevent over-commitment.
