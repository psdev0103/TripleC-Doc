# Triple C — Function Reference

Quick reference for all contract and frontend functions.

---

## Smart contracts

### CustomNFT (Master)

| Function | Type | Description |
|----------|------|-------------|
| `safeMint(to, tier)` | external, owner | Mint a card without payment (internal use). |
| `safeMint(to, referrer, tier)` | external, owner | Same with referrer. |
| `mintWithPayment(to, tier)` | external | User pays USDT; mint card. Splits to queue, SC2, SC3, SC4; overflow to SC1. |
| `mintWithPayment(to, referrer, tier)` | external | Same with referrer. Cards Cashback to SC4 (5% to SC5; 95% onto referrer’s CLC1/CLC2 cards in queue, or retained in SC4 if none); amount depends on referrer's qualification level (1–4). |
| `withdrawRewards(tokenId)` | external | Withdraw CLC reward to wallet. Owner only; requires `rewardComplete`. CLC2: after withdraw, card is dismissed. |
| `claimCLC2(tokenId)` | external | Owner only. Mint deferred CLC2 card when reserve was insufficient at completion (batch mint). |
| `getReferrerCashbackLevel(referrer)` | view | Returns referrer's Cards Cashback qualification level (1–4). Condition 2 = has Platinum + Bronze CLC2 done; 3 = Emerald + Bronze & Platinum CLC2 done; 4 = Diamond + all lower CLC2 done. |
| `creditReferralCashbackFromSc4(referrer, amount)` | external | Only SC4. After SC4 transfers `amount` USDT to Master, applies Cards Cashback to the referrer’s CLC1/CLC2 queue (ascending `tokenId`). |
| `giftClc1ReferralMainFlowUnlocked(tokenId)` | view | **Gift CLC1:** always **`true`** in current logic (legacy name; cashback uses Diamond routing from mint). |
| `giftClc1LoyaltyCapReductionWei(tokenId)` | view | Deprecated; returns **0** (no cap reduction). |
| `referralDiamondMintCountForUpline(upline)` | view | Paid **Diamond CLC1** mints by referred users (upline resolution per gift spec), capped at **3**; gates Gift SC payouts. |
| `giftLoyaltyMilestoneApplied(giftOwner)` | view | Deprecated; always **`false`** (milestone removed). |
| `totalSupply()` | view | Number of tokens minted. |
| `tierOf(tokenId)` | view | Tier (0–3) of the card. |
| `rewardBalance(tokenId)` | view | Current reward balance (USDT) on the card. |
| `rewardComplete(tokenId)` | view | True when card reached cap (ready for withdraw / auto-payout done). |
| `isSecondCycleCard(tokenId)` | view | True = CLC2 (auto-created) card. |
| `ownerOf(tokenId)` | view | ERC721 owner. |
| `setBaseURI(uri)` | external, owner | Set base token URI. |
| `setPayout(addr)` | external, owner | Set payout address. |
| `setOverlapReceiver(addr)` | external, owner | Set SC1 (OverlapReceiver). |
| `setSc2Developer(addr)` | external, owner | Set SC2 (DeveloperReceiver). |
| `setSc3Loyalty(addr)` | external, owner | Set SC3 (LoyaltyLevelVault). |
| `setSc4Referral(addr)` | external, owner | Set SC4 (ReferralFeeHandler). |
| `setGiftCardReceiver(addr)` | external, owner | Set Gift Card SC (GiftCardReceiver). |
| `safeMintGift(to)` | external | Mint a gift card to `to`. Reverts if `balanceOf(to) != 0`. No payment; gift minter only. |
| `safeMintGift(to, referrer)` | external | Same; optional `referrer` sets `referredBy[to]` when unset (like first paid mint), so upline earns Loyalty Points when `to`’s downline mints. Use when the recipient may only ever hold the gift card. |

**Internal / private (called during mint or reward):**  
`_processMint`, `_distributeToPrevTokens`, `_accrueReward`, `_maybeAutoMint`, `_getRewardCap`, `_getWithdrawAmount`, `_setTierConfigs`, `_setReferrer`, `_loyaltyLevelPointsForTier`.

---

### OverlapReceiver (SC1)

| Function | Type | Description |
|----------|------|-------------|
| `onOverlapReceived(token, from, amount)` | external | Called when overlap USDT is sent; emits event (no transfer logic; Master sends USDT directly). |
| `withdrawToken(token, to, amount)` | external, owner | Withdraw received ERC20 to `to`. |

---

### DeveloperReceiver (SC2)

| Function | Type | Description |
|----------|------|-------------|
| `withdrawToken(token, to, amount)` | external, owner | Withdraw Developer & Team profit (USDT) to `to`. |

---

### LoyaltyLevelVault (SC3)

| Function | Type | Description |
|----------|------|-------------|
| `creditPoints(user, loyaltyDelta, levelDelta)` | external | Only Master. Credits loyalty and level points to `user`. |
| `loyaltyPoints(user)` | view | Loyalty points for `user`. |
| `levelPoints(user)` | view | Level points for `user`. |
| `setMaster(newMaster)` | external, owner | Set the Master (CustomNFT) address. |
| `withdrawToken(token, to, amount)` | external, owner | Withdraw ERC20 (e.g. excess USDT) to `to`. |

---

### ReferralFeeHandler (SC4) — Cards Cashback

| Function | Type | Description |
|----------|------|-------------|
| `processReferralPayment(referrer, totalAmount)` | external | Only Master. Called after Master sends the tier’s referral USDT to SC4; SC4 forwards `totalAmount` to Master and Master runs `creditReferralCashbackFromSc4` for the referrer’s queue (see [CARDS_CASHBACK_SPEC.md](CARDS_CASHBACK_SPEC.md)). |
| `withdrawToken(to, amount)` | external, owner | Withdraw any USDT held in SC4 (e.g. when no referrer was set) to `to`. |
| `setMaster(newMaster)` | external, owner | Set the Master (CustomNFT) address. |

---

### GiftCardReceiver (Gift Card SC)

| Function | Type | Description |
|----------|------|-------------|
| `onGiftCLC1CapReached(beneficiary)` | external | Only Master. Sets `giftClc1CapReached` after CLC1 cap finalize ($1000 received). |
| `onGiftCLC2CapReached(beneficiary)` | external | Only Master. Sets `giftClc2CapReached` after gift CLC2 cap finalize ($1000 received). |
| `payoutGiftClc1Bonus(giftCardUser)` | external | Owner only. **$1000** when `referralDiamondMintCountForUpline >= 3` and CLC1 cap recorded; not already paid. |
| `payoutGiftClc2Bonus(giftCardUser)` | external | Owner only. **$1000** when `referralDiamondMintCountForUpline >= 3` and CLC2 cap recorded; not already paid. |
| `payoutBothConditionsMet(giftCardUser)` | external | Owner only. Legacy: **$2000** in one transfer when allowed by older state (reverts if split bonuses used). |
| `setMaster(newMaster)` | external, owner | Set the Master (CustomNFT) address. |
| `setPaymentToken(token)` | external, owner | Set USDT token address. |
| `withdrawToken(token, to, amount)` | external, owner | Withdraw ERC20 to `to`. |

---

### Payment token (ERC20 / USDT)

| Function | Type | Description |
|----------|------|-------------|
| `approve(spender, amount)` | external | Approve CustomNFT to spend USDT for mint. |
| `balanceOf(account)` | view | USDT balance of `account` (used for SC1–SC4 balances in frontend). |

---

## Frontend (TripleC-Frontend)

### Web3Context

| Function / export | Description |
|-------------------|-------------|
| `connect()` | Request wallet connection; switch to app chain if needed. |
| `disconnect()` | Clear account, chain, provider, signer. |
| `getCustomNFTContract(useSigner)` | Return CustomNFT contract instance (read or write). |
| `getPaymentTokenContract(useSigner)` | Return payment token (ERC20) contract instance. |
| `getLoyaltyLevelContract(useSigner)` | Return LoyaltyLevelVault (SC3) contract instance. |
| `account` | Current wallet address (or null). |
| `chainId` | Current chain ID. |
| `isConnected` | True when account is set. |
| `error`, `isConnecting` | Connection error message and loading state. |

### App.jsx (main UI)

| Handler / fetch | Description |
|------------------|-------------|
| `fetchTotalSupply()` | Load `CustomNFT.totalSupply()` into state. |
| `fetchMyCards()` | Load current user’s cards (ownerOf + tier, rewardBalance, rewardComplete, isSecondCycleCard). |
| `fetchAllCards()` | Load all cards (all tokenIds 0..totalSupply-1) for “All Cards” table. |
| `fetchScBalances()` | Load USDT balance in SC1, SC2, SC3, SC4 via `balanceOf(scAddress)`. |
| `fetchLoyaltyPoints()` | Load SC3 `loyaltyPoints(account)` and `levelPoints(account)`. |
| `handleMint(tier)` | Approve USDT if needed, then `mintWithPayment(account, tier)`. |
| `handleWithdraw(tokenId)` | Call `withdrawRewards(tokenId)` for the connected user. |

### Config (contracts.js)

| Export | Description |
|--------|-------------|
| `CONTRACT_ADDRESSES` | By chainId: CustomNFT, PaymentToken, LoyaltyLevelVault, Sc1Overlap, Sc2Developer, Sc4Referral. |
| `SC_LABELS` | Display names for cash flow (Overlap, Developer, Loyalty/Level, Referral). |
| `CARD_TIERS` | Tier names, prices, ROI, CLC caps, queue/referral/loyalty/developer amounts. |
| `LOYALTY_POINTS_PER_TIER` | Points earned per tier (CLC1 only). |
| `LOYALTY_POINT_CONVERSION` | Points-to-USDT conversion table. |

---

**Deploy (SC):**  
`npx hardhat run scripts/deploy-testnet.js --network binanceTestnet`  
Then set `VITE_*` in TripleC-Frontend `.env` from the script output.
