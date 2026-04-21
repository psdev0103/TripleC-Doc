# How Users Get Card Points & Loyalty Points

The app shows two balances on the **Points** page. On-chain they live in **SC3 LoyaltyLevelVault** as two different mappings. Amounts are credited when users mint cards or when their referral network mints. The frontend and backend read them from the contract (or from a synced DB cache).

---

## Terminology (contract vs Points page)

| On-chain (SC3) | Points page label | What users see |
|----------------|-------------------|----------------|
| **`loyaltyPoints(user)`** | **CARD POINT** | **Card Points** — points **you** earn from your own CLC1 activity (paid mints + gift CLC1 cap). |
| **`levelPoints(user)`** | **LOYALTY POINTS** | **Loyalty Points** — points **you** earn from your **referral downline** (their paid CLC1 mints + gift CLC1 cap upline). |

Elsewhere in code or older notes, **`loyaltyPoints`** are sometimes called “loyalty” in the Solidity sense; in the **product UI** that same balance is **Card Points**. Similarly, **`levelPoints`** on-chain are labeled **LOYALTY POINTS** on the Points page (not to be confused with the contract name `loyaltyPoints`).

**Total points** (third box on the Points page) = Card Points + Loyalty Points = `loyaltyPoints + levelPoints`.

---

## Two types (summary)

| Type (app name) | Who gets them | How they are earned |
|-----------------|----------------|----------------------|
| **Card Points** (`loyaltyPoints`) | The **minter** / **gift card owner** | Paid CLC1 mints; **gift CLC1** +1,000 when CLC1 hits **cap** (see Gift section). CLC2 does not add. |
| **Loyalty Points** (`levelPoints`) | **Referrers** in the chain (up to 5) | When someone you referred **mints paid CLC1**, or when a referred **gift card user’s CLC1 hits cap** (Diamond-equivalent upline credits). |

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

**Summary:** **Card Points** increase on **paid CLC1 mints** and when your **gift CLC1** reaches **cap**. CLC2 generation does not add Card Points.

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

- **Paid CLC1 mints** trigger these credits (tier table above). **Gift CLC1 cap** triggers **Diamond** amounts (**800** per upline) when SC3 receives the gift’s **$126** share. CLC2 auto-mints do **not** credit `levelPoints`.

**Summary:** **Loyalty Points** grow when **someone you referred** **mints paid CLC1** or when a referred user’s **gift CLC1 reaches cap**. They add up across many referrals. Chain: minter → direct referrer → … (max 5).

---

## Gift card (CLC1 / CLC2)

Gift cards are **CLC1** NFTs from `safeMintGift` / `safeMintGift(to, referrer)` ($0 at mint). For **Cards Cashback amounts**, a gift user is treated like **Diamond (Condition D)**; **routing** matches **Diamond CLC1** from the first qualifying mint ([GIFT_CARD_AND_RAFFLE_COUPON.md](GIFT_CARD_AND_RAFFLE_COUPON.md)).

If the recipient may **never** pay-mint, admin should call **`safeMintGift(to, referrer)`** with their real upline so `referredBy[recipient]` exists; otherwise when only their **downline** mints, **Loyalty Points** stop at the recipient and the recipient’s referrer gets nothing.

**Card Points (`loyaltyPoints`)**

- No SC3 credit at **gift mint** time.
- When **gift CLC1** hits its **CLC1 reward cap** (nominal **$2500** in default tier config), **$126** goes to SC3; the contract credits **+1,000** to **`loyaltyPoints`** for the gift owner (**Card Points**, same as Diamond CLC1 mint).
- **Gift CLC2** does not add further **`loyaltyPoints`**.

**Loyalty Points (`levelPoints`)**

- At that same gift CLC1 cap event, **`levelPoints`** are credited to the gift owner’s **upline** (up to **5**), **800** each (Diamond-equivalent).
- Upline comes from `referredBy[giftOwner]` (set on first **paid** mint with a referrer). If none, only **`loyaltyPoints`** are credited at cap; no **`levelPoints`**.

**Summary for gift:** For SC3, behaves like a **Diamond CLC1 mint** at **gift CLC1 cap only**, not at gift mint time.

---

## Where values are stored

- **On-chain:** SC3 stores `loyaltyPoints` and `levelPoints` (see terminology table above for app labels).
- **Backend:** Table `loyalty_points` caches `loyalty_points` and `level_points` columns (same split as chain).
- **Frontend:** Reads SC3 (or API fallback).

**Card Points** come from **paid CLC1 mints** and **gift CLC1 cap**. **Loyalty Points** come from **downline paid CLC1** and **referred user’s gift CLC1 cap**. CLC2 does not add **Card Points**. Master calls SC3 `creditPoints`; no separate user “claim” for these two balances.

---

## Points page: the three boxes

| Page label | On-chain source | Meaning |
|------------|-----------------|--------|
| **CARD POINT** | `loyaltyPoints(account)` | **Card Points** — your CLC1 mints (paid tiers) + gift CLC1 cap (+1,000). Not CLC2. |
| **LOYALTY POINTS** | `levelPoints(account)` | **Loyalty Points** — your referral upline rewards from others’ paid CLC1 mints + referred users’ gift CLC1 cap (800 per upline tier Diamond-equivalent). |
| **TOTAL POINTS** | `loyaltyPoints + levelPoints` | Sum; used for achievements / raffle logic. |

**How CARD POINT is calculated:** Not computed in the UI; it is **`loyaltyPoints(user)`** from SC3. Credited on **paid CLC1 mints** and at **gift CLC1 cap**; CLC2 does not add.

**Formula for CARD POINT (`loyaltyPoints`):**

- Sum of **`loyaltyPoints`** credits: each **paid CLC1 mint by you** (+10 / +100 / +500 / +1,000 by tier) plus **gift CLC1 cap** (+1,000 once when $126 hits SC3).
- **CLC2** auto-mint: no **`loyaltyPoints`**.

Example: you mint 1 Platinum CLC1 (+100 **Card Points**) and later CLC2 is created → **CARD POINT** stays 100.
