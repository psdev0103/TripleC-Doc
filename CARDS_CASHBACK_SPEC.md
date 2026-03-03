# Cards Cashback — Specification

**Triple C** referral rewards are called **Cards Cashback**. When a referred user mints a card, their referrer receives a cashback amount. The amount depends on the **referrer’s qualification condition** (1–4) and the **tier of the card the referred user just minted**.  
95% of the cashback goes to the referrer’s wallet; 5% goes to the developer profit wallet (SC2).

---

## 1. Cashback amounts by condition and mint tier

All amounts below are **total cashback per referred mint** (in USDT).  
Split: **95% to referrer**, **5% to developer**.

### Condition 1 (default)

When the referred user joins with **any** card tier, the referrer gets:

| Referred user mints | Total cashback | To referrer | To developer |
|---------------------|----------------|-------------|--------------|
| Bronze              | $1             | $0.95       | $0.05        |
| Platinum            | $1             | $0.95       | $0.05        |
| Emerald             | $1             | $0.95       | $0.05        |
| Diamond             | $1             | $0.95       | $0.05        |

**Who is Condition 1:** Every referrer starts here. No extra requirements.

---

### Condition 2

When the referred user joins with **Platinum** (or higher) card, the referrer gets:

| Referred user mints | Total cashback | To referrer | To developer |
|---------------------|----------------|-------------|--------------|
| Bronze              | $1             | $0.95       | $0.05        |
| Platinum            | $10            | $9.50       | $0.50        |
| Emerald             | $10            | $9.50       | $0.50        |
| Diamond             | $10            | $9.50       | $0.50        |

**Who qualifies for Condition 2:**

1. The referrer must have minted **at least one Platinum card** (any time).
2. The referrer must have **finished their Bronze card on CLC 2** (i.e. they have a Bronze CLC1 that reached cap and had its CLC2 card auto-minted).

---

### Condition 3

When the referred user joins with **Emerald** (or higher) card, the referrer gets:

| Referred user mints | Total cashback | To referrer | To developer |
|---------------------|----------------|-------------|--------------|
| Bronze              | $1             | $0.95       | $0.05        |
| Platinum            | $10            | $9.50       | $0.50        |
| Emerald             | $50            | $47.50      | $2.50        |
| Diamond             | $50            | $47.50      | $2.50        |

**Who qualifies for Condition 3:**

1. The referrer must have minted **at least one Emerald card**.
2. The referrer must have **finished their Bronze card on CLC 2**.
3. The referrer must have **finished their Platinum card on CLC 2**.

---

### Condition 4

When the referred user joins with **Diamond** card, the referrer gets:

| Referred user mints | Total cashback | To referrer | To developer |
|---------------------|----------------|-------------|--------------|
| Bronze              | $1             | $0.95       | $0.05        |
| Platinum            | $10            | $9.50       | $0.50        |
| Emerald             | $50            | $47.50      | $2.50        |
| Diamond             | $100           | $95.00      | $5.00        |

**Who qualifies for Condition 4:**

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

| Referred mints ↓ / Condition → | Condition 1 | Condition 2 | Condition 3 | Condition 4 |
|--------------------------------|-------------|-------------|-------------|-------------|
| Bronze                          | $1          | $1          | $1          | $1          |
| Platinum                        | $1          | $10         | $10         | $10         |
| Emerald                         | $1          | $10         | $50         | $50         |
| Diamond                         | $1          | $10         | $50         | $100        |

Split for every row: **95% to referrer wallet**, **5% to developer (SC2)**.

---

## 4. Implementation notes

- **On-chain:** The NFT contract computes the referrer’s condition (1–4) by inspecting the referrer’s owned cards (tiers, CLC1/CLC2, reward complete, auto-minted). View function: `getReferrerCashbackLevel(referrer)`.
- **Payment flow:** On each mint with a referrer, the contract sends the total cashback amount to SC4 (ReferralFeeHandler). SC4 sends 95% to the referrer and 5% to SC2 (DeveloperReceiver).
- **Backend:** When recording a referral fee (e.g. for history), the backend reads the referrer’s cashback level from the contract and uses the same table so stored amounts match on-chain payouts.

---

## 5. Quick reference: how to reach each condition

| Condition | Require |
|-----------|--------|
| **1**     | (default) |
| **2**     | Mint ≥1 Platinum **and** Bronze CLC2 finished |
| **3**     | Mint ≥1 Emerald **and** Bronze CLC2 **and** Platinum CLC2 finished |
| **4**     | Mint ≥1 Diamond **and** Bronze, Platinum, **and** Emerald CLC2 finished |
