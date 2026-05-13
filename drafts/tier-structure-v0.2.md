# §4.2 PrizePool — Tier Structure (v0.2)

## Allocation

Each draw distributes **100% of the prize pot** across **5 tiers, 401 total winners**:

| Tier | Winners | Pot Share | Per-Winner Share |
|------|---------|-----------|------------------|
| 1    | 1       | 10%       | 10% of pot |
| 2    | 5       | 25%       | 5% of pot each |
| 3    | 20      | 25%       | 1.25% of pot each |
| 4    | 75      | 20%       | ~0.267% of pot each |
| 5    | 300     | 20%       | ~0.067% of pot each |
| **Total** | **401** | **100%** | — |

## Rationale

The v0.1 Tier 1 used a 45% allocation with an *absolute cap*. Above the cap, every draw produced overflow that needed rollover plumbing. v0.2 eliminates absolute caps entirely — every tier is bounded by **percentage**, so allocation always sums to 100% of the pot. No overflow, no rollover logic, no edge cases to audit.

## Computation (Solidity-safe integer math)

```
SHARE_NUM = [10, 25, 25, 20, 20]   // numerators, sum = 100
WINNERS   = [1,  5,  20, 75, 300]  // sum = 401

for each tier i:
  tier_total[i] = (pot * SHARE_NUM[i]) / 100
  per_winner[i] = tier_total[i] / WINNERS[i]
```

Multiplication **before** division to preserve precision. Integer remainders accumulate to **Reserve**, not lost.

## Math verification (pot = $1M USDS, 6 decimals)

| Tier | Per winner | Tier total paid | Remainder |
|------|-----------|-----------------|-----------|
| 1 | $100,000 | $100,000 | 0 |
| 2 | $50,000 | $250,000 | 0 |
| 3 | $12,500 | $250,000 | 0 |
| 4 | $2,666.666666 | $199,999.99995 | $0.00005 |
| 5 | $666.666666 | $199,999.9998 | $0.0002 |
| **Sum** | | **$999,999.99975** | **$0.00025** |

Floor of ~250 base units per draw rolls to Reserve. Trivial.

## Payout model: pull-based

After VRF resolves, contract writes `(winner address → claimable amount)` to a mapping. Winners call `claim()` themselves. 401 push transfers in one tx would blow the block gas limit; pull keeps each draw constant-gas on the contract side.

## Minimum pot threshold (hard requirement)

Tier 5 per-winner = `pot / 1500`. At small pots Tier 5 truncates to 0.

**Locked:** `MIN_POT = $15,000 equivalent per asset`. Below this, the draw is skipped — deposits roll forward, re-attempt next epoch. At $15K, Tier 5 pays ~$10 per winner, covering L2 claim gas with a meaningful prize. Stops degenerate draws paying sub-cent prizes.

## Locked policy decisions

1. **Winner collisions across tiers:** ✅ **Allowed.** A single address may win multiple tiers in the same draw. Rationale: matches PoolTogether convention, simpler VRF expansion, gas-cheaper. The depositor accepted the odds — multi-wins are a feature.

2. **Unclaimed prize timeout:** ✅ **90 days.** After 90 days from draw resolution, unclaimed amounts sweep to Reserve. Prevents indefinite contract liabilities. UI must surface time-to-expiry clearly per claim.

3. **Asset decimal handling:** ✅ **Asset-agnostic at percentage layer.** Decimals (USDS=6, USDC=6, WETH=18, WBTC=8) only matter at the display/UX layer. Tier math operates uniformly on base units. Confirm in §4.1 PrizeVault.

## Dependencies surfaced for §4.3 DrawManager

- VRF expansion: one VRF word → 401 deterministic indices (collision policy = allow duplicates)
- Per-chain VRF subscriptions (Ethereum, Polygon, Arbitrum)
- Trigger gate: `pot >= MIN_POT` checked before VRF request fires

---
**Status:** v0.2 locked. Ready for §4.2 Solidity implementation.
