# How Users Get Card Points & Loyalty Points

The app shows two balances on the **Points** page. On-chain they live in **SC3 LoyaltyLevelVault** as two different mappings. Amounts are credited when users mint cards or when their referral network mints. The frontend and backend read them from the contract (or from a synced DB cache).

---

## Terminology (contract vs Points page)

| On-chain (SC3) | Points page label | What users see |
|----------------|-------------------|----------------|
| **`loyaltyPoints(user)`** | **CARD POINT** | **Card Points** — points **you** earn from your own CLC1 activity (**paid mints** only on-chain; gift CLC1 cap does not credit SC3). |
| **`levelPoints(user)`** | **LOYALTY POINTS** | **Loyalty Points** — points **you** earn from your **referral downline** (their **paid CLC1** mints). |

Elsewhere in code or older notes, **`loyaltyPoints`** are sometimes called “loyalty” in the Solidity sense; in the **product UI** that same balance is **Card Points**. Similarly, **`levelPoints`** on-chain are labeled **LOYALTY POINTS** on the Points page (not to be confused with the contract name `loyaltyPoints`).

**Total points** (third box on the Points page) = Card Points + Loyalty Points = `loyaltyPoints + levelPoints`.

---

## Two types (summary)

| Type (app name) | Who gets them | How they are earned |
|-----------------|----------------|----------------------|
| **Card Points** (`loyaltyPoints`) | The **minter** / **gift card owner** | **Paid CLC1** mints only (tier table below). **Gift CLC1 cap** does not credit SC3. CLC2 does not add. |
| **Loyalty Points** (`levelPoints`) | **Referrers** in the chain (up to 5) | When someone you referred **mints paid CLC1** (tier table below). |

---

## How Card Points are earned (`loyaltyPoints`)

**Paid CLC1 mint**  
When you mint with USDT (Bronze, Platinum, Emerald, or Diamond), the Master contract sends the SC3 USDT share and credits **`loyaltyPoints`** to your wallet — shown in the app as **Card Points**:

| Tier     | Card Points per mint (`loyaltyPoints` delta) |
|----------|-----------------------------------------------|
| Bronze   | 10                                            |
| Platinum | 100                                           |
| Emerald  | 500                                           |
| Diamond  | 1,000                                         |

**CLC2 does not add Card Points.** When your CLC1 reaches cap and CLC2 is auto-minted, the contract does **not** credit `loyaltyPoints` for that.

**Summary:** **Card Points** increase on **paid CLC1 mints** only. **Gift CLC1 cap** does not add Card Points on-chain. CLC2 generation does not add Card Points.

---

## How Loyalty Points are earned (`levelPoints`)

**Loyalty Points** (`levelPoints`) go to the **referral upline**: the person who referred the minter, then that person’s referrer, up to **5** addresses.

- When a user mints **CLC1** with a **referrer** (e.g. `?ref=0x...`), the contract credits **`levelPoints`** to:
  - the **direct referrer**,
  - then each referrer up the chain,
  - up to **5** wallets in total.

- **Amount per upline wallet** follows the **minted card tier** (not the “Card Points” table):

  | Tier     | Loyalty Points per upline wallet (`levelPoints` delta) |
  |----------|----------------------------------------------------------|
  | Bronze   | 8                                                        |
  | Platinum | 80                                                       |
  | Emerald  | 400                                                      |
  | Diamond  | 800                                                      |

- **Paid CLC1 mints** trigger these credits (tier table above). **Gift CLC1 cap** does **not** send USDT to SC3 on finalize (no **$126** slice); **`loyaltyPoints` / `levelPoints`** are **not** credited by Master at that event. CLC2 auto-mints do **not** credit `levelPoints` beyond normal CLC2 rules.

**Summary:** **Loyalty Points** grow when **someone you referred** **mints paid CLC1**. They add up across many referrals. Chain: minter → direct referrer → … (max 5).

---

## Gift card (CLC1 / CLC2)

Gift cards are **CLC1** NFTs from `safeMintGift` / `safeMintGift(to, referrer)` ($0 at mint). For **Cards Cashback amounts**, a gift user is treated like **Diamond (Condition D)**; **routing** matches **Diamond CLC1** from the first qualifying mint ([GIFT_CARD_AND_RAFFLE_COUPON.md](GIFT_CARD_AND_RAFFLE_COUPON.md)).

If the recipient may **never** pay-mint, admin should call **`safeMintGift(to, referrer)`** with their real upline so `referredBy[recipient]` exists; otherwise when only their **downline** mints, **Loyalty Points** stop at the recipient and the recipient’s referrer gets nothing.

**Card Points (`loyaltyPoints`)**

- No SC3 credit at **gift mint** time.
- **Gift CLC1 cap** finalize does **not** credit **`loyaltyPoints`** on-chain (no SC3 USDT leg at that step).
- **Gift CLC2** does not add further **`loyaltyPoints`** beyond normal CLC2 mint behavior.

**Loyalty Points (`levelPoints`)**

- **Gift CLC1 cap** finalize does **not** credit **`levelPoints`** to upline on-chain.

**Summary for gift:** Card / level points from SC3 follow **paid** mints and other paths; **gift CLC1 cap** alone does not trigger the Diamond CLC1 SC3 point credits.

---

## Where values are stored

- **On-chain:** SC3 stores `loyaltyPoints` and `levelPoints` (see terminology table above for app labels).
- **Backend:** Table `loyalty_points` caches `loyalty_points` and `level_points` columns (same split as chain).
- **Frontend:** Reads SC3 (or API fallback).

**Card Points** come from **paid CLC1 mints** only. **Loyalty Points** come from **downline paid CLC1** only. CLC2 does not add **Card Points**. Master calls SC3 `creditPoints`; no separate user “claim” for these two balances.

---

## Points page: the three boxes

| Page label | On-chain source | Meaning |
|------------|-----------------|--------|
| **CARD POINT** | `loyaltyPoints(account)` | **Card Points** — your **paid** CLC1 mints by tier. Not gift CLC1 cap; not CLC2. |
| **LOYALTY POINTS** | `levelPoints(account)` | **Loyalty Points** — your referral upline rewards from others’ **paid** CLC1 mints. |
| **TOTAL POINTS** | `loyaltyPoints + levelPoints` | Sum; used for achievements / raffle logic. |

**How CARD POINT is calculated:** Not computed in the UI; it is **`loyaltyPoints(user)`** from SC3. Credited on **paid CLC1 mints** only; CLC2 does not add.

**Formula for CARD POINT (`loyaltyPoints`):**

- Sum of **`loyaltyPoints`** credits: each **paid CLC1 mint by you** (+10 / +100 / +500 / +1,000 by tier).
- **CLC2** auto-mint: no **`loyaltyPoints`**.

Example: you mint 1 Platinum CLC1 (+100 **Card Points**) and later CLC2 is created → **CARD POINT** stays 100.
