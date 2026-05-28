# Replace CCC Platform (correct points → CCC rate)

`CCCPlatform` is **immutable**: the **`CCC_PER_10_POINTS`** constant is fixed forever at deployment. Current **Solidity source** sets **`CCC_PER_10_POINTS = 3_000_000`** (CCC has **6 decimals**) so **100 points → 30 CCC** and **1,000 → 300 CCC**.

If mainnet redeems **100 → 2 CCC**, that deployment used another constant (typically **`200_000`** per 10‑point math step). Fixing it **requires a new CCCPlatform deployment** wired to the same **`CCCToken`**, **`USDT`**, and **`LoyaltyLevelVault`**. This document is the checklist.

**Nobody can execute mainnet for you here.** Run these on your machine with **vault owner + funded deployer** keys.

---

## 1. Verify the live contract rate

From `SC/`:

```bash
npx hardhat compile
CCC_PLATFORM=0xYourDeployedPlatform npx hardhat run scripts/verify-ccc-platform-rate.js --network bsc
```

Or rely on **`deployments/mainnet.json → CCCPlatform`**:

```bash
npm run verify:ccc-platform:mainnet
```

Confirm **`CCC_PER_10_POINTS`** prints **`3000000`** and **`previewPointsToCcc(100)`** = **30** CCC **after** redeploy.

---

## 2. Deploy a new CCCPlatform (same CCCToken + vault)

Uses **`SC/scripts/deploy-ccc-platform-only.js`**. Writes the **new** address into **`deployments/mainnet.json`** and sets **`CCCPlatformPrevious`**.

Prereqs: Hardhat signer (e.g. **`PRIVATE_KEY`**) with **BNB** for gas. Json must contain **`CCCToken`**, **`PaymentToken`**, **`LoyaltyLevelVault`**, current **`CCCPlatform`**.

```bash
cd SC
npx hardhat compile
npm run deploy:ccc-platform:mainnet
```

---

## 3. Vault owner — SC3 wiring

Use **`TripleC-Frontend` `/admin/ccc-hub-sc3`** connected as **LoyaltyLevelVault `owner()`**, **or**:

```bash
OLD_CCC_PLATFORM=<previous_address> npm run sc3:ccc-consumer:mainnet
npm run sc3:ccc-swap-puller:mainnet
```

---

## 4. Fund the new hub with CCC

Transfers **CCCToken** from the signer to **`CCCPlatform`**.

```bash
CCC_FUND_CCC_AMOUNT=50000000 npm run fund:ccc-platform:ccc:mainnet
```

---

## 5. Sync app configs

```bash
cd SC
node scripts/update-configs-from-deployment.js mainnet
```

Rebuild and redeploy **frontend / backend**.

---

## 6. Old hub migration (operational)

- **Stake / claim:** principal and rewards belong to whichever **CCCPlatform** the user interacted with unless you migrate off-chain/process.
- **`rescueErc20`** on the **old** platform is **owner-only** — use only under governance policy.

See **[CCC_HUB_SPEC.md](CCC_HUB_SPEC.md)**.
