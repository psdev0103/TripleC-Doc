# Triple C — Video Walkthrough Script

Use this as a script to record a video that describes all functions of the project. Each section is a natural “scene” you can narrate.

---

## 1. Introduction (30 sec)

- **Say:** “This is Triple C — Hope and Dream Card Chain. It’s a Web3 dApp on BSC: users mint tiered NFT cards with USDT, earn rewards through a queue, and the system splits each mint into referral fees, loyalty points, developer profit, and queue overflow.”
- **Show:** Browser with the app open (header: “Triple C — Hope & Dream Card Chain”), wallet disconnected.

---

## 2. Connect Wallet & Network (1 min)

- **Say:** “Users connect with MetaMask or any Web3 wallet. The app targets BSC Testnet by default; it will prompt to switch chain if needed.”
- **Do:** Click “Connect Wallet”, approve in MetaMask, ensure you’re on the correct network.
- **Say:** “Once connected, we see the wallet address and a Disconnect button. The rest of the page loads: loyalty points, card tiers, contract stats, and my cards.”

---

## 3. Loyalty Point Reward Program (1 min)

- **Say:** “At the top we have the Loyalty Point Reward Program. Points are stored in a separate contract — SC3 LoyaltyLevelVault. Only the first card cycle, CLC1, earns points; the auto-created CLC2 card does not.”
- **Show:** “Your points (SC3)” — Loyalty and Level values; “Refresh points” button.
- **Show:** “Loyalty Point Reward Conversion” — points per tier (Bronze 10, Platinum 100, etc.) and the base USDT conversion table for redemption.

---

## 4. Card Tiers & Minting (2 min)

- **Say:** “There are four tiers: Bronze, Platinum, Emerald, Diamond. Each has a mint price in USDT, ROI, and CLC caps. The table also shows how each mint is split: Queue, Referral, Loyalty and Level, and Developer.”
- **Show:** The four tier cards with price, ROI, CLC1/CLC2 caps, and Queue / Ref / LP / Dev amounts.
- **Say:** “To mint, we need USDT approved. The app requests approval if needed, then calls mintWithPayment. That pulls USDT from the user and splits it: part goes to the queue for previous cards, part to referral, loyalty vault, and developer contracts. Any queue overflow goes to the Overlap contract.”
- **Do:** (Optional) Mint one card — show approval and mint flow; mention that after mint, total supply and “My Minted Cards” update.

---

## 5. Cards Life Cycle (CLC) (1 min)

- **Say:** “Every card is locked by CLC. CLC1 is the first card: when its reward balance reaches the cap, the contract automatically sends the first payout to the wallet and uses the second part to mint a CLC2 card for the same owner. CLC2 is the auto-created card: when it reaches its cap, the contract pays out to the wallet and then the card is dismissed — no further rewards.”
- **Show:** The “Cards Life Cycle” section and the short explanation on the page.

---

## 6. Contract Stats & My Minted Cards (1 min)

- **Say:** “Contract Stats shows total supply of cards. My Minted Cards lists the current user’s cards: token ID, tier, CLC1 or CLC2, reward balance, and status — In progress, Complete, or Withdrawn/Dismissed.”
- **Show:** Total supply and the list of cards with “Withdraw” where applicable. **Say:** “If the card is complete and has a balance, the user can click Withdraw to call withdrawRewards and receive USDT. After full withdraw, we show Withdrawn or Dismissed and hide the button.”

---

## 7. Cash Flow & SC Balances (1.5 min)

- **Say:** “To test the smart contract flow in detail, we have a Cash Flow section. It shows the USDT balance held in each of the four side contracts.”
- **Show:** “Refresh SC Balances” and the four cards: Overlap (queue overflow), Developer & Team profit, Loyalty Points & Level Points, Referral fee & Withdraw fee pool.
- **Say:** “These balances update after every mint. Queue overflow goes to Overlap; the referral share goes to SC4 (split between referrer and fee pool); loyalty and level share goes to SC3; developer share goes to SC2. Refreshing here lets you verify the split on-chain.”

---

## 8. All Cards (1 min)

- **Say:** “The All Cards section is for testing and support. Click ‘Load All Cards’ to fetch every minted card across all users.”
- **Do:** Click “Load All Cards”. **Show:** The table: Token ID, Owner, Tier, CLC (CLC1/CLC2), Reward balance, Status.
- **Say:** “This helps verify queue order, caps, and status of every card without scanning the blockchain manually.”

---

## 9. Smart Contracts Summary (1 min)

- **Say:** “The mint flow sits on CustomNFT—the master contract: minting, queue distribution, overflow, payouts, and wired sends to SC1, SC2, SC3, SC4, SC5 where configured. SC1 catches queue overflow that doesn’t attach to older cards; SC2 is developer/share; SC3 Loyalty Level Vault receives the loyalty legs and credits points plus holds USDT that can fund CCC swaps. SC4 Cards Cashback receives the referrer deposit—when eligible it sends five percent to the five-percent contract and ninety-five percent is pushed through Master to the referrer’s cards in queue, or retained in SC4 if there’s nowhere to attach it.”
- **Say:** “There are also CCC Hub contracts—CCCToken and CCC Platform—outside that mint diagram: redeem points from SC3 into CCC and optional swap CCC to USDT. Staking earns daily reward CCC via claim rewards; unstake isn’t exposed in our current hub design.” *(See [CCC_HUB_SPEC.md](CCC_HUB_SPEC.md) / [SYSTEM_DIAGRAM.md](SYSTEM_DIAGRAM.md).)*
- **Optional:** Show repo or a diagram (see SYSTEM_DIAGRAM.md).

---

## 10. Outro (20 sec)

- **Say:** “That’s Triple C: tiered card minting, CLC-based auto-payouts, queue and overflow, plus referral, loyalty, and developer splits — all testable from this frontend. Thanks for watching.”
- **Show:** Full page or logo if you have one.

---

**Total suggested length:** ~10–12 minutes if you narrate CCC Hub; shorten any section as needed.
