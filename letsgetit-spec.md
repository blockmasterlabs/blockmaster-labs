# letsgetit-spec.md — LetsGetIt.app (LGI)

**Status:** v0.2 — locked May 17, 2026 spec rewrite session
**Product:** No-loss prize draw. Deposit stablecoin, principal earns yield in DeFi, accumulated yield randomized into prizes (weekly + daily). Principal always withdrawable, instant, no penalty.
**Tagline:** "You can only win."

---

## 1. Assets (v1)

| Asset | Score | Notes |
|---|---|---|
| USDS (Sky) | 5/5 | Primary. DAI accepted as deposit synonym, auto-converted 1:1 via Sky's official converter. |
| USDC | 2/5 | Allowed, failing checks surfaced in UI. |
| WETH | 5/5 | Just ETH with an interface. |
| WBTC | 2/5 | BitGo custodial bridge. v1.5 migration target = tBTC (Threshold). |
| USDT | — | Permanently out. Never relitigate. |

## 2. Chains (v1)

- **Ethereum L1** — USDS only, sUSDS native yield (Sky Savings Rate)
- **Polygon PoS** — 4 assets, Aave V3 yield. Primary mass-reach. POL is gas token.
- **Arbitrum One** — 4 assets, Aave V3 yield. DeFi-depth tier.

No cross-chain bridge in v1.

## 3. Vault topology

9 separate ERC4626-compatible vault contracts (1 + 4 + 4). Each independently auditable, deployable, pausable.

**ERC4626 note:** interface-compatible but non-canonical. Share price pinned at 1.0 by construction. Yield held in separate accounting variable (`accruedYield`), skimmed by PrizePool at draw time. External integrators expecting drift-on-yield will be wrong — flag in code comments and docs.

## 4. Tier structure — weekly draw (no overflow)

| Tier | Winners | Pot share | Per-winner |
|---|---|---|---|
| 1 | 1 | 10% | 10% |
| 2 | 5 | 25% | 5% |
| 3 | 20 | 25% | 1.25% |
| 4 | 75 | 20% | ~0.27% |
| 5 | 300 | 20% | ~0.067% |

401 winners per weekly draw. No per-payout whale cap (original 5% cap was a math bug — forced 45% to overflow every draw). Sybil deterred at 10 USDS minimum deposit floor + TWAB economics.

## 5. Daily micro-draw

25 winners, equal share, ~30% of weekly projected yield split across 7 days. Stablecoin pools only — yield too thin on volatile pools.

## 6. Yield distribution (per weekly draw)

- 10% protocol fee — 5% Reserve, 5% Treasury
- 90% to prize pot

## 7. Randomness

Chainlink VRF v2.5, per-chain subscription, pre-funded from Reserve. No fallback substitution — honest delay beats dishonest randomness. Subscription auto-tops-up via 1inch (USDS→LINK, 1% slippage cap, $500 max per top-up event, Uniswap V3 fallback).

## 8. Frontend hosting

Dual-channel: Filebase (managed pin) + self-hosted Kubo on dedicated VPS (not same box as Hermes Agent — resource conflict). Hetzner CX22 ($4.50/mo, 4GB RAM, accepts crypto) recommended for Kubo. CAR-file portable across providers.

## 9. Domains

- `letsgetit.crypto` (Unstoppable) — canonical
- `letsgetit.eth` (ENS) — mirror
- `letsgetit.app` — self-hosted HTTPS mirror, emergency only

## 10. Wallets supported

MetaMask, Rabby, Frame, Safe, Trust, OKX, any WalletConnect v2-compatible. Argent removed (rebranded to Ready, Starknet-focused).

## 11. Audit budget

9 vaults × 3,500–5,000 LOC total ≈ $400K–$800K for two top-tier audits. Largest single pre-mainnet capital requirement.

---

## 12. Banned — never in scope

- Coinbase Wallet, Coinbase Onramp, Base chain
- USDT
- Any custodial wallet integration
- Any KYC at protocol layer (belongs at user-elected on-ramp boundary only)
- Any protocol-issued token
- Upgrade-key pattern on core funds-touching contracts — non-upgradeable v1, bugs ship as v2 at new address + migration UI
- Admin key that can pause withdrawals (pause on deposit is fine)
- LLM-mediated transaction signing on production funds

## 13. Security doctrine (funds-touching)

**Signing & deployment:** Safe multisig UI + hardware wallet (Ledger or equivalent) per signer. Never a hot key.

**Treasury:** 3-of-5 Gnosis Safe. Signers named or pseudonymous, publicly listed.

**Reserve:** on-chain whitelisted spend categories with timelock. No discretionary withdrawals.

**Emergency pause:** 2-of-3 separate multisig from Treasury. Pauses deposit only, never withdraw. Auto-expires after 14 days unless re-triggered.

**Keys:** seed phrases/private keys never touch chat, logs, or any tool. Paper or metal backup, geographically separated. No cloud, no photo, no encrypted file on an internet-connected machine.

**Code:** two independent audits before mainnet (min 2 of: Trail of Bits, OpenZeppelin, Spearbit, ChainSecurity), reports published. Pre-audit pyramid: Slither (0 high/medium), Echidna + Medusa (50M+ fuzz cases on invariants), Solhint clean, forge coverage ≥95%, formal verification of TWAB and tier-distribution math via Certora or Halmos. Bug bounty via Immunefi, ≥30 days pre-launch, up to 10% of TVL or $1M whichever is lower. Guarded launch with scaling TVL caps.

**Frontend:** IPFS CID is source of truth, signed CID mirrored across ≥3 channels, SRI hashes on external scripts, verify-CID widget in-app.

---

## 14. 5-Check framework (dependency scoring)

1. Decentralized — no single entity controls the protocol
2. Censorship-resistant — no party can block, freeze, seize funds
3. Permissionless — anyone can use without approval
4. Open-source — code public, auditable
5. Real blockchain problem — solves something centralized can't

Score every external dependency 0–5. Sub-5 allowed only with honest trade named + migration target listed. USDT scores 1/5 — never allowed regardless.
