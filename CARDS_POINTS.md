# How Users Get Cards Points (Loyalty & Level Points)

Cards points are **loyalty points** and **level points** stored on-chain in **SC3 LoyaltyLevelVault**. They are credited automatically when users mint cards or when someone they referred mints. The frontend and backend read these from the contract (or from a synced DB cache).

---

## Two types of points

| Type | Who gets them | How they are earned |
|------|----------------|----------------------|
| **Loyalty points** | The **minter** / **gift card owner** | Paid CLC1 mints; **gift CLC1** +1,000 when CLC1 hits **cap** (see Gift section). CLC2 does not add. |
| **Level points** | **Referrers** in the chain (up to 5) | When someone you referred **mints paid CLC1**, or when a referred **gift card user’s CLC1 hits cap** (Diamond-equivalent upline credits) |

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

**Summary:** You get loyalty (card) points on **paid CLC1 mints** and when your **gift CLC1** reaches **cap**. CLC2 generation does not add points.

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

- **Paid CLC1 mints** trigger level points (tier table above). **Gift CLC1 cap** triggers **Diamond** amounts (800 per upline) when SC3 receives the gift’s $126 share. CLC2 auto-mints do **not** credit level points to anyone.

**Summary:** You get level points when **someone you referred** **mints paid CLC1** or when a referred user’s **gift CLC1 reaches cap**. You can receive level points from many referrals; they add up. The chain is: minter → direct referrer → referrer’s referrer → … (max 5).

---

## Gift card (CLC1 / CLC2)

Gift cards are **CLC1** NFTs minted via `safeMintGift` ($0 to the recipient at mint). For **Cards Cashback**, a gift card user is treated like **Diamond (Condition D)** on-chain.

**Loyalty (“card”) points**

- There is **no** SC3 USDT (and **no** loyalty points credit) at the moment the gift is minted.
- When the **gift CLC1** card reaches its **$2500 cap**, the contract sends **$126** to SC3 (same economic role as the Diamond tier’s loyalty/level USDT share on a paid mint). At that moment the contract credits **loyalty points to the gift card owner** using the **Diamond** amounts: **+1,000** loyalty points (same as a Diamond CLC1 mint).
- **Gift CLC2** generation and cap payouts do **not** add further loyalty points (aligned with “CLC2 does not give card points” for paid tiers).

**Level points**

- When the gift CLC1 hits cap and **$126** is sent to SC3, the contract credits **level points** to the gift owner’s **referral upline** (up to **5** addresses), using the **Diamond** per-referrer amount: **800** level points each — same as when a referred user mints a **Diamond** CLC1 card.
- The upline is read from `referredBy[giftOwner]` (set on the gift owner’s **first paid mint** with a referrer, same as other users). If the gift owner has no referrer recorded, only loyalty is credited at cap; no level points are distributed.

**Summary for gift:** Points behave like a **Diamond CLC1 mint** for SC3, but **only when the gift CLC1 reaches cap**, not at gift mint time.

---

## Where points are stored and how the app shows them

- **On-chain:** SC3 (LoyaltyLevelVault) holds `loyaltyPoints(user)` and `levelPoints(user)`.
- **Backend:** The `loyalty_points` table is a cache; it can be filled by syncing from chain (e.g. `npm run sync:all` or loyalty sync script).
- **Frontend:** When the user’s wallet is connected, the app reads **loyalty** and **level** points from the SC3 contract. If the contract is not available, it can fall back to the backend API (which uses the cached DB or can fetch from chain).

So: **loyalty points** come from **paid CLC1 mints** and from **gift CLC1 reaching cap**; **level points** come from **downline paid CLC1 mints** and from a referred user’s **gift CLC1 cap** (upline). CLC2 generation does not add loyalty points. All crediting is done by the Master contract calling SC3; no separate “claim” step is needed for points to appear.

---

## Points page: CARD POINT vs LOYALTY POINTS

On the **Points** page, the three boxes mean:

| Page label        | Source (contract / context) | Meaning |
|-------------------|-----------------------------|--------|
| **CARD POINT**    | `loyaltyPoints(account)` from SC3 | Loyalty points: earned by **you** when **you** mint paid CLC1 cards; **gift CLC1** adds the same when your gift CLC1 hits **cap** (+1,000 like Diamond). CLC2 does not add. |
| **LOYALTY POINTS**| `levelPoints(account)` from SC3 | Level points: earned when **your downline** mints paid CLC1 cards, and when a **gift card user you referred** hits **gift CLC1 cap** (Diamond-equivalent upline credits). |
| **TOTAL POINTS**  | `loyaltyPoints + levelPoints` | Sum of the two; used for reward achievements. |

**How CARD POINT is calculated:** It is **not** calculated on the Points page. It is **read from the chain** — SC3 LoyaltyLevelVault's `loyaltyPoints(user)`. The contract credits it on **paid CLC1 mints** and when a **gift CLC1** reaches **cap**; CLC2 generation does not add card points.

**Formula for CARD POINT (loyalty points):**

- **Card point** = sum of loyalty credits from **paid CLC1 mints** plus **gift CLC1 cap** (+1,000 once).
- Each **paid CLC1 mint by you**: +10 (Bronze), +100 (Platinum), +500 (Emerald), or +1,000 (Diamond).
- **Gift CLC1** reaches **cap**: +1,000 (same as Diamond) when SC3 receives the $126 share.
- **CLC2 auto-generated for you**: no points (not counted for card point).

Example: you mint 1 Platinum CLC1 (+100) and later your CLC2 is auto-minted → your **CARD POINT** stays 100 (CLC2 does not add points).
