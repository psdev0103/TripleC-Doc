# CCC Hub — Product & Technical Spec

Companion to [FULL_PROJECT_SPEC.md](FULL_PROJECT_SPEC.md) §3.3–3.5. Describes **CCCToken**, **CCCPlatform**, and how **LoyaltyLevelVault (SC3)** supplies points debits and USDT for swaps.

---

## 1. What this is (non-technical)

- Users **redeem on-chain loyalty + level points** for **CCC** at a fixed rate (only in multiples of **10** points).
- Users can **swap CCC for USDT** at a fixed price, with a **burn fee** on the CCC they send.
- Users can **stake CCC** to earn **daily CCC rewards**. There is **no “unstake”** button in the current design: **staked principal stays in the hub contract** until a future migration or redeploy changes that. Users **claim reward CCC** separately.
- **No AMM liquidity pool.** Two treasuries are funded by the operator:
  - **CCC** sitting on **CCCPlatform** — pays points redemptions and **mining / staking rewards**.
  - **USDT** sitting on **LoyaltyLevelVault (SC3)** — pays CCC→USDT swaps via a restricted **pull** function.

---

## 2. Contracts & addresses

| Contract | Proxy? | Role |
|----------|--------|------|
| **CCCToken** | No | ERC20, **6 decimals**, fixed supply (**100_000_000** CCC minted to deployer at deploy). |
| **CCCPlatform** | No (immutable) | Points→CCC, CCC→USDT, stake, claim rewards — **new hub = redeploy + SC3 rewire.** |
| **LoyaltyLevelVault** | Upgradeable | Points ledger; **`debitTotalPoints`** for redemptions; **`pullUsdtForCccSwap`** for swap payouts. |

Frontend: **`TripleC-Frontend`** — **`/ccc-hub`**, **`CccHub.jsx`**, **`CONTRACT_ADDRESSES`** in `src/config/contracts.js`.

---

## 3. Economics (matches on-chain constants)

| Rule | Detail |
|------|--------|
| Point step | Multiples of **10** points only |
| Points → CCC | **100 points → 30 CCC** (also **10 → 3**, **1,000 → 300**). |
| CCC → USDT | **5%** of CCC in is **burned**; remainder converts at **$0.05 per 1 CCC** (USDT **18 decimals**, CCC **6 decimals** — math is encoded in Solidity). |
| Stake | **5%** of CCC sent is **burned**; **net** amount increases staked principal. |
| Mining / staking APR | **0.8% per calendar day** on **net staked principal** (`STAKE_DAILY_BPS = 80`). |
| Reward claim | **`claimStakeRewards`** pays **accrued reward CCC** from **CCCPlatform** balance (**`reward pool`** revert if empty). |

**Important:** The contract has **no `unstake`**. Principal is **not** returned to users by the platform contract. Operational recovery of stuck principal (if ever desired) would be an **explicit product decision** (e.g. new contract + migration); **`rescueErc20`** (owner-only) can move **any** ERC20 held by the contract, including CCC that backs stakes — treat as **privileged emergency** only.

---

## 4. Flows

### 4.1 Points → CCC (`swapPointsForCcc`)

1. User calls with `totalPointsToBurn` (valid multiple of **10**, ≤ wallet’s loyalty+level on SC3).
2. **CCCPlatform** requires its own **CCC balance ≥ payout** (`"CCC pool"` if not).
3. **SC3** **`debitTotalPoints(user, burn)`** — burns loyalty first, then level (**consumer** allowlist).
4. **CCC** transferred from **CCCPlatform** to user.

### 4.2 CCC → USDT (`swapCccForUsdt`)

1. User **approves** CCC to **CCCPlatform**; transfers **CCC in** (`safeTransferFrom`).
2. **5%** CCC → **`0xdEaD`**; remainder priced for USDT out.
3. **CCCPlatform** calls **SC3** **`pullUsdtForCccSwap(usdt, user, amount)`**. The vault invokes **Master** **`applyCccSwapUsdtAgainstWalletCaps`** so the user’s NFT **wallet payout caps** are reduced (`amountPaidOutToWallet` ↑) **before** USDT is transferred. Only **`cccPlatformSwapPuller`** may call; vault must hold enough **USDT**. **If NFT headroom is less than `amount`, the swap reverts.**

### 4.3 Stake & rewards (`stake`, `claimStakeRewards`, `pendingStakeRewards`)

1. **`stake`** — **5%** burn; **net** adds to **`stakes[user].amount`**; accrual clock updated.
2. Rewards compound in **`rewardsAccrued`** (**0.8%/day** on net stake while `amount > 0`).
3. **`claimStakeRewards`** — transfers **reward** CCC to user; **does not reduce staked principal**.

---

## 5. Operator checklist (deployment & wiring)

See [FULL_PROJECT_SPEC.md](FULL_PROJECT_SPEC.md) §3.5. Summary:

1. **`setLoyaltyPointConsumer(CCCPlatform, true)`** on SC3 — points debits for redemptions.
2. **`setCccPlatformSwapPuller(CCCPlatform)`** — USDT pulls for swaps.
3. Fund **SC3** with enough **USDT** for swap volume.
4. Fund **CCCPlatform** with **CCC** for redemptions and **staking reward payouts**.

Hardhat helpers: **`sc3:ccc-consumer:*`**, **`sc3:ccc-swap-puller:*`**, **`deploy:ccc-platform:*`** in **`SC/package.json`**.

---

## 6. Frontend behavior

| Area | Behavior |
|------|-----------|
| **Points → CCC** | Multiples of 10; uses **`swapPointsForCcc`**. |
| **CCC → USDT** | Approve CCC → **`swapCccForUsdt`**; liquidity hint = **`USDT.balanceOf(LoyaltyLevelVault)`**. Swap also requires sufficient **NFT wallet payout headroom** (same **`amountPaidOutToWallet`** / withdraw limit buckets as incremental card payouts). |
| **Mining** | **Stake** + **Claim rewards** only (no unstake UI for current bytecode). |
| **Gating** | **None.** Full **`/ccc-hub`** UI for all visitors; connect wallet for on-chain data and transactions. Helpers like **`canAccessCccHubNavSync`** in **`adminContractWallets.js`** are **not** used to hide the hub. |

---

## 7. ABI & bytecode drift

**`TripleC-Frontend/src/abis/CCCPlatform.json`** should match **`SC`** Hardhat artifact **`artifacts/contracts/CCCPlatform.sol/CCCPlatform.json`** after `npx hardhat compile`. Older **deployed** platforms may still expose **`unstake`** until redeployed; docs describe **current source** in this repo.

---

## 8. Related docs

| Doc | Contents |
|-----|----------|
| [FULL_PROJECT_SPEC.md](FULL_PROJECT_SPEC.md) | §3 CCC tables, frontend §5, deploy scripts |
| [FUNCTION_REFERENCE.md](FUNCTION_REFERENCE.md) | **`CCCPlatform`** / **`LoyaltyLevelVault`** function list |
| [SMART_CONTRACTS_AND_PAYMENTS.md](SMART_CONTRACTS_AND_PAYMENTS.md) | When CCC / SC3 pay |
| [SYSTEM_DIAGRAM.md](SYSTEM_DIAGRAM.md) | CCC Hub Mermaid |
| [CARDS_POINTS.md](CARDS_POINTS.md) | SC3 credit rules + CCC redemption (**`debitTotalPoints`** burn order) |
