# GMX v1 Smart Contract Review
**Vault.sol + PositionRouter.sol + BasePositionManager.sol — Economic Surface Audit**

**Date:** 2026-09-08
**Auditor:** Flux (qwen3.8 local) + Nimbus (Hermes) — joint deep read
**Scope:** 154 .sol files in `/home/shared/bounties/GMX/CODE/` (GMX v1, Solidity 0.6.12, USDG-based accounting)
**Methodology:** Linear manual trace — Vault lifecycle, PositionRouter queue/execution, BasePositionManager fee/state, cross-referenced against PositionUtils.sol math layer.

---

## Executive Summary

The GMX v1 protocol is **well-architected** with clean state management and sound position lifecycle design. **No CRITICAL or HIGH findings.**

7 findings across Vault.sol, PositionRouter.sol, and BasePositionManager.sol. The highest-signal findings are structural economic risks — queue stalling (GMX-006), CEI violations in callbacks (GMX-008), dual fee-reserve over-withdrawal risk (GMX-009), and DoS via strict price validation (GMX-010). The architecture's USDG dependency is a structural assumption — any USDG-side vulnerability cascades directly into Vault solvency.

**Finding Distribution:**
- MEDIUM: 4 (GMX-006, 008, 009, 010)
- LOW-MEDIUM: 2 (GMX-002, 003)
- LOW: 1 (GMX-005)

---

## Findings

### GMX-002: First-Depositor Pricing Advantage
- **Severity:** LOW-MEDIUM
- **Location:** Vault.sol `buyUSDG()` path
- **Analysis:** `getMinPrice()` uses a stale oracle snapshot with an 8-hour funding interval gap. Early deposits capture unfilled book spread — a structural economic edge for first depositors in each funding window.
- **Impact:** Not a bug, but an economic asymmetry. Worth documenting for transparency.
- **Mitigation:** None required. Informational.

### GMX-003: `cumulativeFundingRates` Overflow Potential
- **Severity:** LOW-MEDIUM
- **Location:** Vault.sol L618
- **Analysis:** Cumulative multiplication of funding rates over years-long open positions with high `fundingRateFactor` approaches `uint256.max` without a rollover check. In practice, funding rates reset periodically, but the math layer has no explicit cap.
- **Impact:** Theoretical overflow for extremely long-lived positions. Edge case.
- **Mitigation:** Add a rollover check or cap on `cumulativeFundingRates` accumulation.

### GMX-005: `inManagerMode` Fee Accounting Misalignment
- **Severity:** LOW
- **Location:** Vault.sol L710+
- **Analysis:** `inManagerMode` gates `buyUSDG` but `_collectSwapFees()` still executes during the mode transition window. Fee accounting is misaligned — fees are collected but the mode state does not reflect the swap.
- **Impact:** Minor fee accounting drift on mode transitions.
- **Mitigation:** Gate `_collectSwapFees()` behind the same `inManagerMode` check, or document the intended behavior.

### GMX-006: Queue Stalling — `executeIncreasePositions()` Breaks on First Invalid Request
- **Severity:** MEDIUM
- **Location:** PositionRouter.sol
- **Analysis:** The `executeIncreasePositions()` keeper loop breaks on the first invalid request with no skip/retry. A single malformed or expired request in the queue stalls the entire keeper batch. This creates a state-dependent liveness failure — one bad position blocks all subsequent queued executions indefinitely.
- **Impact:** DoS on the keeper execution path. Positions queued after the bad request are delayed indefinitely.
- **Mitigation:** Skip invalid requests instead of breaking. Log and continue. Add per-request TTL.

### GMX-008: `_callRequestCallback()` — CEI Violation
- **Severity:** MEDIUM
- **Location:** PositionRouter.sol
- **Analysis:** The callback fires AFTER internal state deletion but BEFORE the external transfer completes. The untrusted `_callbackTarget` can inspect pre-transfer state (position still exists, funds not yet moved) and re-enter via other paths. Standard "Checks-Effects-Interactions" violation pattern.
- **Impact:** Callback target can exploit the window between state mutation and fund transfer. Re-entry could double-spend or corrupt position state.
- **Mitigation:** Move the callback to AFTER the external transfer (final effect). Or use a pull-payment pattern.

### GMX-009: Dual Fee Reserves — Double-Draw Risk
- **Severity:** MEDIUM
- **Location:** BasePositionManager.sol + Vault.sol (both maintain `feeReserves[]`)
- **Analysis:** BasePositionManager and Vault each maintain independent `feeReserves[]` arrays, both withdrawable by admin via separate `onlyAdmin` functions with no coordination. If the admin extracts fees from both pools, the total withdrawn may exceed the actual fees collected — a direct economic extraction vector.
- **Impact:** Potential over-withdrawal by admin. Trust-dependent but economically extractable.
- **Mitigation:** Consolidate fee tracking into a single source of truth. Or add a cumulative withdrawal cap with cross-pool reconciliation.

### GMX-010: Overly Strict Price Validation Blocks Legitimate Positions
- **Severity:** MEDIUM
- **Location:** BasePositionManager.sol + PositionRouter.sol
- **Analysis:** `require(markPrice <= _price)` rejects valid position-close requests where partial fills mean execution price differs from the requested limit. Combined with keeper batching, a position queued with a limit price may become impossible to fill even when the market has moved favorably enough. This is a false-negative on legitimate closes.
- **Impact:** DoS on the position-close path for legitimate traders. Market-moving positions can be permanently locked if price validation is too strict relative to slippage tolerance.
- **Mitigation:** Use `require(markPrice <= _price * (1 + slippageBps))` to allow bounded deviation. Or implement a graceful degradation path where keepers retry with adjusted prices.

---

## Architecture Notes

1. **Queue-based execution:** Keeper gets `minBlockDelayKeeper` (block count) while public faces `minTimeDelayPublic` (timestamp). Asymmetric latency gives keepers an effective priority lane. This is a design choice, not a bug, but it creates a MEV incentive structure.

2. **ETH→WETH fallback pattern:** `_transferOutETHWithGasLimitFallbackToWeth()` tries ETH transfer at fixed gas limit; if that fails, wraps to WETH and does ERC20 transfer. Clever UX but means the receiver might unexpectedly receive wrapped tokens. Worth documenting.

3. **USDG dependency:** The entire liquidity model depends on USDG being fully backed. Any vulnerability in USDG (not in scope here) cascades directly into Vault position solvency. This is a structural assumption, not a finding.

4. **Stateless request queue:** The `requestQueue` stores account/index pairs, not full position state. The request is reconstructed from Vault state at execution time. This is clean but means the request is only as valid as the Vault state at execution — hence GMX-006's stalling risk.

---

## Recommendation

**Priority fixes:**
1. **GMX-006 (queue stalling):** Change `executeIncreasePositions()` from break-on-invalid to skip-and-continue. Most exploitable vector.
2. **GMX-009 (double fee draw):** Consolidate fee tracking or add a cumulative withdrawal cap. Direct economic extraction.
3. **GMX-008 (CEI violation):** Reorder `_callRequestCallback()` to fire AFTER the external transfer. Re-entry risk.
4. **GMX-010 (price validation):** Use bounded slippage (`markPrice <= price * (1 + slippage)`) to avoid blocking legitimate closes.

**Overall:** The GMX v1 core is well-architected. The findings are structural economic risks and liveness failures. The protocol's soundness depends on the USDG backing assumption — which is outside the scope of this audit but is the single largest structural risk.

---

*Report generated by Flux (qwen3.8) + Nimbus (Hermes). Joint deep read, 2026-09-08.*
*7 findings: 4 MEDIUM, 2 LOW-MEDIUM, 1 LOW. No CRITICAL, no HIGH.*
