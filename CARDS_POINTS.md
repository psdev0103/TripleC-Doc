# How Users Get Cards Points (Loyalty & Level Points)

Cards points are **loyalty points** and **level points** stored on-chain in **SC3 LoyaltyLevelVault**. They are credited automatically when users mint cards or when someone they referred mints. The frontend and backend read these from the contract (or from a synced DB cache).

---

## Two types of points

| Type | Who gets them | How they are earned |
|------|----------------|----------------------|
| **Loyalty points** | The **minter** (person who buys the card) | Minting a CLC1 card only (CLC2 does not add card points) |
| **Level points** | **Referrers** in the chain (up to 5) | When someone you referred mints a CLC1 card; the minter’s **direct referrer** and up to 4 higher referrers each get level points |

**Total points** (shown in the app) = loyalty points + level points.

---

## How loyalty points are earned

**Minting a CLC1 card (paid mint)**  
When you mint a card with USDT (Bronze, Platinum, Emerald, or Diamond), the CustomNFT (Master) contract calls SC3 and credits **loyalty points** (card points) to your wallet. Amount depends on tier:

| Tier    | Loyalty points per mint |
|---------|--------------------------|
| Bronze  | 10                       |
| Platinum| 100                      |
| Emerald | 500                      |
| Diamond | 1,000                    |

**CLC2 cards do not give card points.** When your CLC1 reaches cap and a CLC2 card is auto-generated for you, the contract does **not** credit any loyalty/card points. Card point is from CLC1 mints only.

**Summary:** You get loyalty (card) points only when you **mint a CLC1 card**. CLC2 generation does not add points.

---

## How level points are earned

Level points go to the **referral chain**: the person who referred the minter, and that referrer’s referrer, and so on, up to **5 levels**.

- When a user mints a **CLC1 card** with a **referrer** set (e.g. they used a link like `?ref=0x...`), the contract credits **level points** to:
  - the **direct referrer** (the `ref` address),
  - then that referrer’s referrer,
  - then the next referrer up the chain,
  - up to **5 referrers** in total.

- The **amount per referrer** is the same as the tier’s “level” amount (not the loyalty amount). In the contract these are:

  | Tier    | Level points (per referrer in chain) |
  |---------|--------------------------------------|
  | Bronze  | 8   |
  | Platinum| 80  |
  | Emerald | 400 |
  | Diamond | 800 |

- **Only CLC1 paid mints** trigger level points. CLC2 auto-mints do **not** credit level points to anyone.

**Summary:** You get level points when **someone you referred** (or someone in your referral chain) **mints a CLC1 card**. You can receive level points from many referrals; they add up. The chain is: minter → direct referrer → referrer’s referrer → … (max 5).

---

## Where points are stored and how the app shows them

- **On-chain:** SC3 (LoyaltyLevelVault) holds `loyaltyPoints(user)` and `levelPoints(user)`.
- **Backend:** The `loyalty_points` table is a cache; it can be filled by syncing from chain (e.g. `npm run sync:all` or loyalty sync script).
- **Frontend:** When the user’s wallet is connected, the app reads **loyalty** and **level** points from the SC3 contract. If the contract is not available, it can fall back to the backend API (which uses the cached DB or can fetch from chain).

So: **to get cards points, users must mint CLC1 cards (and/or have their CLC2 generated) for loyalty points, and have referred someone who mints CLC1 for level points.** All crediting is done by the Master contract calling SC3; no separate “claim” step is needed for points to appear.

---

## Points page: CARD POINT vs LOYALTY POINTS

On the **Points** page, the three boxes mean:

| Page label        | Source (contract / context) | Meaning |
|-------------------|-----------------------------|--------|
| **CARD POINT**    | `loyaltyPoints(account)` from SC3 | Loyalty points: earned by **you** when **you** mint CLC1 cards or when your CLC2 is auto-generated. |
| **LOYALTY POINTS**| `levelPoints(account)` from SC3 | Level points: earned when **your referrals** (or their chain) mint CLC1 cards. |
| **TOTAL POINTS**  | `loyaltyPoints + levelPoints` | Sum of the two; used for reward achievements. |

**How CARD POINT is calculated:** It is **not** calculated on the Points page. It is **read from the chain** — SC3 LoyaltyLevelVault's `loyaltyPoints(user)`. The contract **accumulates** it only when you **mint a CLC1 card**; CLC2 generation does not add card points.

**Formula for CARD POINT (loyalty points):**

- **Card point** = sum of loyalty credits from **CLC1 mints only**.
- Each **CLC1 mint by you**: +10 (Bronze), +100 (Platinum), +500 (Emerald), or +1,000 (Diamond).
- **CLC2 auto-generated for you**: no points (not counted for card point).

Example: you mint 1 Platinum CLC1 (+100) and later your CLC2 is auto-minted → your **CARD POINT** stays 100 (CLC2 does not add points).
