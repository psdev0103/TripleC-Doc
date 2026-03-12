# Main Flow & Cash Flow — All Cards

This document describes the **main user and system flow** and the **cash flow** (where USDT goes) for each card tier on mint.

---

## Part 1 — Main flow

### 1.1 User journey (high level)

1. **Connect wallet**  
   User connects a Web3 wallet (e.g. MetaMask) and switches to the correct network (e.g. BSC Testnet).

2. **Optional: set referrer**  
   If the user has a referral link (`?ref=0x...`), the referrer is set once (on first mint). The referrer will earn Cards Cashback when this user mints.

3. **Mint a card**  
   User chooses a tier (Bronze, Platinum, Emerald, or Diamond), optionally enters quantity (1–99), approves USDT and confirms. The contract pulls the tier price × quantity and mints that many cards. Each mint triggers the cash flow below.

4. **Card in queue**  
   Each minted card joins the global queue. New mints send a “queue amount” into the contract; that is distributed to the **oldest previous cards** (up to 1 or 5 cards depending on tier). Cards’ reward balances grow until they reach their CLC cap.

5. **CLC1 → cap reached**  
   When a card’s reward balance reaches its **CLC1 cap**, the contract:
   - Pays out the first part to the owner’s wallet (95% user, 5% to the 5% SC).
   - Uses the second part to **auto-mint a CLC2 card** for the same owner (same tier; that card is “withdraw only”, no second cycle).

6. **CLC2 → cap reached**  
   When the CLC2 card’s reward balance reaches its **CLC2 cap**, the contract pays out to the owner’s wallet (95% / 5% split again), then the card is **dismissed** (no more rewards).

7. **Withdraw (optional)**  
   If a card is complete and there is remaining balance (e.g. final chunk not yet sent), the owner can call **Withdraw** to receive it. After that, the card shows as Withdrawn or Dismissed.

8. **Points & history**  
   - **Loyalty & Level points** are credited in SC3 when the user mints (CLC1 only; CLC2 does not add points).  
   - **Cards Cashback history** shows USDT the user received as a referrer when their referrals minted (95% of cashback per mint).

---

### 1.2 System flow (what happens on each paid mint)

```
User approves USDT → mintWithPayment(to, referrer, tier)
  │
  ├─ Pull tier price from user (e.g. $10 / $100 / $500 / $1000)
  │
  ├─ Split (see Part 2):
  │    • Developer (SC2)     — fixed $ per tier
  │    • Loyalty & Level (SC3) — fixed $ per tier
  │    • Cards Cashback (SC4)  — depends on referrer condition (1–4) and tier
  │    • Queue amount         — into contract reserve, then:
  │         – First mint ever: part to Overlap (SC1), rest to reserve
  │         – Later mints: distribute to up to 1 or 5 previous cards (oldest first)
  │                         per card = minting tier’s rewardToPrev; overflow → SC1
  │
  ├─ New card minted (tokenId), reward balance may get “self” share from queue
  │
  └─ If any previous card reached cap during this distribution:
       → Payout to that card’s owner (95% / 5%)
       → If CLC1: optionally auto-mint CLC2 for owner (from reserve)
       → On CLC2 mint: flow in row queue (see 3.4) + 95% of flow to SC1b + 5% of flow to POINTS SC (SC3)
```

---

### 1.3 Card life cycle (CLC) in one diagram

```
[User mints] → CLC1 card created
                    ↓
              Reward balance grows (from later mints’ queue distribution)
                    ↓
              Balance reaches CLC1 cap
                    ↓
              Payout to wallet (95% user, 5% SC5) + CLC2 card auto-minted
                    ↓
              CLC2 card reward balance grows
                    ↓
              Balance reaches CLC2 cap
                    ↓
              Payout to wallet (95% user, 5% SC5) → card dismissed
```

---

## Part 2 — Cash flow for all cards

Every **paid mint** splits the tier price as below. Amounts are in **USDT per card** (one mint of that tier).

**Cards Cashback** is variable: it depends on the **referrer’s qualification condition** (1–4). See [CARDS_CASHBACK_SPEC.md](CARDS_CASHBACK_SPEC.md) for the exact table. The figures in the “Cards Cashback” column below are the **maximum** (Condition 4); lower conditions give smaller amounts.

**First mint in the system:** For the very first card ever minted, the “Queue” column amount is not fully distributed (there are no previous cards). The **first-mint overlap** in the last column is sent to **SC1 OverlapReceiver** instead; the rest stays in contract reserve for future queue payouts.

---

### 2.1 Bronze — $10 per card

| Destination        | Amount (USDT) | Note |
|--------------------|---------------|------|
| **Queue**          | $5            | Distributed to **1** previous card (oldest), $5 each; overflow → SC1 |
| **Cards Cashback** | $1            | Sent to SC4; when paid out: 95% referrer, 5% SC5 |
| **Loyalty & Level** | $1.75         | To SC3 (LoyaltyLevelVault) |
| **Developer**      | $2.25         | To SC2 (DeveloperReceiver) |
| **First-mint overlap** | $5         | Only when no cards exist yet; goes to SC1 |

**CLC:** CLC1 cap $20 ($10 to wallet, $10 to mint CLC2). CLC2 cap $10 ($10 to wallet, then dismissed).

---

### 2.2 Platinum — $100 per card

| Destination        | Amount (USDT) | Note |
|--------------------|---------------|------|
| **Queue**          | $50           | Distributed to **5** previous cards (oldest first), $10 each; overflow → SC1 |
| **Cards Cashback** | $1–$10        | Depends on referrer condition; sent to SC4, then 95% referrer / 5% SC5 on payout |
| **Loyalty & Level** | $13          | To SC3 |
| **Developer**      | $27           | To SC2 |
| **First-mint overlap** | $10       | Only when no cards exist yet; goes to SC1 |

**CLC:** CLC1 cap $200 ($100 to wallet, $100 to mint CLC2). CLC2 cap $100 ($100 to wallet, then dismissed).

---

### 2.3 Emerald — $500 per card

| Destination        | Amount (USDT) | Note |
|--------------------|---------------|------|
| **Queue**          | $250          | Distributed to **5** previous cards, $50 each; overflow → SC1 |
| **Cards Cashback** | $1–$50        | Depends on referrer condition; sent to SC4, then 95% referrer / 5% SC5 on payout |
| **Loyalty & Level** | $63          | To SC3 |
| **Developer**      | $137          | To SC2 |
| **First-mint overlap** | $250      | 5×$50 when no cards exist yet; goes to SC1 |

**CLC:** CLC1 cap $1125 ($625 to wallet, $500 to mint CLC2). CLC2 cap $625 ($625 to wallet, then dismissed).

---

### 2.4 Diamond — $1000 per card

| Destination        | Amount (USDT) | Note |
|--------------------|---------------|------|
| **Queue**          | $500          | Distributed to **5** previous cards, $100 each; overflow → SC1 |
| **Cards Cashback** | $1–$100       | Depends on referrer condition; sent to SC4, then 95% referrer / 5% SC5 on payout |
| **Loyalty & Level** | $125.5       | To SC3 |
| **Developer**      | $274.5        | To SC2 |
| **First-mint overlap** | $500     | 5×$100 when no cards exist yet; goes to SC1 |

**CLC:** CLC1 cap $2500 ($1500 to wallet, $1000 to mint CLC2). CLC2 cap $1500 ($1500 to wallet, then dismissed).

---

## Part 3 — Summary tables

### 3.1 Fixed amounts per tier (USDT per mint)

| Tier     | Price | Queue | Loyalty & Level (SC3) | Developer (SC2) | Flow to previous cards |
|----------|-------|-------|------------------------|-----------------|-------------------------|
| Bronze   | $10   | $5    | $1.75                  | $2.25           | $5 × 1 card             |
| Platinum | $100  | $50   | $13                    | $27             | $10 × 5 cards           |
| Emerald  | $500  | $250  | $63                    | $137            | $50 × 5 cards           |
| Diamond  | $1000 | $500  | $125.5                 | $274.5          | $100 × 5 cards          |

### 3.2 CLC caps and payouts (USDT)

| Tier     | CLC1 cap | CLC1 → wallet | CLC1 → CLC2 mint | CLC2 cap | CLC2 → wallet |
|----------|----------|---------------|-------------------|----------|----------------|
| Bronze   | $20      | $10           | $10               | $10      | $10            |
| Platinum | $200     | $100          | $100              | $100     | $100           |
| Emerald  | $1125    | $625          | $500              | $625     | $625           |
| Diamond  | $2500    | $1500         | $1000             | $1500    | $1500          |

### 3.3 First-mint overlap (when no cards exist)

| Tier     | Amount to SC1 Overlap |
|----------|------------------------|
| Bronze   | $5                     |
| Platinum | $50 (5×$10)            |
| Emerald  | $250 (5×$50)           |
| Diamond  | $500 (5×$100)          |

### 3.4 When a CLC2 card is generated

When a CLC1 card reaches cap and a **CLC2 card is auto-minted**, the following split applies (from the CLC2 funding amount = flow in row queue for that tier):

| Tier     | Flow in row queue       | To CLC2 Overlap SC (SC1b) | To POINTS SC (SC3) |
|----------|--------------------------|----------------------------|---------------------|
| Bronze   | $5                       | $5 × 0.95 = $4.75         | $5 × 0.05 = $0.25   |
| Platinum | $50 ($10 × 5)            | $50 × 0.95 = $47.50       | $50 × 0.05 = $2.50  |
| Emerald  | $250 ($50 × 5)           | $250 × 0.95 = $237.50     | $250 × 0.05 = $12.50 |
| Diamond  | $500 ($100 × 5)          | $500 × 0.95 = $475        | $500 × 0.05 = $25   |

- **Flow in row queue:** Distributed to the oldest previous cards (same rules as a paid mint).
- **CLC2 Overlap SC (SC1b):** OverlapReceiver2 receives **95%** of the flow amount; any unallocated queue remainder also goes to SC1b.
- **POINTS SC (SC3):** LoyaltyLevelVault receives **5%** of the flow amount per CLC2 card generated.

---

## Part 4 — Contract roles (cash flow)

| Contract / destination | Role in cash flow |
|------------------------|-------------------|
| **User wallet**        | Pays tier price on mint; receives 95% of CLC payouts and of Cards Cashback (when they are the referrer). |
| **CustomNFT (Master)** | Holds queue reserve; distributes to previous cards; performs CLC payouts and CLC2 mint; sends shares to SC1, SC1b, SC2, SC3, SC4. |
| **SC1 OverlapReceiver** | Receives queue overflow and first-mint overlap (CLC1). |
| **SC1b OverlapReceiver2** | Receives 95% of the CLC2 flow amount (queue amount for that tier) and any CLC2 queue remainder when a CLC2 card is generated. |
| **SC2 DeveloperReceiver** | Receives the fixed developer/team profit amount per mint. |
| **SC5 FivePercentReceiver (5% SC)** | Receives only the 5% part of payments sent to user wallets: 5% of CLC payouts from Master and 5% of cashback payouts from SC4. |
| **SC3 LoyaltyLevelVault** | Receives loyalty/level amount per paid mint; receives 5% of the CLC2 flow amount per CLC2 card generated (POINTS SC); holds USDT; credits points. |
| **SC4 ReferralFeeHandler** | Receives Cards Cashback amount on every paid mint. If a referrer exists, SC4 sends 95% to the referrer and 5% to SC5. If no referrer exists, the cashback amount stays in SC4. |
