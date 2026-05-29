# Replace CCC Platform (correct points → CCC rate)

`CCCPlatform` is now **upgradeable** (transparent proxy). The **proxy address** stays fixed in your app; logic updates use `npm run upgrade:ccc:mainnet`.

If mainnet redeems **100 → 2 CCC**, that deployment used another constant. Fixing it **requires a new CCCPlatform proxy** wired to the same **`CCCToken`**, **`USDT`**, and **`LoyaltyLevelVault`**.

**Nobody can execute mainnet for you here.** Run these on your machine with **vault owner + funded deployer + hub owner** keys.

---

## Migration overview (old hub → new hub)

| What | Old hub | New hub |
|------|---------|---------|
| Stake records (`stakes` mapping) | Stays until imported | Receives copies via `importStakesBatch` |
| CCC tokens (principal + reward pool) | Stays on old contract | Move via `rescueErc20` or treasury fund |
| Points redeem / swap / new stakes | Retire after cutover | Active after SC3 rewire |

**You do not lose CCC on-chain.** You must **import stake rows** and **move CCC** to the new hub so users can claim on the new address.

---

## Step-by-step checklist

### 1. Verify the live contract rate

```bash
cd SC
npx hardhat compile
npm run verify:ccc-platform:mainnet
```

### 2. Deploy new upgradeable CCCPlatform (same CCCToken + vault)

```bash
npm run deploy:ccc-platform:mainnet
```

Writes **new** `CCCPlatform` proxy into `deployments/mainnet.json` and sets **`CCCPlatformPrevious`** to the old address.

### 3. CCCToken — tax exempt on new hub

```bash
npm run ccc:tax-exempt:mainnet
```

(auto-run during deploy if deployer owns CCCToken)

### 4. Vault owner — SC3 wiring

```bash
OLD_CCC_PLATFORM=<old_address> npm run sc3:ccc-consumer:mainnet
npm run sc3:ccc-swap-puller:mainnet
```

Or use **TripleC-Frontend** `/admin/ccc-hub-sc3`.

### 5. Fund the new hub

Points pool + buffer for migrated stakes:

```bash
CCC_FUND_CCC_AMOUNT=50000000 npm run fund:ccc-platform:ccc:mainnet
```

### 6. Dry-run stake migration

```bash
DRY_RUN=1 npm run migrate:ccc-stakes:mainnet
```

Review staker count, total principal, pending rewards, and NEW hub CCC balance.

### 7. Execute stake migration (NEW hub `owner()` signer)

```bash
npm run migrate:ccc-stakes:mainnet
```

This calls `openStakeMigration` → `importStakesBatch` (batched) → `closeStakeMigration` on the **new** hub.

### 8. Move CCC from old hub → new hub

**Old hub owner** (Admin UI `/admin/ccc-hub-sc3` → *Rescue from old hub*, or on-chain):

`rescueErc20(CCCToken, newHub, amount)` on **CCCPlatformPrevious**

Move at least: **sum(staked principal) + sum(pending rewards)** from the dry-run.

### 9. Sync app configs

```bash
node scripts/update-configs-from-deployment.js mainnet
```

Rebuild / redeploy frontend + backend.

### 10. Cutover

- Users **stake / claim / redeem / swap** on the **new** hub only.
- Old hub is **read-only** (optional rescue only).

---

## After first migration

Future logic changes (tax wallet, burn %, etc.):

```bash
npm run upgrade:ccc:mainnet
# or
npm run upgrade:mainnet:all
```

**No address changes** in `contracts.js` — same proxy.

---

## Related

- **[CCC_HUB_SPEC.md](CCC_HUB_SPEC.md)** — economics and flows
- **`SC/scripts/migrate-ccc-stakes.js`** — stake import script
