# §7.10 Reserve Swap Threat Model — USDS → LINK (v0.2)

## Context

`Reserve.sol` maintains LINK balance for VRF subscriptions on each chain. Excess USDS in Reserve (from tier rounding, expired unclaimed prizes, sponsorship deposits) is periodically swapped to LINK to top up VRF subs. This introduces a DEX dependency the rest of LGI deliberately avoids. This section enumerates attacks on that swap path and the mitigations.

## Swap path

- **Primary router:** 1inch Aggregation Router v6 (per-chain deployment, immutable)
- **Fallback router:** Uniswap V3 SwapRouter02 (per-chain, immutable)
- **Slippage cap:** 1% per swap (100 bps)
- **Per-event cap:** $500 USD equivalent
- **Per-chain daily cap:** $2,000 USD equivalent, rolling 24h
- **Cooldown:** 6 hours between top-up events per chain
- **Reference price:** Chainlink `USDS/USD` + `LINK/USD` feeds (never pool TWAP)
- **Trigger access:** `onlyKeeper`

## Threats

### T1 — Slippage manipulation (sandwich)

**Attack:** Adversary observes the swap tx in the mempool, front-runs to push pool price unfavorably, back-runs to capture the spread.

**Mitigation:** 1% slippage cap → worst-case per-event loss = $5. $500 event cap + 6h cooldown → worst-case daily extraction from this vector = $20. Slippage check is on-chain via `minAmountOut` parameter.

**Residual:** Acceptable.

### T2 — Reference-price manipulation

**Attack:** If slippage uses pool TWAP as reference, attacker manipulates TWAP via flash-loan swap.

**Mitigation:** Reference price is computed from **Chainlink USDS/USD and LINK/USD oracles**, not pool state. Two independent flash-resistant feeds. `expected_out` derived from Chainlink, `minAmountOut = expected_out * 99 / 100`.

**Residual:** Bounded by Chainlink trust assumptions (same as VRF itself).

### T3 — Approval drain via compromised router

**Attack:** Reserve grants USDS allowance to router. Compromised or maliciously upgraded router drains the allowance.

**Mitigation:** **Exact-amount approval per swap**, never unlimited. `approve(router, swap_amount)` immediately before swap, `approve(router, 0)` immediately after. Router addresses immutable per chain and verified in deployment.

**Residual:** Per-swap exposure only ($500 max). No standing approval.

### T4 — Reentrancy via swap callback

**Attack:** Uniswap V3 invokes `uniswapV3SwapCallback` on the caller. Malicious pool reenters Reserve mid-swap.

**Mitigation:** `nonReentrant` modifier on all state-changing Reserve functions. Callback functions check `msg.sender == ROUTER` and refuse otherwise. Reserve state changes are finalized before external swap call.

**Residual:** Negligible.

### T5 — Aggregator routes through hostile pool

**Attack:** 1inch splits the trade across multiple pools to optimize; one pool is manipulated.

**Mitigation:** The `minAmountOut` check (T1) is enforced on the total output, regardless of route. If 1inch reverts on `minAmountOut` repeatedly (3 strikes), automatic fallback to Uniswap V3 direct path (single pool, USDS → LINK or USDS → WETH → LINK).

**Residual:** Acceptable.

### T6 — Trigger access (gas grief / forced unfavorable swap)

**Attack:** Public `topUpVRF()` lets attacker spam swaps at maximally unfavorable in-tolerance prices.

**Mitigation:** `onlyKeeper` modifier. Cooldown enforced even for keeper. No public path to trigger.

**Residual:** Negligible (keeper is not adversarial).

### T7 — Cross-chain liquidity divergence

**Attack:** USDS/LINK pool depth varies per chain. Thin pools on Polygon or Arbitrum allow disproportionate slippage.

**Mitigation:** Per-chain liquidity check at deployment — verify USDS/LINK pool depth > $10K minimum at chosen router. Where direct USDS/LINK is thin, allow 2-hop via USDC or WETH. Post-MATIC→POL migration on Polygon, confirm POL gas sufficiency for routing.

**Residual:** Discovered at deployment audit, not at runtime.

### T8 — LINK price tail risk

**Attack:** LINK price moves sharply, breaking the USD-denominated swap economics.

**Mitigation:**
- LINK target balance is denominated in **LINK units**, not USD equivalent.
- $500 USD per-event cap means low LINK price → more LINK acquired per event (favorable when needed).
- High LINK price → less LINK per event. Keeper compensates with multiple top-ups subject to cooldown, or Guardian injects LINK manually.

**Residual:** Operational; not exploitable.

## Locked parameters

| Parameter | Value | Notes |
|-----------|-------|-------|
| `MAX_SWAP_USD` | $500 | Per-event cap |
| `MAX_DAILY_SWAP_USD` | $2,000 | Per-chain, rolling 24h |
| `SLIPPAGE_BPS` | 100 | 1% maximum |
| `TOPUP_COOLDOWN` | 6 hours | Per chain |
| `PRIMARY_ROUTER` | 1inch v6 | Per chain, immutable |
| `FALLBACK_ROUTER` | Uniswap V3 SwapRouter02 | Per chain, immutable |
| `PRICE_REFERENCE` | Chainlink USDS/USD + LINK/USD | Never pool TWAP |
| `TRIGGER_ACCESS` | `onlyKeeper` | No public trigger |
| `CIRCUIT_BREAKER_STRIKES` | 3 | Consecutive both-router failures → halt |

## Locked policy decisions

1. **Keeper trust model:** ✅ `onlyKeeper` trigger gated by Guardian `pause()` role. Revisit multisig keeper pre-mainnet if economic exposure increases.
2. **Cross-chain LINK target:** ✅ Chain-specific `LINK_TARGET` constants, set per-chain at deployment.
3. **Dual-router failure mode:** ✅ 3-strike circuit breaker. After 3 consecutive failures across both routers, emit `SwapPathDown(chain)` event, halt further top-ups on that chain, require Guardian manual LINK injection to resume.

## Cross-references

- §4.4 `Reserve.sol` — implementing contract
- §4.3 `DrawManager.sol` — VRF subscription consumer of LINK
- §4.6 Pause clarity — Guardian pause role used here
- §7 parent threat model
- §10 open questions — DEX dependency cost/benefit framing

---
**Status:** v0.2 locked. Ready for §4.4 Solidity implementation.
