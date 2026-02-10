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
| `mintWithPayment(to, referrer, tier)` | external | Same with referrer (10% to SC4, 5% to referrer, 5% fee pool). |
| `withdrawRewards(tokenId)` | external | Withdraw CLC reward to wallet. Owner only; requires `rewardComplete`. CLC2: after withdraw, card is dismissed. |
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

### ReferralFeeHandler (SC4)

| Function | Type | Description |
|----------|------|-------------|
| `processReferralPayment(referrer, totalAmount)` | external | Only Master. Splits 50% to referrer (if not zero), rest stays as fee pool. |
| `withdrawFeePool(to, amount)` | external, owner | Withdraw fee pool USDT to `to`. |
| `setMaster(newMaster)` | external, owner | Set the Master (CustomNFT) address. |

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
