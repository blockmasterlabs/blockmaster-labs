# §10 Open Questions & Status (v0.2)

## Locked decisions log — this session

**Tier structure (§4.2):**
- 5 tiers, 401 winners total: 1 / 5 / 20 / 75 / 300
- Allocation: 10% / 25% / 25% / 20% / 20% of pot
- Winner collisions across tiers: allowed
- Unclaimed timeout: 90 days, sweeps to Reserve
- Asset-agnostic at percentage layer (decimals = display only)
- `MIN_POT = $15,000` equivalent per asset (hard gate)
- Pull-based payout

**DEX swap path (§7.10):**
- Primary: 1inch Aggregation Router v6 (immutable per chain)
- Fallback: Uniswap V3 SwapRouter02
- Slippage cap: 1% (100 bps)
- Per-event: $500 USD equiv
- Per-day per chain: $2,000 USD equiv rolling 24h
- Cooldown: 6 hours per chain
- Reference price: Chainlink USDS/USD + LINK/USD oracles (never pool TWAP)
- Exact-amount approvals (never unlimited)
- Trigger: `onlyKeeper` + Guardian `pause()`
- Circuit breaker: 3 consecutive both-router failures → `SwapPathDown(chain)` event → halt → Guardian manual injection to resume

## Verified — 2026 ground truth

- Fleek hosting **DEAD** Jan 31, 2026 → §3 / §7.1 / §9.4 rewrites required
- USDS is the canonical Sky stablecoin; DAI legacy w/ 1:1 converter (Coinbase delisted DAI April 30; Binance auto-converted)
- The Graph hosted service **DEAD** June 12, 2024 → decentralized network is sole option
- Argent rebranded to Ready, Starknet-focused → removed from wallet list
- Polygon: MATIC→POL migration complete, AggLayer v0.3 live (June 2025), zkEVM deprecated, gas is POL
- Hermes Agent (Nous Research, Feb 2026, MIT) + Paperclip (March 2026, MIT) confirmed cypherpunk-aligned, queued for post-mainnet ops layer

## Blocked / queued for v0.2

- §4.1 `PrizeVault.sol` — multi-asset generalization + non-canonical ERC4626 explanation **(next blocker)**
- §4.3 `DrawManager.sol` — per-chain VRF subs, depends on §4.2 ✅
- §4.4 `Reserve.sol` — DEX dependency wired up, depends on §7.10 ✅
- §4.6 Pause clarity — Guardian role definition, used in §7.10
- §5 Sybil floor — per-asset recalibration
- §8 webhook URL policy — explicit "no webhook URL configured" stance
- §9 migration cleanup — Fleek out, Graph hosted out, tBTC migration target in, Gnosis chain question
- §11 v1 scope boundary — multi-asset, multi-chain decision already drafted, paste-in pending
- §12 pre-audit checklist — Slither / Echidna / Medusa / forge coverage gates
- §13 per-asset voice copy — USDS / USDC / WETH / WBTC vault strings

## Strategic open questions — DECISION REQUIRED

### Q1 — Audit budget: the existential question

**Numbers:**
- 9 vault contracts × multi-chain ≈ 3,500–5,000 LOC custom
- Two top-tier audits (e.g. Trail of Bits + Spearbit, OZ + ChainSecurity): **$400K–$800K combined**
- Current revenue path: $150 funnel sessions (Joe Kimbrough, Dennis Sennett, Toby)
- Math at current cadence: 3,000–5,000 sessions to fund both audits → ~30+ months

**Paths:**
- **A — Funnel-only:** ~30+ months. Too slow; market moves past the window.
- **B — Pre-seed raise:** Target $500K–$1M; covers audit + 18-month runway. Trades equity / control for speed.
- **C — Aggressive LOC reduction:** Lean hard on OZ contracts, Aave V3 interfaces, ERC4626 reference implementations. Target <2,000 LOC custom code. Drops audit to $200K–$400K.
- **D — Staged launch:** v1.0 = USDS-only, Ethereum L1 only. One audit covers that surface ($150K–$300K). Multi-asset + multi-chain expansion in v1.1 / v1.5, funded by mainnet revenue.

**Working recommendation (needs explicit lock):** C + D combined. Cut LOC ruthlessly; ship narrow first; let mainnet revenue fund the expansion audits. Trades scope for survivability.

### Q2 — tBTC migration target (v1.5)

WBTC has BitGo wrapper risk. tBTC is bridge-native, threshold-signed, decentralization-aligned. **Question:** migrate WBTC → tBTC in v1.5? **Pre-condition:** tBTC L2 liquidity reaches USDC parity.

### Q3 — Gnosis Chain inclusion

Gnosis has native USDS via Sky expansion, cheap gas, EVM-compatible. **Question:** add to v1.0 chain list (4th chain) or defer to v1.5? **Recommendation lean:** defer to v1.5, keep v1.0 audit surface bounded.

### Q4 — VRF cost projection

Per-draw VRF: ~0.25 LINK Ethereum, ~0.0005 LINK Polygon, ~0.005 LINK Arbitrum (verify at deployment). At 1 draw/week × 3 chains × 4 assets = 156 draws/year. **Question:** at what TVL does draw revenue cover VRF + keeper gas + LINK top-ups? Needs spreadsheet.

### Q5 — Hermes Agent integration timing

Hostinger $5 VPS evaluated. Hermes can run 24/7 ops: monitoring draws, alerting on circuit breaker, posting metrics. **Question:** pre-mainnet or post-mainnet integration? **Recommendation lean:** post-mainnet — current bandwidth goes to shipping.

### Q6 — Paperclip orchestration

Paperclip wraps Hermes (and others) as a "zero-human company." Org charts, budgets, governance for agent teams. **Question:** consider for v2 ops stack, not v1.

## Risks tracking

| Risk | Severity | Likelihood | Mitigation |
|------|----------|------------|-------------|
| Audit budget exceeds capital | **HIGH** | **HIGH** | LOC reduction + staged launch (Q1) |
| Chainlink VRF cost spike | MED | MED | LINK Reserve buffer + Q4 projection |
| Aave V3 protocol risk | MED | LOW | Per-asset yield diversification post-mainnet |
| Sky/USDS protocol risk | MED | LOW | Multi-asset design isolates per-asset failures |
| DEX liquidity collapse | LOW | LOW | §7.10 circuit breaker + Guardian injection |
| Keeper key compromise | MED | LOW | `onlyKeeper` + Guardian `pause()`; revisit multisig pre-mainnet |
| Mobile dev session continuity (this session) | LOW | n/a | GitHub-as-bridge + Termux. Validated tonight. |

## Cross-references

- §2 Five-Check Scorecard — capital path framing for Q1
- §4 contract architecture — implementation blockers above
- §11 v1 scope boundary — what's in/out of audit surface
- §12 pre-audit checklist — Slither / Echidna / Medusa / forge gates before any auditor touches code

---

**Status:** v0.2 session #1 closed. Two pieces locked and pushed (§4.2, §7.10). Next session priority: Q1 capital path decision + §4.1 PrizeVault rewrite.
