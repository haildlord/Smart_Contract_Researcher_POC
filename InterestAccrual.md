# Incorrect Interest Accrual and Overestimated totalUsage in DebtToken::mint

**Author:** hailthelord 

**Prototcol:** RAAC

**Proof Link: ** https://codehawks.cyfrin.io/c/2025-02-raac/s/4502

**Severity:** Medium  

**Category:** InterestAccrual  

**Type:** Incorrect Mathematical Calculation (Ray Math + Debt Accounting)  

**Protocol:** Lords Protocol (Lending / crvUSD DebtToken)  

## Summary

The `mint` function in `DebtToken` incorrectly calculates `totalUsage` by using `rayDiv(index)` instead of `rayMul(index)`. This causes systematic underestimation of protocol debt. Additionally, the `balanceIncrease` logic artificially inflates `totalUsage` by double-applying interest, leading to overestimated debt, distorted utilization ratios, incorrect interest rates, and potential overcharging of borrowers.

## Description

In lending protocols that use ray math (27 decimals), `DebtToken` tracks scaled balances. The `totalUsage` (used for utilization ratio, interest rates, etc.) must correctly reflect the **current value** of debt (principal + accrued interest) by multiplying the scaled supply by the current index.

### Vulnerable Code (Problematic Areas)

```solidity
// Inside mint()
uint256 TotalUsage = super.totalSupply();
TotalUsage = TotalUsage.rayDiv(index);           // ← Wrong: should be rayMul

// Later...
if (_userState[onBehalfOf].index != 0 && _userState[onBehalfOf].index < index) {
    balanceIncrease = scaledBalance.rayMul(index) - scaledBalance.rayMul(_userState[onBehalfOf].index);
}

uint256 amountToMint = amount + balanceIncrease;  // ← Artificially inflates debt
```

```solidity
// totalSupply() override
function totalSupply() public view override(ERC20, IERC20) returns (uint256) {
    uint256 scaledSupply = super.totalSupply();
    return scaledSupply.rayDiv(ILendingPool(_reservePool).getNormalizedDebt()); // ← Wrong
}
```

### Incorrect vs Correct Math (Examples from PoC)

**Scenario 1: Borrowing 100 crvUSD at index = 2e27**

- Wrong (`rayDiv`):
  ```solidity
  super.totalSupply = 100e27 / 2e27 = 50
  totalUsage = 50e27 / 2e27 = 25   // Underestimates debt
  ```

- Correct (`rayMul`):
  ```solidity
  totalUsage = 50e27 * 2e27 / 1e27 = 100   // Accurate
  ```

**Scenario 2: Second borrow of 100 at index = 4e27 (after first borrow at 2e27)**

With buggy `balanceIncrease` logic:
- Total borrowed = 200
- Interest accrued (incorrectly) = 500
- `totalUsage` becomes **700** (heavily inflated)

Without `balanceIncrease` and using correct `rayMul`:
- Total borrowed = 200
- Interest accrued (correct) = 100
- `totalUsage` = **300** (accurate)

## Impact

- **Underestimated totalUsage** → Lower utilization ratio → Artificially low interest rates → Protocol earns less interest than it should.
- **Overestimated totalUsage** (due to `balanceIncrease`) → Inflated debt metrics → Borrowers can be overcharged interest → Distorted `currentUsageRate`, `currentLiquidityRate`, and liquidity index.
- **Protocol Instability** — Core economic invariants (total debt vs actual borrowed amounts) become unreliable.
- Affects downstream calculations: borrowing power, liquidation thresholds, reserve factors, and overall capital efficiency.

## Proof of Concept

### 1. Modified `DebtToken::mint` for Testing (Replace existing mint)

```solidity
function mint(
    address user,
    address onBehalfOf,
    uint256 amount,
    uint256 index
) external override returns (bool, uint256, uint256) {
    scb += amount.rayDiv(index);

    if (user == address(0) || onBehalfOf == address(0)) revert InvalidAddress();

    uint256 TotalUsage = super.totalSupply();
    TotalUsage = TotalUsage.rayDiv(index);

    if (amount == 0) {
        return (false, 0, TotalUsage);
    }

    uint256 amountScaled = amount.rayDiv(index);
    if (amountScaled == 0) revert InvalidAmount();

    uint256 scaledBalance = super.balanceOf(user);
    scaledBalance = scaledBalance.rayMul(index);
    bool isFirstMint = scaledBalance == 0;

    uint256 balanceIncrease = 0;
    // mitigation recommends commenting or removing the concept of balanceIncrease
    if (_userState[onBehalfOf].index != 0 && _userState[onBehalfOf].index < index) {
        balanceIncrease = scaledBalance.rayMul(index) - scaledBalance.rayMul(_userState[onBehalfOf].index);
    }

    _userState[onBehalfOf].index = index.toUint128();

    uint256 amountToMint = amount + balanceIncrease;

    uint256 scaledAmount = uint256(amountToMint.toUint128()).rayDiv(index);
    super._update(address(0), user, scaledAmount);

    emit Transfer(address(0), onBehalfOf, amountToMint);
    emit Mint(user, onBehalfOf, amountToMint, balanceIncrease, index);
    console.log("scaledDebtBalance :", scb);
    uint256 balanceOf = super.balanceOf(user);
    console.log("super.balanceOf :", balanceOf);
    console.log("balanceOf :", balanceOf.rayMul(index));
    uint256 Total_Usage = super.totalSupply();
    console.log("super.totalSupply :", Total_Usage);
    Total_Usage = TotalUsagee.rayDiv(index); // mitigation involves using rayMul instead
    console.log("TotalUsage :", Total_Usage);
    return (scaledBalance == 0, amountToMint, Total_Usage);
}
```

### 2. Test Case (Add to `LendingPool.test.js`)

```js
describe("Lords Test", function () {
    it("mint rectification", async function () {
        const depositAmount = ethers.parseEther("1000");
        await crvusd.connect(user2).approve(lendingPool.target, depositAmount);
        await lendingPool.connect(user2).deposit(depositAmount);

        let user5;
        [user5] = await ethers.getSigners();

        // Deploy DebtToken contract
        const DebtToken = await ethers.getContractFactory("DebtToken");
        let debtToken = await DebtToken.deploy("DebtToken", "DT", owner.address);

        await debtToken.mint(user5, user5, 100, ethers.parseUnits("2", 27));
        console.log("---------------------------");
        await debtToken.mint(user5, user5, 100, ethers.parseUnits("4", 27));
    });
});
```

Running the test clearly demonstrates the incorrect `totalUsage` values due to `rayDiv` and the inflating effect of `balanceIncrease`.

## Recommendation

### 1. Remove the flawed `balanceIncrease` logic entirely

```diff
-        uint256 balanceIncrease = 0;
-        if (_userState[onBehalfOf].index != 0 && _userState[onBehalfOf].index < index) {
-           balanceIncrease = scaledBalance.rayMul(index) - scaledBalance.rayMul(_userState[onBehalfOf].index);
-        }
...
-       uint256 amountToMint = amount + balanceIncrease;
+       uint256 amountToMint = amount;
```

### 2. Fix `totalSupply()` to use `rayMul` instead of `rayDiv`

```diff
function totalSupply() public view override(ERC20, IERC20) returns (uint256) {
     uint256 scaledSupply = super.totalSupply();
-    return scaledSupply.rayDiv(ILendingPool(_reservePool).getNormalizedDebt());
+    return scaledSupply.rayMul(ILendingPool(_reservePool).getNormalizedDebt());
}
```

### 3. (Optional but recommended) Also fix the `mint` function's internal `TotalUsage` calculation

Change:
```solidity
TotalUsage = TotalUsage.rayDiv(index);
```
to:
```solidity
TotalUsage = TotalUsage.rayMul(index);
```

These changes ensure `totalUsage` accurately reflects the current value of all outstanding debt (principal + accrued interest) without artificial inflation.
