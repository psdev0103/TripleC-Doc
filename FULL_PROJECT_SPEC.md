# Triple C — Full Project Specification

Single reference for **all repositories under `TripleC`**: layout, environments, smart-contract economics, CCC Hub, frontend gates, backend API, and deployment scripts. Detailed narratives remain in the linked docs at the end.

---

## 1. Repository layout

| Path | Role |
|------|------|
| **`TripleC-Frontend/`** | Vite + React 19 SPA; wallet connect; mint / withdraw / loyalty / CCC Hub; reads chain state via ethers.js v6. |
| **`TripleC-Backend/`** | Express + MySQL; REST API for users, cards, loyalty cache, referrals, referral fees, rewards withdrawn; optional chain sync scripts. |
| **`SC/`** | Hardhat project: CustomNFT (Master), receivers SC1–SC5, GiftCardReceiver, upgradeable **LoyaltyLevelVault**, **CCCToken**, **CCCPlatform** (non-proxy); deploy & upgrade scripts. |
| **`deploy/`** | EC2 deploy helpers (PowerShell, server notes); domain docs. |
| **`docs/`** | Product & technical markdown (this file + flow, cashback, gift card, function reference, diagrams). |

There is **no separate `SC3` Node service folder** in this repo; “SC3” means the **LoyaltyLevelVault** contract on-chain.

---

## 2. Networks & configuration

### 2.1 Chain IDs

| Chain ID | Network |
|---------|---------|
| **56** | BSC Mainnet |
| **97** | BSC Testnet |

### 2.2 Frontend (`TripleC-Frontend`)

- **Default chain:** `VITE_CHAIN_ID` in `.env`; if unset or invalid, **`src/config/chains.js`** defaults to **56**.
- **Contract addresses:** `src/config/contracts.js` — objects `TESTNET` (97) and `MAINNET` (56). Keys include: `CustomNFT`, `PaymentToken`, `LoyaltyLevelVault`, `Sc1Overlap`, `Sc1bOverlap2`, `Sc2Developer`, `Sc4Referral`, `Sc5FivePercent`, `GiftCardReceiver`, `MasterWallet`, `DoubleSignA`, `CustomNFTPaymentSplitLib`, **`CCCToken`**, **`CCCPlatform`**.
- **Sync from deploy JSON:** `SC/scripts/update-configs-from-deployment.js` (see SC `package.json` / deploy output).
- **Backend URL:** `VITE_API_URL` (optional).

### 2.3 Backend (`TripleC-Backend`)

- **Chain selection:** `CHAIN_ID` in `.env` (typically 56 or 97).
- **RPC:** `RPC_URL` for on-demand reads / sync scripts.
- **Addresses:** `config/contracts.js` — `nftContractAddress`, `loyaltyContractAddress`, `paymentTokenAddress`, SC1/SC1b/SC2/SC4/SC5, `giftCardReceiverAddress`, `masterWalletAddress`, `doubleSignWalletAddress`.  
- **Note:** CCC token and CCC platform addresses are **not** duplicated in backend config today; CCC is frontend- and chain-driven.

---

## 3. Smart contracts — card system (summary)

### 3.1 CustomNFT (Master)

- Entry: **`mintWithPayment(to, referrer, tier)`** (USDT paid by user).
- Splits tier price into: **queue** (reserve → previous cards), **SC2** fixed developer share, **SC3** loyalty USDT share + on-chain **loyalty/level points** (CLC1 mints; see vault), **SC4** Cards Cashback deposit, **SC1** for unallocated queue / overflow / first-mint overlap, **SC1b** fixed on **CLC2 auto-mint**, **SC5** on user-facing payouts (95% user / 5% SC5 when configured).
- **Gift cards**, **CLC caps**, **queue math**, and **cashback conditions** are documented in [MAIN_FLOW_AND_CASHFLOW.md](MAIN_FLOW_AND_CASHFLOW.md), [CARDS_CASHBACK_SPEC.md](CARDS_CASHBACK_SPEC.md), [GIFT_CARD_AND_RAFFLE_COUPON.md](GIFT_CARD_AND_RAFFLE_COUPON.md).

### 3.2 LoyaltyLevelVault (SC3) — points & CCC liquidity

**Upgradeable** (OpenZeppelin upgradeable pattern).

| Responsibility | Detail |
|----------------|--------|
| **USDT** | Receives loyalty share from Master on mints; holds balance usable for CCC→USDT swaps. |
| **Points** | `loyaltyPoints`, `levelPoints`; **`creditPoints`** from Master **or** allowlisted **`loyaltyPointCreditors`** (e.g. `CustomNFTPaymentSplitLib`). |
| **Point burn** | **`debitTotalPoints(user, totalBurn)`** — loyalty balance burned first, then level; callers: Master **or** allowlisted **`loyaltyPointConsumers`** (CCC platform). |
| **CCC swap pull** | **`cccPlatformSwapPuller`** — single address (CCCPlatform). **`pullUsdtForCccSwap(token, to, amount)`** transfers USDT from vault to swap recipient. Owner sets via **`setCccPlatformSwapPuller`**. |

### 3.3 CCCToken

- **ERC20**, **6 decimals**, fixed supply **100,000,000 CCC** minted to deployer at construction.
- Platform pulls CCC from its own balance for points redemption; staking rewards paid from platform CCC balance (**must be funded** for rewards).

### 3.4 CCCPlatform (non-proxy)

Immutable deployment; wired at constructor to **CCC**, **USDT**, **LoyaltyLevelVault**.

| Constant / behavior | Value |
|---------------------|--------|
| Point step | Multiples of **10** points only |
| Points → CCC | **10 points → 3 CCC** → **1,000 points → 300 CCC** (`CCC_PER_10_POINTS` = 3e6 smallest units) |
| CCC → USDT | **1 CCC = $0.05 USDT** (after 5% burn on input CCC) |
| Swap / stake tax | **5%** of CCC (`TAX_BPS = 500`) sent to burn address **`0x…dEaD`** |
| Staking APR | **0.8% per day** on **net** staked amount (`STAKE_DAILY_BPS = 80`) |
| Staked principal | **No user `unstake`** in current source — principal stays on **CCCPlatform**; users only withdraw **reward** CCC via **`claimStakeRewards`**. See [CCC_HUB_SPEC.md](CCC_HUB_SPEC.md). |
| USDT source for swap | **Not** platform balance — **`pullUsdtForCccSwap`** on SC3 |

**Main functions:** `previewPointsToCcc`, `swapPointsForCcc`, `swapCccForUsdt`, `stake`, `claimStakeRewards`, `pendingStakeRewards`, `rescueErc20` (owner).

### 3.5 On-chain ops checklist (CCC)

After deploying or changing **CCCPlatform**:

1. **`LoyaltyLevelVault.setLoyaltyPointConsumer(CCCPlatform, true)`** — allow point debits for points→CCC.  
2. **`LoyaltyLevelVault.setCccPlatformSwapPuller(CCCPlatform)`** — allow USDT pull for swaps.  
3. Ensure **SC3 holds enough USDT** for expected CCC→USDT volume.  
4. **Fund CCCPlatform** with CCC for redemptions and staking reward pool.

Full narrative: **[CCC_HUB_SPEC.md](CCC_HUB_SPEC.md).

Hardhat helpers in `SC/package.json`: `sc3:ccc-consumer:*`, `deploy:ccc:*`, `deploy:ccc-platform:*`.  
If replacing an old platform, run the consumer script with **`OLD_CCC_PLATFORM`** set to revoke the previous consumer (see `scripts/set-sc3-ccc-consumer.js`).

### 3.6 Product notes (off-chain / future on-chain)

- **Card-queue caps:** Copy in i18n describes **policy** (earnable headroom reduced in line with swap economics); **automatic on-chain enforcement on CustomNFT** is deferred (EIP-170 size limits; may require separate library/upgrade).

---

## 4. Economics — quick tables

### 4.1 Card mint (CLC1) — fixed USDT splits (per card)

Aligned with [MAIN_FLOW_AND_CASHFLOW.md](MAIN_FLOW_AND_CASHFLOW.md) §2.0.

| Tier | Price | Queue | SC3 loyalty USDT | SC2 developer | Cards Cashback to SC4 (max cond. 4) |
|------|-------|-------|-------------------|---------------|-------------------------------------|
| Bronze | $10 | $5 | $1.75 | $2.25 | $1 |
| Platinum | $100 | $50 | $13 | $27 | up to $10 |
| Emerald | $500 | $250 | $63 | $137 | up to $50 |
| Diamond | $1000 | $500 | $125.5 | $274.5 | up to $100 |

### 4.2 CLC2 auto-mint — fixed legs

Per tier: **SC2** and **SC1b** fixed amounts; **SC3** receives **$0.50** on CLC2 generation (not “5% of queue”).

### 4.3 CCC Hub

| Rule | Detail |
|------|--------|
| Points → CCC | **1,000 POINT** total (loyalty + level) → **300 CCC** |
| CCC → USDT | **5%** CCC burn; remainder at **$0.05 / CCC** |
| Stake | **5%** burn on stake; **0.8%/day** on net stake |
| Stake principal exit | **Not** callable by users in current bytecode — no **`unstake`**; rewards only via **`claimStakeRewards`**.

---

## 5. Frontend specification (`TripleC-Frontend`)

### 5.1 Stack

React 19, Vite 7, ethers.js 6, Tailwind 4, react-i18next; ABIs under `src/abis/`.

### 5.2 Routes / major UI

- **Home:** radial menu; **Points Swap** spoke links **`/ccc-hub`** (inside the page: admins full UI, others Coming Soon — §5.3).
- **`/ccc-hub`:** **`CccHub.jsx`** — points redeem, CCC/USDT swap preview, **stake + claim rewards** (no unstake in UI for current hub); shows **SC3 USDT balance** as swap liquidity when vault address is configured.
- **Gating:** **`CccHubGate`** always mounts **`CccHub`**. **`useCccHubNavAccess`** selects full UI vs Coming Soon (**MasterWallet**, **`VITE_TRUSTED_DEPLOYER_WALLETS`**, **`EXTRA_FULL_ADMIN_NAV_WALLETS`**, or on-chain **`CustomNFT.owner()`** / **`initialDeployer()`**); see **`adminContractWallets.js`**.

### 5.3 Points Swap / CCC Hub visibility

- **`/ccc-hub`:** **Admins** see the full hub (same rules as nav access: **MasterWallet**, **`VITE_TRUSTED_DEPLOYER_WALLETS`**, **`EXTRA_FULL_ADMIN_NAV_WALLETS`**, or on-chain **`CustomNFT.owner()`** / **`initialDeployer()`**). **Non-admins** see localized **Coming Soon** (`home.pointsSwapComingSoon`) on **testnet and mainnet** (development builds use the same rule). While the wallet is connected and on-chain roles are still loading, **`ccc.accessChecking`** is shown briefly.
- Home radial **Points Swap** always links to **`/ccc-hub`**; the gate is **inside** the page.

**Deprecated note:** Previously, “Coming Soon” applied only to production mainnet by build flag; this is replaced by the **admin wallet** check above.

### 5.4 i18n

Locale files under `src/locales/*.json`; CCC copy in `ccc.*` and `home.pointsSwapComingSoon`.

---

## 6. Backend specification (`TripleC-Backend`)

### 6.1 Stack & run

Node (ES modules), Express, MySQL2, ethers v6. Default port **3001** (or `PORT`).

### 6.2 Database (high level)

Tables: **users**, **cards**, **loyalty_points**, **referrals**, **referral_fees**, **rewards_withdrawn**.  
Card **tier** stored as integer **0–3** (Bronze–Diamond).

### 6.3 API (representative)

- `GET /health`  
- Users: `GET/POST/PATCH /api/users`, `GET /api/users/:wallet`  
- Cards: `GET /api/cards`, `GET/PATCH .../token/:tokenId`, `POST /api/cards`  
- Loyalty: `GET /api/loyalty`, `PUT /api/loyalty/:wallet`  
- Referrals, referral fees, rewards withdrawn — see [TripleC-Backend/README.md](../TripleC-Backend/README.md).

### 6.4 Scripts (`npm run`)

| Script | Purpose |
|--------|---------|
| `db:init` | Apply schema |
| `loyalty:sync` | `scripts/add-loyalty-for-wallet.js` — upsert loyalty from chain for one wallet |
| `sync:cards` | Full card sync from chain |
| `sync:all` | Broader backfill after DB reset |
| `backfill:reward-sources` | Optional reward metadata backfill |

---

## 7. SC project (`SC/`)

### 7.1 Tooling

Hardhat ^2.22, Solidity **^0.8.27**, OpenZeppelin contracts + upgradeable plugin.

### 7.2 Deploy / ops scripts (npm)

| Pattern | Scripts |
|---------|---------|
| Full deploy | `deploy:mainnet`, `deploy:testnet`, continue/resume variants |
| Upgrades | `upgrade:mainnet`, `upgrade:testnet`, `upgrade:*:all` |
| CCC | `deploy:ccc:testnet`, `deploy:ccc:mainnet`, `deploy:ccc-platform:testnet`, `deploy:ccc-platform:mainnet` |
| SC3 wiring | `sc3:creditor:mainnet`, `sc3:creditor:testnet`, `sc3:ccc-consumer:mainnet`, `sc3:ccc-consumer:testnet` |

Deployment artifacts: typically **`deployments/mainnet.json`**, **`deployments/testnet.json`** (when used by your pipeline).

---

## 8. Deploy folder (`deploy/`)

- **`deploy/README.md`** — EC2-oriented steps (example `.env` uses testnet IDs; align **`CHAIN_ID`** / **`RPC_URL`** / contract addresses with production).
- **`DOMAIN-SETUP.md`**, **`AWS-DOMAIN-CARDCHAIN.md`**, **`EC2-MYSQL-SETUP.md`** — infrastructure notes.

---

## 9. Related documentation index

| Document | Contents |
|----------|----------|
| [README.md](README.md) | Index of docs in `docs/` |
| [CCC_HUB_SPEC.md](CCC_HUB_SPEC.md) | CCC Hub: swaps, staking, treasuries, no unstake, ops checklist |
| [MAIN_FLOW_AND_CASHFLOW.md](MAIN_FLOW_AND_CASHFLOW.md) | User journey; CLC; USDT splits |
| [SMART_CONTRACTS_AND_PAYMENTS.md](SMART_CONTRACTS_AND_PAYMENTS.md) | Who receives USDT when |
| [CARDS_CASHBACK_SPEC.md](CARDS_CASHBACK_SPEC.md) | Referrer conditions 1–4 |
| [CARDS_POINTS.md](CARDS_POINTS.md) | Points program details |
| [GIFT_CARD_AND_RAFFLE_COUPON.md](GIFT_CARD_AND_RAFFLE_COUPON.md) | Gift card & raffle |
| [FUNCTION_REFERENCE.md](FUNCTION_REFERENCE.md) | Contract & frontend function list |
| [SYSTEM_DIAGRAM.md](SYSTEM_DIAGRAM.md) | Mermaid architecture diagrams |
| [VIDEO_SCRIPT.md](VIDEO_SCRIPT.md) | Narration script |

---

## 10. Change log (documentation)

| Item | Note |
|------|------|
| CCC swap liquidity | USDT from **LoyaltyLevelVault**, not CCCPlatform balance |
| CCC staking | **`unstake` removed** from **CCCPlatform** source; principal not user-withdrawable; **`claimStakeRewards`** only for reward CCC |
| SC3 CLC2 | **$0.50** fixed — not a percentage of queue |
| Points Swap UI | **Admin-wallet** gated **Coming Soon** on **`/ccc-hub`** for non-admins (all networks/builds); see §5.3 |

When smart contracts or env contracts change, update **`contracts.js`** (frontend + backend as applicable), **`deployments/*.json`**, and this file’s **§2** / **§3** if behavior changes.
