# Triple C — Documentation

This folder contains documentation you can use to describe the project (e.g. for a video or onboarding).

| File | Purpose |
|------|--------|
| **FULL_PROJECT_SPEC.md** | **Master spec:** monorepo layout (Frontend, Backend, SC, deploy), networks & env, all smart-contract roles including CCC Hub / LoyaltyLevelVault swap plumbing, economics tables, CCC Hub UI (§5), backend API & scripts, Hardhat npm scripts, links to deeper docs. |
| **CCC_HUB_SPEC.md** | **CCC Hub deep dive:** points→CCC, CCC→USDT, staking vs rewards, **no user unstake**, treasury funding (CCC on platform + USDT on SC3), operator checklist, links to ABI/deploy notes. |
| **MAIN_FLOW_AND_CASHFLOW.md** | Main user and system flow (incl. **CCC Hub step** §1.1); cash flow (where USDT goes) for all card tiers on mint; CLC caps; first-mint overlap. |
| **SMART_CONTRACTS_AND_PAYMENTS.md** | When each contract receives/sends USDT; CCC Hub role (staking, no unstake). |
| **CARDS_CASHBACK_SPEC.md** | Cards Cashback (referral) spec: conditions 1–4, amounts per tier, full amount to referrer’s card queue on Master (no SC5 split on cashback), and how referrers qualify. |
| **CARDS_POINTS.md** | Card Points vs Loyalty Points (SC3); **CCC Hub burn order** at end (`debitTotalPoints`). |
| **GIFT_CARD_AND_RAFFLE_COUPON.md** | Gift card: eligibility, sending, CLC1/CLC2 flow, $2000 when both conditions met, admin flow, deferred payout when reserve low. Raffle coupon: 1 per 1000 points, serial = wallet, when created, where shown. |
| **VIDEO_SCRIPT.md** | Walkthrough narration: mint/CLC/SC splits (correct SC4 cashback), optional CCC Hub line, outro. |
| **FUNCTION_REFERENCE.md** | List of all contract and frontend functions with short descriptions. |
| **SYSTEM_DIAGRAM.md** | Mermaid diagrams: architecture, cash flow on mint, CLC state, frontend sections, contract roles. |

Render Mermaid in GitHub, VS Code (Markdown Preview), or [mermaid.live](https://mermaid.live).
