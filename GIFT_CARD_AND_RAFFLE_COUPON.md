# Gift Card & Raffle Coupon

This document describes the **Gift Card** flow (admin-sent card, $2000 payout when both conditions are met) and the **Raffle Coupon** system (1 coupon per 1,000 points; serial = wallet).

---

## Part 1 — Gift Card

### 1.1 Overview

A **gift card** is a special card sent by an **admin** to a wallet that has **never minted any card**. The recipient does not pay; the card participates in the same queue flow as paid cards. When the gift card’s CLC1 and CLC2 caps are reached, payouts go to the **Gift Card SC** (GiftCardReceiver), not to the user. When **both** conditions below are met, the Gift Card SC sends **$2000 USDT** to the gift card user (once per user), triggered by an admin.

---

### 1.2 Eligibility

- **Only wallets with zero cards** can receive a gift card.
- The contract enforces this: `safeMintGift(to)` reverts if `balanceOf(to) != 0` (ErrUserAlreadyMinted).
- The admin frontend checks eligibility (e.g. “Never minted”) before sending.

---

### 1.3 How a gift card is sent

1. Admin opens the **Gift Card** admin page.
2. Admin enters the **recipient wallet** and checks eligibility (backend/contract: user has never minted).
3. Admin calls the **CustomNFT (Master)** function **`safeMintGift(to)`** (e.g. from the admin Gift Card page). No payment is required; the contract mints one **Gift** tier card to `to`. (The contract allows any caller; in practice the admin frontend calls it.)
4. The gift card is a **CLC1 card** (same queue logic as other tiers). It receives “queue amount” from **later mints** — in particular, from mints by users who have this gift card user as **referrer**.

---

### 1.4 Queue flow and referrer condition

- When **referred users** (users who have the gift card user as referrer) mint cards, the **queue amount** from those mints is distributed to **previous cards** in the global queue. The gift card is one of those previous cards, so its **reward balance** grows.
- **Diamond** mints send **$500** per previous card. So when referred users mint **5 Diamond cards** (or equivalent flow), the gift CLC1 card reaches its **CLC1 cap of $2500**.
- The gift card user is treated as **Condition 4 (Diamond)** for Cards Cashback: when someone they referred mints, the cashback amount is the same as if the referrer had a Diamond card. See [CARDS_CASHBACK_SPEC.md](CARDS_CASHBACK_SPEC.md).

---

### 1.5 What happens when gift CLC1 cap is reached ($2500)

When the gift CLC1 card’s reward balance reaches **$2500**, the Master contract:

| Destination        | Amount (USDT) | Note                          |
|--------------------|----------------|-------------------------------|
| **Gift Card SC**   | $1000          | Received by GiftCardReceiver  |
| **SC3 Loyalty**    | $126           | POINTS vault                  |
| **SC1 Overlap**    | $374           | OverlapReceiver               |
| **CLC2 mint**      | $1000          | Auto-mints gift CLC2 for owner |

There is **no direct payout to the user** at CLC1 cap; the user’s CLC2 card is created and will receive flow until it reaches its cap.

The contract requires **contractReserve ≥ $2500** when the gift CLC1 cap is reached: $1500 for the cap payout (1000 + 126 + 374) and $1000 for the CLC2 generation that follows. Otherwise it reverts with `ErrReserveGiftCap`. So the contract must hold at least 2500 USDT in reserve when a gift card is about to hit cap (e.g. from ongoing mints).

**Gift CLC2 generation — same cash flow as Diamond CLC2 ($1000)**  
When the gift CLC1 cap is reached, the contract uses **$1000** (amountForNewCard) from reserve to generate the gift CLC2 card. That **$1000** is paid out in the **same way** as a Diamond CLC2:

| Destination        | Amount (USDT) | Note                     |
|--------------------|----------------|--------------------------|
| **Cash in flow to queue** | $500     | Queue remainder → SC1 Overlap |
| **SC2 Developer**  | $274.50        | Developer & Team profit  |
| **SC1b Overlap2**  | $225           | OverlapReceiver2         |
| **SC3 Loyalty**    | $0.50          | Loyalty vault            |
| **Total**          | **$1000**      | Same as Diamond CLC2     |

So the **$1000** from the gift CLC1 cap is used entirely for the CLC2 generation, with the same split as Diamond CLC2. No extra amount is needed.

---

### 1.6 What happens when gift CLC2 cap is reached ($1000)

When the **gift CLC2** card’s reward balance reaches **$1000**:

| Destination      | Amount (USDT) | Note                                      |
|------------------|----------------|-------------------------------------------|
| **Gift Card SC** | $1000          | Master calls `onGiftCLC2CapReached(beneficiary)` |

Again, no user payout; the Gift Card SC records that **condition 2** is met for that beneficiary. The contract requires **contractReserve ≥ $1000** when the gift CLC2 cap is reached; otherwise it reverts with `ErrReserveGiftCLC2`.

---

### 1.7 When the gift card user receives $2000

The **Gift Card SC** holds the USDT it receives. It sends **$2000** to the gift card user **once** when **both** conditions are met:

| Condition | Description |
|-----------|-------------|
| **Condition 1** | **Loyalty users minted 10 Diamond cards** — Users who have the gift card user as **referrer** have minted **10 Diamond cards in total** (counted off-chain; admin verifies). |
| **Condition 2** | **Gift CLC2 cap reached** — The gift card user’s gift CLC2 card has reached its max cap. The Master contract has called `onGiftCLC2CapReached(beneficiary)` so the Gift Card SC has recorded this. |

**How it’s triggered:** An admin calls the Gift Card SC function **`payoutBothConditionsMet(giftCardUser)`** after verifying condition 1 (e.g. via admin dashboard “Diamond mints by loyalty users”). The contract checks that condition 2 is recorded and that this user has not already been paid, then sends **$2000 USDT** to `giftCardUser`.

---

### 1.8 Admin flow summary

1. **Send gift card** — Admin uses the Gift Card admin page to send a gift card to an eligible wallet (never minted). Contract: `mintGiftCard(to)`.
2. **Track progress** — Admin can see gift card users, whether gift CLC2 cap is reached, and how many Diamond cards were minted by users who have the gift card user as referrer.
3. **Payout $2000** — When both conditions are met, admin calls `payoutBothConditionsMet(giftCardUser)` on the Gift Card SC contract (e.g. from the same admin page or via contract interaction).

The owner of the Gift Card SC can withdraw any remaining USDT in the contract via `withdrawToken`.

---

## Part 2 — Raffle Coupon

### 2.1 Overview

The **1 Million Triple C Card Raffle** uses **raffle coupons** to determine participation. Each user can earn coupons based on their **total points** (Card points + Loyalty points). Coupons are stored in the **backend database** and displayed on the **Global Cards** page (public list) and in the **Admin** section (full list).

---

### 2.2 How coupons are earned

- **1 coupon per 1,000 points** (total points).
- **Total points** = **Card (loyalty) points** + **Level points** (from referral chain).  
  See [CARDS_POINTS.md](CARDS_POINTS.md) for how these are earned.
- **Serial number** of the coupon = the user’s **wallet address** (one serial per wallet).

**Examples:**

| Total points | Coupons |
|--------------|---------|
| 500          | 0       |
| 1,000        | 1       |
| 2,500        | 2       |
| 10,000       | 10      |

---

### 2.3 When coupons are created

Coupons are **not** stored on-chain. The backend creates and updates them when **loyalty/points** are synced:

1. **After a user mints** — The frontend syncs loyalty from the contract and calls the backend **upsert loyalty** (wallet, loyalty_points, level_points). The backend then runs **ensure raffle coupons** for that user: it computes how many coupons they should have (floor(totalPoints / 1000)) and inserts any missing coupons.
2. **When loyalty is fetched from chain** — If the backend returns loyalty data from the chain (GET loyalty by wallet), it can persist that data and run the same “ensure raffle coupons” logic, so coupons appear even if the user didn’t trigger upsert after mint.
3. **Full sync script** — The backend script that syncs all users and loyalty from chain also runs “ensure raffle coupons” for each user after upserting their points.

So: **any path that updates the user’s loyalty_points/level_points in the DB** triggers a recomputation of how many coupons they should have, and new coupons are inserted up to that number.

---

### 2.4 Where coupons are shown

| Place              | Description |
|--------------------|-------------|
| **Global Cards**   | Public list of raffle coupons (wallet/serial, points at award, date). Paginated. |
| **Admin → Raffle coupons** | Full list of all raffle coupons for admins. Same fields; message that coupons are auto-created when points are synced. |

---

### 2.5 Database and API

- **Table:** `raffle_coupons` (user_id, wallet_address, points_at_award, created_at). Migration: `004_raffle_coupons.sql`.
- **Public API:** `GET /api/raffle-coupons` — list coupons (limit, offset) and total count.
- **Admin API:** `GET /api/admin/raffle-coupons` — same for admins.

The backend never deletes coupons when points drop; it only **adds** coupons when total points cross the next 1,000-point threshold. So a user with 2,500 points will have 2 coupons even if they never gain more points.

---

## Summary

| Feature        | Gift Card | Raffle Coupon |
|----------------|-----------|----------------|
| **Who**        | Admin sends to wallet that never minted | Any user with 1000+ total points (Card + Level) |
| **On-chain**  | Yes (CustomNFT + GiftCardReceiver) | No (backend only) |
| **Reward**     | $2000 USDT when both conditions met (10 Diamond by referrals + gift CLC2 cap) | 1 coupon per 1000 points; serial = wallet |
| **Trigger**    | Admin sends card; admin calls payout when both conditions met | Backend creates coupons when loyalty is synced/upserted |

For full cash flow of the gift card (CLC1/CLC2 caps, SC3, SC1), see [MAIN_FLOW_AND_CASHFLOW.md](MAIN_FLOW_AND_CASHFLOW.md) §2.5 and [SMART_CONTRACTS_AND_PAYMENTS.md](SMART_CONTRACTS_AND_PAYMENTS.md) (Gift Card SC).
