# Cards Cashback — Specification

**Triple C** referral rewards are called **Cards Cashback**. When a referred user mints a card, **the referrer** receives a reward. The amount is determined by **the referrer’s own condition** (1–4), **not** by the referred user’s condition. The referred user only determines **which tier** was minted (Bronze / Platinum / Emerald / Diamond), which selects the row in the table.

**Settlement rule:** Every paid Cards Cashback amount is applied **in full** to the referrer’s **CLC1/CLC2 card queue on Master** (`CustomNFT`). It is **not** split: **no portion** of this cashback is sent to **SC5** or taken as a separate “5%” leg. (Other flows—such as normal card **withdrawals**—may still use the protocol’s 95%/5% wallet vs 5% SC rules; that is separate from Cards Cashback settlement.)

**Flow:** On each paid mint, Master sends the tier’s full **referral line item** (Bronze $1 … Diamond $100) in USDT to **SC4**. Master then calls `processReferralPayment(referrer, amount)` where `amount` is the **condition-based** total from the table (e.g. Condition 1 + Diamond mint ⇒ **$1**, not $100). SC4 moves that **entire** `amount` to Master and Master credits the referrer’s NFTs in **ascending `tokenId` order** (same `rewardBalance` / incremental payout mechanics as mint-driven rewards). **Surplus** USDT left in SC4 (tier deposit minus condition-based `amount`) stays in SC4.

If the referrer **owns no NFT** at settlement, the **full** condition-based amount **stays in SC4** (event `ReferralCashbackRetainedInSC4`). If there is **no referrer**, `processReferralPayment` is not used for that leg and the tier’s referral USDT remains in SC4.

**Important:** Condition (1–4) is always the **referrer’s** qualification.

**Implementation note:** Master applies cashback across at most **200** of the referrer’s lowest `tokenId` cards per call (gas bound). Any portion not applied in that pass is either sent to **SC1 overlap** (`overlapReceiver`) or, if overlap is unset, left in Master’s **contract reserve** (see §1.0).

### 1.0 Queue routing (CLC1 → virtual CLC2 → CLC2 token → overlap)

Cards Cashback is applied in **ascending `tokenId`** order for the referrer’s wallet.

1. **CLC1 — first wallet tranche**  
   Cashback uses the same mechanics as mint-driven rewards: accrual to `rewardBalance` and incremental **95% / 5%** payouts until **`withdrawAmount`** (first tranche) for that CLC1 card is satisfied.

2. **CLC1 — paired CLC2 not minted yet**  
   Once that CLC1’s **first tranche** is fully paid out, further cashback does **not** fill the remainder of CLC1’s bucket toward CLC2 funding. Instead it:
   - Pays **95% / 5%** toward the **paired CLC2 card’s first tranche** (`withdrawAmountCLC2`) **even if the CLC2 NFT does not exist yet** (“virtual” CLC2 payout on-chain: gross amount is tracked on the **CLC1** `tokenId` and reduces the future CLC2 card’s obligation).
   - **Reduces** the paired card’s **`rewardCapCLC2`** by the same absorbed amounts (first tranche via payouts; any further absorption up to `rewardCapCLC2` can reduce the cap **without** additional wallet payout until the CLC2 token is minted).

3. **CLC2 token (after auto-mint)**  
   When the CLC2 card exists, cashback only applies until its **first wallet tranche** (`withdrawAmountCLC2`) is satisfied (using the reduced cap and any prepaid `amountPaidOutToWallet` seeded at mint).

4. **All first tranches satisfied**  
   If **every** token in the processed queue has **no remaining “first tranche” capacity** for cashback (CLC1 first tranche done, virtual + minted CLC2 first tranche done, and CLC2 cap fully absorbed where applicable), **any leftover** cashback USDT for that settlement is transferred to **SC1 overlap** (`overlapReceiver`). If overlap is not configured, that remainder stays in Master’s reserve.

**Gift-tier CLC1:** **Cards Cashback** is applied entirely through **`rewardBalance`** accrual on the Gift CLC1 token (no CLC1 wallet tranche; **`withdrawAmount` = 0**). The contract does **not** route Gift CLC1 cashback into the virtual-CLC2 path before CLC2 is minted, so the **$2000** CLC1 cap can be reached from queue + cashback like a single bucket. **Gift CLC2** follows §1.0 like other CLC2 cards. See [GIFT_CARD_AND_RAFFLE_COUPON.md](GIFT_CARD_AND_RAFFLE_COUPON.md).

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

- **CustomNFT:** `getReferrerCashbackLevel`, `_getReferralAmountCashback`; `referrerHasCashbackEligibleCards`; `creditReferralCashbackFromSc4` (SC4-only). Storage: `clc1ParentOfClc2`, `clc2TokenOfClc1`, `referralCashbackClc2WalletPrepaid`, `referralCashbackClc2CapReduction`; event `ReferralCashbackVirtualClc2Payout`. `_getRewardCap` for CLC2 subtracts `referralCashbackClc2CapReduction[parentCLC1]`.
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
