# Cards Cashback — Specification

**Triple C** referral rewards are called **Cards Cashback**. When a referred user mints a card, **the referrer** receives a reward. The amount is determined by **the referrer’s own condition** (1–4), **not** by the referred user’s condition. The referred user only determines **which tier** was minted (Bronze / Platinum / Emerald / Diamond), which selects the row in the table.

**Settlement rule:** Every paid Cards Cashback amount is applied **in full** to the referrer’s **CLC1/CLC2 card queue on Master** (`CustomNFT`). It is **not** split: **no portion** of this cashback is sent to **SC5** or taken as a separate “5%” leg. (Other flows—such as normal card **withdrawals**—may still use the protocol’s 95%/5% wallet vs 5% SC rules; that is separate from Cards Cashback settlement.)

**Flow:** On each paid mint, Master sends the tier’s full **referral line item** (Bronze $1 … Diamond $100) in USDT to **SC4**. Master then calls `processReferralPayment(referrer, amount)` where `amount` is the **condition-based** total from the table (e.g. Condition 1 + Diamond mint ⇒ **$1**, not $100). SC4 moves that **entire** `amount` to Master and Master credits the referrer’s NFTs in **ascending `tokenId` order** (same `rewardBalance` / incremental payout mechanics as mint-driven rewards). **Surplus** USDT left in SC4 (tier deposit minus condition-based `amount`) stays in SC4.

If the referrer **owns no NFT** at settlement, the **full** condition-based amount **stays in SC4** (event `ReferralCashbackRetainedInSC4`). If there is **no referrer**, `processReferralPayment` is not used for that leg and the tier’s referral USDT remains in SC4.

**Important:** Condition (1–4) is always the **referrer’s** qualification.

**Implementation note:** Master applies cashback across at most **200** of the referrer’s lowest `tokenId` cards per call (gas bound). Any portion not applied to `rewardBalance` in that pass remains in Master’s **contract reserve**.

---

## 1. Cashback amounts by referrer condition and mint tier

All amounts below are **total cashback per referred mint** (USDT)—the value passed to `processReferralPayment`. The **entire** amount goes to the referrer’s card queue (or stays in SC4 if they have no card).

### Condition 1

| Referred user mints | Total cashback (all to queue / SC4) |
|---------------------|--------------------------------------|
| Bronze              | $1                                   |
| Platinum            | $1                                   |
| Emerald             | $1                                   |
| Diamond             | $1                                   |

**Who is in Condition 1:** First minted CLC1 (smallest `tokenId`) is Bronze, or they have no CLC1 cards.

### Condition 2

| Referred user mints | Total cashback |
|---------------------|----------------|
| Bronze              | $1             |
| Platinum            | $10            |
| Emerald             | $10            |
| Diamond             | $10            |

### Condition 3

| Referred user mints | Total cashback |
|---------------------|----------------|
| Bronze              | $1             |
| Platinum            | $10            |
| Emerald             | $50            |
| Diamond             | $50            |

### Condition 4

| Referred user mints | Total cashback |
|---------------------|----------------|
| Bronze              | $1             |
| Platinum            | $10            |
| Emerald             | $50            |
| Diamond             | $100           |

---

## 2. How condition is decided and upgraded

- **Initial condition** follows the **first minted CLC1** (smallest `tokenId`): Bronze → 1, Platinum → 2, Emerald → 3, Diamond → 4.
- **Completed in CLC 2:** CLC1 is **reward complete** and **auto-minted** CLC2 exists.
- **Upgrade:** When the defining card completes in CLC2, condition can rise from the **next** CLC1 in queue. Condition **never** downgrades.

---

## 3. Summary table

| Referred user mints ↓ / Referrer condition → | Condition 1 | Condition 2 | Condition 3 | Condition 4 |
|---------------------------------------------|-------------|-------------|-------------|-------------|
| Bronze                                       | $1          | $1          | $1          | $1          |
| Platinum                                     | $1          | $10         | $10         | $10         |
| Emerald                                      | $1          | $10         | $50         | $50         |
| Diamond                                      | $1          | $10         | $50         | $100        |

---

## 4. Implementation notes

- **CustomNFT:** `getReferrerCashbackLevel`, `_getReferralAmountCashback`; `referrerHasCashbackEligibleCards`; `creditReferralCashbackFromSc4` (SC4-only).
- **ReferralFeeHandler (SC4):** `processReferralPayment` — full `totalAmount` to Master + credit, or retained in SC4; **no** SC5 split on this path. Events: `ReferralCashbackCredited`, `ReferralCashbackRetainedInSC4`, `ReferrerPayoutSkipped`.
- **Backend:** `record` stores the **full** condition-based amount (wei), aligned with on-chain settlement.
- **Frontend:** Copy should describe **full cashback to card queue** (and SC4 retention when the referrer had no NFT), not a 95%/5% split on cashback.

---

## 5. Quick reference: how the referrer reaches each condition

| Condition | Deciding card |
|-----------|----------------|
| **1**     | First mint **Bronze** (or no cards) |
| **2**     | First mint **Platinum**, or upgraded when next is Platinum |
| **3**     | First mint **Emerald**, or upgraded when next is Emerald |
| **4**     | First mint **Diamond**, or upgraded when next is Diamond |
