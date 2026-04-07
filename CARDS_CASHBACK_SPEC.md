# Cards Cashback — Specification

**Triple C** referral rewards are called **Cards Cashback**. When a referred user mints a card, **the referrer** (the user who receives the cashback) gets a reward. The cashback amount is determined by **the referrer’s own condition** (1–4), **not** by the referred user’s condition or tier history. The only thing the referred user contributes is **which card tier they just minted** (Bronze / Platinum / Emerald / Diamond), which picks the row in the table.  
The **Cards Cashback amount** is always sent into **SC4 ReferralFeeHandler**. When SC4 settles cashback for a referrer, **5%** goes to the **5% SC** (FivePercentReceiver, SC5). **95%** is credited onto the **referrer’s CLC1 cards in queue** on the Master (CustomNFT) contract—the same reward mechanics as mint queue flow (including incremental 95%/5% wallet payouts as caps allow). If the referrer has **no CLC1 card** in queue, that **95% remains in SC4**. If no referrer exists, the full cashback amount stays in SC4.

**Important:** Condition (1–4) is always the **referrer’s** qualification — i.e. the person receiving the cashback. The referred user only determines the mint tier (which card they bought).

---

## 1. Cashback amounts by referrer condition and mint tier

All amounts below are **total cashback per referred mint** (in USDT), routed **for the referrer** via SC4.  
Split when SC4 settles: **95%** to the referrer’s **CLC1 card queue** on Master (or **retained in SC4** if they have no CLC1 card), **5% to 5% SC**.

The **columns** are the **referrer’s condition** (who receives the cashback). The **rows** are the **tier of the card the referred user just minted**.

### Condition 1 (default) — referrer’s level

When **the referrer** is in Condition 1, they receive (for any mint tier of their referred user):

| Referred user mints | Total cashback | 95% (queue or SC4) | To 5% SC |
|---------------------|----------------|--------------------|----------|
| Bronze              | $1             | $0.95              | $0.05    |
| Platinum            | $1             | $0.95              | $0.05    |
| Emerald             | $1             | $0.95              | $0.05    |
| Diamond             | $1             | $0.95              | $0.05    |

**Who is in Condition 1:** Their **first minted card** (oldest CLC1 by tokenId) is Bronze, or they have no cards.

---

### Condition 2 — referrer’s level

When **the referrer** is in Condition 2, they receive (amount depends on what tier the referred user just minted):

| Referred user mints | Total cashback | 95% (queue or SC4) | To 5% SC |
|---------------------|----------------|--------------------|----------|
| Bronze              | $1             | $0.95              | $0.05    |
| Platinum            | $10            | $9.50              | $0.50    |
| Emerald             | $10            | $9.50              | $0.50    |
| Diamond             | $10            | $9.50              | $0.50    |

**Who qualifies for Condition 2:** Their **first minted card** (in queue order) is Platinum — or they started lower and **upgraded** to 2 when that card was completed in CLC2 and the next card in queue is Platinum (see §2).

---

### Condition 3 — referrer’s level

When **the referrer** is in Condition 3, they receive (amount depends on what tier the referred user just minted):

| Referred user mints | Total cashback | 95% (queue or SC4) | To 5% SC |
|---------------------|----------------|--------------------|----------|
| Bronze              | $1             | $0.95              | $0.05    |
| Platinum            | $10            | $9.50              | $0.50    |
| Emerald             | $50            | $47.50             | $2.50    |
| Diamond             | $50            | $47.50             | $2.50    |

**Who qualifies for Condition 3:** Their **first minted card** (in queue order) is Emerald — or they **upgraded** to 3 when the previous defining card was completed in CLC2 and the next card in queue is Emerald.

---

### Condition 4 — referrer’s level

When **the referrer** is in Condition 4, they receive (amount depends on what tier the referred user just minted):

| Referred user mints | Total cashback | 95% (queue or SC4) | To 5% SC |
|---------------------|----------------|--------------------|----------|
| Bronze              | $1             | $0.95              | $0.05    |
| Platinum            | $10            | $9.50              | $0.50    |
| Emerald             | $50            | $47.50             | $2.50    |
| Diamond             | $100           | $95.00             | $5.00    |

**Who qualifies for Condition 4:** Their **first minted card** (in queue order) is Diamond — or they **upgraded** to 4 when the previous defining card was completed in CLC2 and the next card in queue is Diamond.

---

## 2. How condition is decided and upgraded

- **Initial condition** is set by the user’s **first minted card** (the CLC1 card with the smallest tokenId they own): Bronze → Condition 1, Platinum → 2, Emerald → 3, Diamond → 4.
- **“Completed in CLC 2”** (for that card): the CLC1 card has reached its reward cap (**reward complete**) and the contract has already created its CLC2 card (**auto-minted**).
- **Upgrade:** When the card that currently defines the referrer’s condition is **completed in CLC 2**, the condition can be **upgraded** by the **next first card in the queue** (the next CLC1 card by tokenId). The new condition is the tier of that next card (or higher — condition is never downgraded).
- **Example:** User’s first mint is Bronze → Condition 1. When that Bronze CLC1 is completed in CLC2, they mint Platinum. The next card in queue is that Platinum → condition upgrades to Condition 2. Condition can never go down.

---

## 3. Summary table (total cashback per referred mint)

**Columns** = referrer’s condition (the person who receives the cashback).  
**Rows** = tier of the card the referred user just minted.

| Referred user mints ↓ / Referrer condition → | Condition 1 | Condition 2 | Condition 3 | Condition 4 |
|---------------------------------------------|-------------|-------------|-------------|-------------|
| Bronze                                       | $1          | $1          | $1          | $1          |
| Platinum                                     | $1          | $10         | $10         | $10         |
| Emerald                                      | $1          | $10         | $50         | $50         |
| Diamond                                      | $1          | $10         | $50         | $100        |

Split for every paid cashback: **95% to the referrer’s CLC1 cards in queue** on Master (or **left in SC4** if the referrer has no CLC1 card), **5% to 5% SC (SC5)**.

---

## 4. Implementation notes

- **On-chain:** The NFT contract computes **the referrer’s** condition (1–4) from the **first minted card** (smallest CLC1 tokenId) and **upgrades** when that card is completed in CLC2 (reward complete + auto-minted), using the next CLC1 in queue; condition is never downgraded. View function: `getReferrerCashbackLevel(referrer)`. The referred user’s condition or history does not affect the cashback amount; only the referrer’s level and the tier of the card just minted do.
- **Payment flow:** On each paid mint, the contract sends the cashback amount to SC4 (ReferralFeeHandler). If a referrer exists, Master calls `processReferralPayment` with a flag indicating whether the referrer has any CLC1 card. SC4 sends 5% to SC5 (or SC2 if SC5 is not set) and pulls the 95% into Master via `creditReferralToReferrerQueue` when that flag is true; otherwise the 95% stays in SC4. If no referrer exists, the cashback amount remains in SC4.
- **Backend:** When recording a referral fee (e.g. for history), the backend reads the **referrer’s** cashback level from the contract and uses the same table so stored amounts match on-chain payouts.

---

## 5. Quick reference: how the referrer reaches each condition

These apply to **the referrer** (the user who receives the cashback). **Condition is decided by the first card in the user’s mint queue** (oldest CLC1); it **upgrades** when that card is completed in CLC2 and the next card in queue sets a higher condition; it **never downgrades**.

| Condition | Deciding card (first in queue or after upgrade) |
|-----------|--------------------------------------------------|
| **1**     | First mint is **Bronze** (or no cards) |
| **2**     | First mint is **Platinum**, or upgraded from 1 when Bronze completed in CLC2 and next is Platinum |
| **3**     | First mint is **Emerald**, or upgraded when defining card completed in CLC2 and next is Emerald |
| **4**     | First mint is **Diamond**, or upgraded when defining card completed in CLC2 and next is Diamond |
