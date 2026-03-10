# Cards Cashback — Specification

**Triple C** referral rewards are called **Cards Cashback**. When a referred user mints a card, **the referrer** (the user who receives the cashback) gets a reward. The cashback amount is determined by **the referrer’s own condition** (1–4), **not** by the referred user’s condition or tier history. The only thing the referred user contributes is **which card tier they just minted** (Bronze / Platinum / Emerald / Diamond), which picks the row in the table.  
The **Cards Cashback amount** is always sent into **SC4 ReferralFeeHandler**. When SC4 pays cashback to a referrer, **95%** goes to the referrer’s wallet and **5%** goes to the **5% SC** (FivePercentReceiver, SC5). If no referrer exists, the cashback amount stays in SC4.

**Important:** Condition (1–4) is always the **referrer’s** qualification — i.e. the person receiving the cashback. The referred user only determines the mint tier (which card they bought).

---

## 1. Cashback amounts by referrer condition and mint tier

All amounts below are **total cashback per referred mint** (in USDT), paid **to the referrer**.  
Split when paid out by SC4: **95% to referrer**, **5% to 5% SC**.

The **columns** are the **referrer’s condition** (who receives the cashback). The **rows** are the **tier of the card the referred user just minted**.

### Condition 1 (default) — referrer’s level

When **the referrer** is in Condition 1, they receive (for any mint tier of their referred user):

| Referred user mints | Total cashback | To referrer | To 5% SC |
|---------------------|----------------|-------------|--------------|
| Bronze              | $1             | $0.95       | $0.05        |
| Platinum            | $1             | $0.95       | $0.05        |
| Emerald             | $1             | $0.95       | $0.05        |
| Diamond             | $1             | $0.95       | $0.05        |

**Who is in Condition 1:** Every referrer starts here. No extra requirements.

---

### Condition 2 — referrer’s level

When **the referrer** is in Condition 2, they receive (amount depends on what tier the referred user just minted):

| Referred user mints | Total cashback | To referrer | To 5% SC |
|---------------------|----------------|-------------|--------------|
| Bronze              | $1             | $0.95       | $0.05        |
| Platinum            | $10            | $9.50       | $0.50        |
| Emerald             | $10            | $9.50       | $0.50        |
| Diamond             | $10            | $9.50       | $0.50        |

**Who qualifies for Condition 2 (the referrer’s level):**

1. The referrer must have minted **at least one Platinum card** (any time).
2. The referrer must have **finished their Bronze card on CLC 2** (i.e. they have a Bronze CLC1 that reached cap and had its CLC2 card auto-minted).

---

### Condition 3 — referrer’s level

When **the referrer** is in Condition 3, they receive (amount depends on what tier the referred user just minted):

| Referred user mints | Total cashback | To referrer | To 5% SC |
|---------------------|----------------|-------------|--------------|
| Bronze              | $1             | $0.95       | $0.05        |
| Platinum            | $10            | $9.50       | $0.50        |
| Emerald             | $50            | $47.50      | $2.50        |
| Diamond             | $50            | $47.50      | $2.50        |

**Who qualifies for Condition 3 (the referrer’s level):**

1. The referrer must have minted **at least one Emerald card**.
2. The referrer must have **finished their Bronze card on CLC 2**.
3. The referrer must have **finished their Platinum card on CLC 2**.

---

### Condition 4 — referrer’s level

When **the referrer** is in Condition 4, they receive (amount depends on what tier the referred user just minted):

| Referred user mints | Total cashback | To referrer | To 5% SC |
|---------------------|----------------|-------------|--------------|
| Bronze              | $1             | $0.95       | $0.05        |
| Platinum            | $10            | $9.50       | $0.50        |
| Emerald             | $50            | $47.50      | $2.50        |
| Diamond             | $100           | $95.00      | $5.00        |

**Who qualifies for Condition 4 (the referrer’s level):**

1. The referrer must have minted **at least one Diamond card**.
2. The referrer must have **finished their Bronze card on CLC 2**.
3. The referrer must have **finished their Platinum card on CLC 2**.
4. The referrer must have **finished their Emerald card on CLC 2**.

---

## 2. “Finished on CLC 2” (definition)

For a given tier (e.g. Bronze), a referrer has **finished that tier on CLC 2** when:

- They own at least one **CLC1** card of that tier (the first card they minted of that tier).
- That CLC1 card has reached its reward cap (**reward complete**).
- The contract has already created the **CLC2** card for it (**auto-minted**).

So: the referrer has progressed that tier through CLC1 and into CLC2 completion.

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

Split for every paid cashback: **95% to referrer wallet**, **5% to 5% SC (SC5)**.

---

## 4. Implementation notes

- **On-chain:** The NFT contract computes **the referrer’s** condition (1–4) by inspecting **the referrer’s** owned cards (tiers, CLC1/CLC2, reward complete, auto-minted). View function: `getReferrerCashbackLevel(referrer)`. The referred user’s condition or history does not affect the cashback amount; only the referrer’s level and the tier of the card just minted do.
- **Payment flow:** On each paid mint, the contract sends the cashback amount to SC4 (ReferralFeeHandler). If a referrer exists, SC4 sends 95% to the referrer and 5% to the 5% SC (SC5 FivePercentReceiver), or to SC2 if SC5 is not set. If no referrer exists, the cashback amount remains in SC4.
- **Backend:** When recording a referral fee (e.g. for history), the backend reads the **referrer’s** cashback level from the contract and uses the same table so stored amounts match on-chain payouts.

---

## 5. Quick reference: how the referrer reaches each condition

These requirements apply to **the referrer** (the user who receives the cashback), not to the referred user.

| Condition | Referrer must |
|-----------|----------------|
| **1**     | (default) |
| **2**     | Mint ≥1 Platinum **and** Bronze CLC2 finished |
| **3**     | Mint ≥1 Emerald **and** Bronze CLC2 **and** Platinum CLC2 finished |
| **4**     | Mint ≥1 Diamond **and** Bronze, Platinum, **and** Emerald CLC2 finished |
