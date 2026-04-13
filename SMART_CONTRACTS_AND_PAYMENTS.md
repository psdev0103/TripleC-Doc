# Smart Contracts — When Each Receives Payment

This document describes each smart contract (SC) in the TripleC system and **when it receives USDT payments**. For the full cash flow by card (CLC1 and CLC2) and tier amounts, see [MAIN_FLOW_AND_CASHFLOW.md](MAIN_FLOW_AND_CASHFLOW.md) (especially §2.0 Cashflow summary by card).

---

## Overview

| Contract | Role | Receives from |
|----------|------|----------------|
| **CustomNFT (Master)** | Holds reserve; distributes queue; pays out CLC; sends to SC1, SC1b, SC2, SC3, SC4, SC5, Gift Card SC | User (mint price) |
| **Gift Card SC (GiftCardReceiver)** | Gift card CLC1/CLC2 cap payouts | Master |
| **SC1 OverlapReceiver** | Queue remainder | Master |
| **SC1b OverlapReceiver2** | CLC2 fixed amount per tier | Master |
| **SC2 DeveloperReceiver** | Developer/team (CLC1 + CLC2) | Master |
| **SC3 LoyaltyLevelVault** | Loyalty & Level (CLC1); CLC2 $0.50 | Master |
| **SC4 ReferralFeeHandler** | Cards Cashback | Master |
| **SC5 FivePercentReceiver** | 5% of user payouts | Master, SC4 |

---

## Gift Card SC — GiftCardReceiver

**Role:** Receives USDT when a **gift card** reaches CLC1 cap or CLC2 cap. No user payout for gift cards; those amounts go to this contract instead.

**When Gift Card SC receives payment:**

1. **Gift CLC1 cap reached** — **$1000** (when the gift card’s reward balance reaches the CLC1 cap of $2500). At the same time, Master sends $126 to SC3, $374 to SC1, and uses $1000 to auto-mint the gift CLC2 card.
2. **Gift CLC2 cap reached** — **$1000** (when the gift CLC2 card’s reward balance reaches the CLC2 cap of $1000). No payout to the user.

**When Gift Card SC sends $2000:**

$2000 is sent to the gift card user when **both** conditions are met (once per user):

1. **Loyalty users minted 3 Diamond cards** — Users who have the gift card user as referrer have minted 3 Diamond (CLC1) cards in total (verified off-chain by admin; see `GiftCardReceiver.sol` NatSpec).
2. **Gift CLC2 card reached max cap** — The user’s gift CLC2 card has reached its max cap (Master calls `onGiftCLC2CapReached(beneficiary)` to record this).

When both are true, an admin calls `payoutBothConditionsMet(giftCardUser)` to send **$2000 USDT** to that user.

The owner of the Gift Card SC contract can withdraw any remaining USDT via `withdrawToken`.

---

## SC1 — OverlapReceiver

**Role:** Receives all **remainder USDT** from the main card flow in queue (the “overlap”).

**When SC1 receives payment:**

1. **First mint in the system**  
   When the very first card is minted, there are no previous cards to receive the queue flow. The first-mint overlap is sent to SC1.
   - Bronze: $5  
   - Platinum: $50 (5×$10)  
   - Emerald: $250 (5×$50)  
   - Diamond: $500 (5×$100)

2. **Later mints (CLC1 or CLC2) — unallocated queue**  
   When there are not enough previous cards to absorb the full queue amount, the unallocated part is sent to SC1.  
   Example: Platinum mint, queue $50; only 3 previous cards eligible at $10 each → $30 distributed, **$20 to SC1**.

3. **Overflow from queue distribution**  
   When the contract tries to add reward to a card that is already at cap (or would exceed cap), that amount is “overflow” and is sent to SC1.  
   Applies on both **CLC1 paid mints** and **CLC2 auto-mints** (when a CLC2 card is generated, any overflow from distributing its queue also goes to SC1).

**Summary:** SC1 receives payment whenever queue USDT is **not** fully applied to previous cards: no cards in queue, or remainder/overflow after distribution.

---

## SC1b — OverlapReceiver2

**Role:** Receives a **fixed amount** when a **CLC2 card** is auto-generated. Used only for CLC2; CLC1 queue remainder goes to SC1.

**When SC1b receives payment:**

- **Only when a CLC2 card is generated** (a CLC1 card reaches cap and the system auto-mints a CLC2 card for that owner).  
- SC1b receives a **fixed amount per tier** (not a percentage of queue). Queue remainder (unallocated + overflow) goes to SC1, not SC1b.

| Tier    | SC1b receives (CLC2) |
|---------|------------------------|
| Bronze  | $2.25                  |
| Platinum | $22.50                |
| Emerald | $112.50                |
| Diamond | $225                   |

SC1b does **not** receive: first-mint overlap, unallocated remainder, or overflow (those go to SC1).

---

## SC2 — DeveloperReceiver

**Role:** Developer/team profit on CLC1 paid mints and on CLC2 card generation.

**When SC2 receives payment:**

1. **On every CLC1 paid mint** (user mints a card with USDT) — fixed amount per tier.  
2. **When a CLC2 card is generated** — same fixed amount per tier as CLC1.

| Tier     | SC2 receives (USDT) |
|----------|----------------------|
| Bronze   | $2.25                |
| Platinum | $27                  |
| Emerald  | $137                 |
| Diamond  | $274.5               |

Owner mints (no payment) do **not** send anything to SC2.

---

## SC3 — LoyaltyLevelVault

**Role:** Loyalty & Level points funding; POINTS share when a CLC2 card is generated.

**When SC3 receives payment:**

1. **On every paid mint (CLC1)**  
   Fixed amount per tier, sent at mint time. SC3 also credits loyalty and level points to the minter and referrer chain.

   | Tier     | SC3 receives (USDT) |
   |----------|----------------------|
   | Bronze   | $1.75                |
   | Platinum | $13                  |
   | Emerald  | $63                  |
   | Diamond  | $125.5               |

2. **When a CLC2 card is generated**  
   SC3 receives **$0.50** (all tiers).

On CLC2 **auto-mint**, no loyalty/card points are credited (card points are from CLC1 mints only); USDT to SC3 is the fixed **$0.50** per CLC2 generation.

---

## SC4 — ReferralFeeHandler (Cards Cashback)

**Role:** Holds and pays out **Cards Cashback** (referral amount per mint).

**When SC4 receives payment:**

- **On every paid mint** (CLC1).  
- SC4 receives the tier’s full **referral amount** (max cashback) in USDT. Amount depends on tier:

| Tier     | SC4 receives per mint (USDT) |
|----------|------------------------------|
| Bronze   | $1                           |
| Platinum | up to $10 (depends on referrer condition) |
| Emerald  | up to $50                    |
| Diamond  | up to $100                   |

When a referrer exists and is qualified, SC4 **pays out** part of that balance: **5% to SC5**; **95%** is transferred to **Master** and credited onto the **referrer’s CLC1 or CLC2 cards in queue** (same accrual rules as mint queue flow). If the referrer has **no CLC1 or CLC2 card**, the **95% stays in SC4**. The remainder of what SC4 received (above the condition-based payout) stays in SC4 until used. If there is no referrer, the full amount stays in SC4.

So: **SC4 receives** on every paid mint; **SC5 and Master (referrer queue)** receive only when SC4 runs payout (referrer exists and condition-based amount > 0).

---

## SC5 — FivePercentReceiver (5% SC)

**Role:** Receives the **5%** portion of payments that are otherwise sent to users (CLC payouts and cashback payouts).

**When SC5 receives payment:**

1. **From Master — CLC payout**  
   When a card (CLC1 or CLC2) reaches cap and the contract pays the owner, the split is 95% to owner, 5% to SC5. So SC5 receives **5% of each CLC payout** (wallet payout when cap is reached).

2. **From SC4 — Cashback payout**  
   When SC4 runs Cards Cashback for a referrer, it sends **5% to SC5** and credits **95%** onto the referrer’s CLC1/CLC2 cards on Master (or keeps the 95% in SC4 if the referrer has no such card). So SC5 receives 5% of each cashback payout.

If SC5 is not set, Master sends the 5% CLC share to SC2 (DeveloperReceiver) instead.

---

## CustomNFT (Master)

**Role:** Not a “receiver” in the same sense — it **collects** USDT from the user on mint and **sends** to SC1, SC1b, SC2, SC3, SC4, and (for CLC payouts) SC5. It holds the queue reserve and distributes to previous cards; when a card reaches cap it pays the owner (95%) and SC5 (5%), and may auto-mint CLC2.

**When Master receives USDT:**

- On every **paid mint**: the full tier price (e.g. $10 / $100 / $500 / $1000) from the user.  
  It then immediately splits that to SC2, SC3, SC4, and (via reserve/queue) to previous cards, SC1, or SC1b/SC3 when CLC2 is generated.

---

## Quick reference — payment triggers

| SC  | Receives when |
|-----|----------------|
| SC1 | First mint (overlap); any mint where queue is not fully absorbed (unallocated + overflow). CLC1 and CLC2. |
| SC1b | Only when a CLC2 card is auto-minted; fixed amount per tier (e.g. Bronze $2.25, Diamond $225). |
| SC2 | Every paid mint; fixed amount per tier. |
| SC3 | Every paid mint (Loyalty & Level); plus when CLC2 is generated (5% of queue). |
| SC4 | Every paid mint (full referral amount); then pays referrer + SC5 when applicable. |
| SC5 | When Master pays a card owner at cap (5% of payout); when SC4 pays a referrer (5% of cashback). |
| Gift Card SC | Receives: CLC1 cap $1000, CLC2 cap $1000. Sends: $2000 to gift card user when both conditions met (3 Diamond by referrals + gift CLC2 cap reached; condition 2 on-chain, condition 1 verified off-chain before `payoutBothConditionsMet`). |