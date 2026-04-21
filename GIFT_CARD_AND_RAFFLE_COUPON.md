# Gift Card & Raffle Coupon

This document describes the **Gift Card** flow (admin-sent card; **queue + Cards Cashback** accrue like Diamond CLC1 from the start; **$1000** to Gift Card SC at CLC1 cap finalize and **$1000** at CLC2 cap finalize; **two** separate **$1000** user payouts from Gift Card SC when **three** referred Diamond CLC1 mints are recorded on-chain **and** the corresponding cap flag is set on Gift Card SC) and the **Raffle Coupon** system (1 coupon per 1,000 points; serial = wallet).

---

## Part 1 — Gift Card

### 1.1 Overview

A **gift card** is minted by an **admin** to a wallet that has **never minted any card**. The recipient does not pay. **Gift CLC1** receives **main queue** (`rewardToPrev`) and **Cards Cashback** (Condition 4, same as Diamond referrer) from the first qualifying mint — there is **no** locked phase, **no** incremental USDT routing to Gift Card SC before cap, and **no** **$300** milestone. **USDT** accrues on the gift card’s **`rewardBalance`** until the **CLC1 reward cap** ($2500 nominal in default tier config). When CLC1 hits cap, Master finalizes: **$1000** to **Gift Card SC**, SC3/SC1 slices per spec, and **$1000** funds **gift CLC2** generation. When **gift CLC2** hits cap, **$1000** goes to Gift Card SC. The **gift card user** may receive **$1000** from Gift Card SC after **3 referred Diamond CLC1 mints** (on-chain counter) **and** CLC1 cap recorded on Gift Card SC, and **another $1000** after the same **3** diamonds **and** CLC2 cap recorded — see §1.8.

---

### 1.2 Eligibility

- **Only wallets with zero cards** can receive a gift card.
- The contract enforces this: `safeMintGift(to)` reverts if `balanceOf(to) != 0` (ErrUserAlreadyMinted).
- The admin frontend checks eligibility (e.g. “Never minted”) before sending.

---

### 1.3 How a gift card is sent

1. Admin opens the **Gift Card** admin page.
2. Admin enters the **recipient wallet** and checks eligibility (backend/contract: user has never minted).
3. Admin calls **`safeMintGift(to)`** or **`safeMintGift(to, referrer)`** on **CustomNFT (Master)** (admin Gift Card page). No payment is required; the contract mints one **Gift** tier card to `to`. If the recipient may **never** do a paid first mint, pass **`referrer`** as the wallet that referred them (optional). That sets `referredBy[to]` on-chain so when the recipient’s **downline** mints, **Loyalty Points** (`levelPoints`) credit the full upline (e.g. recipient + their referrer). Omit `referrer` only when there is no referrer or it will be set on the recipient’s first paid mint later.
4. The gift card is a **CLC1 card** (same queue logic as other tiers). It receives “queue amount” from **later mints** — in particular, from mints by users who have this gift card user as **referrer**.

---

### 1.4 Three referred Diamond CLC1 mints (counter on Master)

Each **paid Diamond CLC1** mint by a user whose **on-chain referrer** is the **gift card owner** increments **`referralDiamondMintCountForUpline[giftOwner]`** up to **3** (same upline resolution: **`referredBy[mintee]`** else **`referrerOf[mintedTokenId]`**). This counter **only** gates **Gift Card SC** user payouts (§1.8). When the count reaches **3**, Master emits **`GiftReferralDiamondCountReached(giftReferrerUpline, 3)`**.

**Cards Cashback** and **queue** behave like **Diamond CLC1** for the gift owner from the first mint (SC4 settles to Master; queue accrues to **`rewardBalance`**).

---

### 1.5 Queue flow and referrer condition

- Gift CLC1 uses the same **queue** and **Cards Cashback** rules as **Diamond CLC1** ([CARDS_CASHBACK_SPEC.md](CARDS_CASHBACK_SPEC.md)). Referrer tier condition for cashback amounts is **Condition 4**.

---

### 1.6 What happens when gift CLC1 cap is reached ($2500 nominal)

When the gift CLC1 card’s **`rewardBalance` reaches the CLC1 reward cap** (nominal **$2500** in default `TierConfigLib`), the Master contract:

| Destination        | Amount (USDT) | Note                          |
|--------------------|----------------|-------------------------------|
| **Gift Card SC**   | $1000          | Received by GiftCardReceiver  |
| **SC3 Loyalty**    | $126           | POINTS vault (same slice as Diamond CLC1 mint’s SC3 share); Master credits **loyalty points** to the gift owner and **level points** to their referral upline — see [CARDS_POINTS.md](CARDS_POINTS.md) |
| **SC1 Overlap**    | $374           | OverlapReceiver               |
| **CLC2 mint**      | $1000          | Auto-mints gift CLC2 for owner |

There is **no direct payout to the user** at CLC1 cap; the user’s CLC2 card is created and will receive flow until it reaches its cap.

Master calls **`GiftCardReceiver.onGiftCLC1CapReached(beneficiary)`** after sending the **$1000** slice so Gift Card SC can record **condition A** for the **first $1000** user payout (§1.8).

The contract requires **contractReserve ≥ $2500** when gift CLC1 cap finalization runs: **$1500** for the cap payout (1000 + 126 + 374) and **$1000** for the CLC2 generation that follows. If reserve is insufficient, Master reverts with `ErrReserveGiftCap`.

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

### 1.7 What happens when gift CLC2 cap is reached ($1000)

When the **gift CLC2** card’s reward balance reaches **$1000**:

| Destination      | Amount (USDT) | Note                                      |
|------------------|----------------|-------------------------------------------|
| **Gift Card SC** | $1000          | Master calls `onGiftCLC2CapReached(beneficiary)` |

Again, no user payout at finalize; the Gift Card SC records **CLC2 cap** for that beneficiary via **`onGiftCLC2CapReached`**. The contract requires **contractReserve ≥ $1000** when the gift CLC2 cap is reached; otherwise it reverts with `ErrReserveGiftCLC2`.

---

### 1.8 When the gift card user receives $1000 + $1000

**Gift Card SC** holds USDT from CLC1/CLC2 finalizations. The **owner** calls:

| Function | Requirements (enforced on-chain) | Payout |
|----------|-----------------------------------|--------|
| **`payoutGiftClc1Bonus(giftCardUser)`** | `referralDiamondMintCountForUpline(giftCardUser) >= 3` and `giftClc1CapReached[giftCardUser]` | **$1000** |
| **`payoutGiftClc2Bonus(giftCardUser)`** | Same **3** diamonds and `giftClc2CapReached[giftCardUser]` | **$1000** |

**Legacy:** **`payoutBothConditionsMet`** still sends **$2000** in one transfer when only CLC2 was historically recorded (older deployments). It **reverts** if either split bonus was already used (`paidGiftClc1Bonus` / `paidGiftClc2Bonus`).

---

### 1.9 Admin flow summary

1. **Send gift card** — **`safeMintGift(to)`** / **`safeMintGift(to, referrer)`** on CustomNFT.
2. **Track** — Diamond count: **`referralDiamondMintCountForUpline`**; caps: **`giftClc1CapReached`** / **`giftClc2CapReached`** on Gift Card SC (set by Master on finalize).
3. **Payout** — **`payoutGiftClc1Bonus`** / **`payoutGiftClc2Bonus`** when each row is ready (admin UI on Gift Card page).

The owner of the Gift Card SC can withdraw any remaining USDT via **`withdrawToken`**.

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
| **Reward**     | **$1000** + **$1000** from Gift Card SC (§1.8) after 3 referred Diamonds + CLC1/CLC2 cap flags | 1 coupon per 1000 points; serial = wallet |
| **Trigger**    | Admin sends card; admin calls `payoutGiftClc1Bonus` / `payoutGiftClc2Bonus` | Backend creates coupons when loyalty is synced/upserted |

For full cash flow of the gift card (CLC1/CLC2 caps, SC3, SC1), see [MAIN_FLOW_AND_CASHFLOW.md](MAIN_FLOW_AND_CASHFLOW.md) §2.5 and [SMART_CONTRACTS_AND_PAYMENTS.md](SMART_CONTRACTS_AND_PAYMENTS.md) (Gift Card SC).
