# CompoundV2

Compound 是一个去中心化的银行系统，它由三个最核心的合约组成：

1. **Comptroller** —— 风控中枢
2. **cToken（cDAI, cETH…）** —— 资产池与利息计算机器
3. **PriceOracle** —— 喂价服务（告诉系统资产的美元价值）

## 角色

### 供应者 Supplier：存钱的，提供流动性的

Supplier 把自己的资产（比如 DAI）存进 cDAI 合约（一个 ERC20 合约）。\
然后：

* 他得到一份 cDAI 代币，代表他在资金池中的“份额”；
* 这份 cDAI 会不断升值（通过 <code>**exchangeRate**</code>\*\* 增大\*\*来反映利息收益）
* 对应源码（在 CErc20.sol）

#### getAccountLiquidityInternal()

```solidity
function getAccountLiquidityInternal(
        address account
    ) internal view returns (Error, uint, uint) {
        return
            getHypotheticalAccountLiquidityInternal(
                account,
                CToken(address(0)),
                0,
                0
            );
    }
```

```solidity
function getHypotheticalAccountLiquidityInternal(
        address account,
        CToken cTokenModify,
        uint redeemTokens,
        uint borrowAmount
    ) internal view returns (Error, uint, uint) {
        AccountLiquidityLocalVars memory vars; // Holds all our calculation results
        uint oErr;

        // For each asset the account is in
        CToken[] memory assets = accountAssets[account];
        for (uint i = 0; i < assets.length; i++) {
            CToken asset = assets[i];

            // Read the balances and exchange rate from the cToken
            (
                oErr,
                vars.cTokenBalance,
                vars.borrowBalance,
                vars.exchangeRateMantissa
            ) = asset.getAccountSnapshot(account);
            if (oErr != 0) {
                // semi-opaque error code, we assume NO_ERROR == 0 is invariant between upgrades
                return (Error.SNAPSHOT_ERROR, 0, 0);
            }
            vars.collateralFactor = Exp({
                mantissa: markets[address(asset)].collateralFactorMantissa
            });
            vars.exchangeRate = Exp({mantissa: vars.exchangeRateMantissa});

            // Get the normalized price of the asset
            vars.oraclePriceMantissa = oracle.getUnderlyingPrice(asset);
            if (vars.oraclePriceMantissa == 0) {
                return (Error.PRICE_ERROR, 0, 0);
            }
            vars.oraclePrice = Exp({mantissa: vars.oraclePriceMantissa});

            // Pre-compute a conversion factor from tokens -> ether (normalized price value)
            vars.tokensToDenom = mul_(
                mul_(vars.collateralFactor, vars.exchangeRate),
                vars.oraclePrice
            );

            // sumCollateral += tokensToDenom * cTokenBalance
            vars.sumCollateral = mul_ScalarTruncateAddUInt(
                vars.tokensToDenom,
                vars.cTokenBalance,
                vars.sumCollateral
            );

            // sumBorrowPlusEffects += oraclePrice * borrowBalance
            vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(
                vars.oraclePrice,
                vars.borrowBalance,
                vars.sumBorrowPlusEffects
            );

            // Calculate effects of interacting with cTokenModify
            if (asset == cTokenModify) {
                // redeem effect
                // sumBorrowPlusEffects += tokensToDenom * redeemTokens
                vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(
                    vars.tokensToDenom,
                    redeemTokens,
                    vars.sumBorrowPlusEffects
                );

                // borrow effect
                // sumBorrowPlusEffects += oraclePrice * borrowAmount
                vars.sumBorrowPlusEffects = mul_ScalarTruncateAddUInt(
                    vars.oraclePrice,
                    borrowAmount,
                    vars.sumBorrowPlusEffects
                );
            }
        }

        // These are safe, as the underflow condition is checked first
        if (vars.sumCollateral > vars.sumBorrowPlusEffects) {
            return (
                Error.NO_ERROR,
                vars.sumCollateral - vars.sumBorrowPlusEffects,
                0
            );
        } else {
            return (
                Error.NO_ERROR,
                0,
                vars.sumBorrowPlusEffects - vars.sumCollateral
            );
        }
    }
```

#### mint()

```solidity
function mint(uint mintAmount) override external returns (uint) {
        mintInternal(mintAmount);
        return NO_ERROR;
    }
```

源码调用链如下（简化）：

```solidity
function mint(uint mintAmount) external returns (uint) {
    accrueInterest(); // 更新利息
    mintFresh(msg.sender, mintAmount);
}
```

然后看内部函数 mintFresh：

```solidity
function mintFresh(address minter, uint mintAmount) internal returns (uint) {
    // Step 1: Comptroller 检查是否允许铸造
    uint allowed = comptroller.mintAllowed(address(this), minter, mintAmount);
    if (allowed != 0) return fail(...);
    // Step 2: 将用户的 underlying（例如 DAI）转入池子
    doTransferIn(minter, mintAmount);
    
    // Step 3: 根据 exchangeRate 计算要铸造多少 cToken
    uint mintTokens = mintAmount / exchangeRate;
    
    // Step 4: 更新状态
    totalSupply += mintTokens;
    accountTokens[minter] += mintTokens;
    
    emit Mint(minter, mintAmount, mintTokens);
```

| 步骤 | 发生的事 | 涉及合约 | 备注 |
| --- | --- | --- | --- |
| 1 | 用户调用 `cDAI.mint()` | CErc20 | |
| 2 | Comptroller 检查权限 (`mintAllowed`<br/>) | Comptroller | 可能被暂停、限制等 |
| 3 | ERC20 代币（如 DAI）转进合约 | CErc20 | |
| 4 | 铸造出 cToken（如 cDAI）发给用户 | CErc20 | 按 `exchangeRate`<br/> 计算 |
| 5 | 用户赚利息（通过 exchangeRate 增加） | CErc20 | 每次 accrueInterest 后 cToken 价值升高 |

Supplier 的收益不是“数量多了”，而是“每个 cToken 更值钱了”\
这由 exchangeRateStored() 的增长体现。

#### `exchangeRate`原理

**<font style="background-color:#FBDFEF;">exchangeRate = (cash + totalBorrows - reserves) / totalSupply</font>**

也就是说：

1. 池子里借款人越多，赚的利息越多；
2. exchangeRate 就涨；
3. Supplier 手里的 cToken 就值更多 underlying。

#### 一些其他函数

##### `enterMarkets(address[] cTokens)`：注册抵押资产市场。

##### `exitMarket(address cToken)`：退出抵押市场。

##### `mintAllowed(...)` / `redeemAllowed(...)` / `borrowAllowed(...)` / `liquidateBorrowAllowed(...)`：在 cToken 调用这些操作前被询问，判断是否允许。

```solidity
function mintAllowed(
        address cToken,
        address minter,
        uint mintAmount
    ) external override returns (uint) {
        // Pausing is a very serious situation - we revert to sound the alarms
        require(!mintGuardianPaused[cToken], "mint is paused");
        //Guardian 模式。
        //在 Compound 升级中，增加了一个**“紧急开关”（guardian），当出现漏洞或风险时，可以暂停某个市场的 mint 行为**

        // Shh - currently unused
        minter;
        mintAmount;

        if (!markets[cToken].isListed) {
            //检查市场是否被列入
            return uint(Error.MARKET_NOT_LISTED);
        }

        // Keep the flywheel moving
        updateCompSupplyIndex(cToken); //更新该市场的 COMP 奖励累计指数；
        distributeSupplierComp(cToken, minter); //给当前存钱的用户 minter 发放他应得的 COMP

        return uint(Error.NO_ERROR);
        //只要没被暂停、市场存在、分发成功，就返回 0（NO_ERROR）。
        //否则，cToken 合约那边会收到错误码并中止执行
    }

```

##### `getAccountLiquidity(address)`：计算可借额度与是否短缺（会影响清算）。来源：Comptroller 文档 & 源码。:contentReference\[oaicite:19]{index=19}

### 借款人 Borrower

Borrower 把一部分资产（例如 ETH）作为抵押品（存进 cETH）。\
然后他去另一个市场（比如 cDAI）借钱。\
借钱之前，Comptroller 会计算他抵押资产的价值与借款上限。

#### 核心函数：borrow()

调用链：

```solidity
function borrow(uint borrowAmount) external returns (uint) {
  accrueInterest();
  borrowFresh(msg.sender, borrowAmount);
}
```

`borrowFresh`：

```solidity
function borrowFresh(address borrower, uint borrowAmount) internal {
  // Step 1: Comptroller 风险检查
  uint allowed = comptroller.borrowAllowed(address(this), borrower, borrowAmount);
  if (allowed != 0) return fail(...);

  // Step 2: 检查合约现金是否足够
  require(getCashPrior() >= borrowAmount, "insufficient cash");

  // Step 3: 更新账本
  totalBorrows += borrowAmount;
  accountBorrows[borrower].principal += borrowAmount;

  // Step 4: 转出资金
  doTransferOut(borrower, borrowAmount);
}
```

#### comptroller的作用

1. 检查所借 cToken 市场是否被列入（markets\[cToken].isListed）
2. 若借款人尚未在该市场“加入”（accountMembership\[borrower]\[cToken] = false），调用 addToMarketInternal 加入
3. 检查该市场借款是否被暂停（borrowGuardianPaused\[cToken]）
4. 调用 getAccountLiquidityInternal(borrower) 计算：
   * 抵押品总价值 = ∑ (用户在各市场的 cTokenBalance × exchangeRate × underlyingPrice × collateralFactor)
   * 借款总价值 = ∑ (用户在各市场的 borrowBalance × underlyingPrice)
   * 如果 shortfall > 0（即借款总价值 > 抵押价值），拒绝借款
5. 检查本市场是否存在借款上限（borrowCaps\[cToken]），若借款金额超出限额，则拒绝
6. 若以上全部通过，返回 NO\_ERROR，允许继续借款

> 注意：虽然 Comptroller 是审核“是否允许”借款，但真正的资金转出、借款余额更新、活期利息计算等是在对应 cToken 合约中执行的。

| 步骤 | 发生的事 | 涉及合约 |
| --- | --- | --- |
| 1 | 用户先存一笔抵押资产（如 cETH.mint） | CEther |
| 2 | 调用另一个市场借款（如 cDAI.borrow） | CErc20 |
| 3 | Comptroller 检查抵押价值 | Comptroller |
| 4 | 通过后，从池子里转出资金 | CErc20 |
| 5 | 账户 `accountBorrows`<br/> 被记录 | CErc20 |

### 清算人 Liquidator

当 Borrower 抵押物贬值，导致账户借款价值 > 可抵押价值时，系统允许别人来“清算”他。\
Liquidator 相当于“替 borrower 还一部分债”，系统让他以**打折价**拿走 borrower 的抵押品。

#### 对应源码

```solidity
function liquidateBorrow(address borrower, uint repayAmount, CToken cTokenCollateral) external returns (uint) {
    accrueInterest(); // 更新本市场
    cTokenCollateral.accrueInterest(); // 更新抵押市场

    uint allowed = comptroller.liquidateBorrowAllowed(
        address(this), address(cTokenCollateral), msg.sender, borrower, repayAmount
    );
    if (allowed != 0) return fail(...);

    uint repayResult = repayBorrowFresh(msg.sender, borrower, repayAmount);

    // 按清算比例 seize borrower 的抵押品
    cTokenCollateral.seize(msg.sender, borrower, seizeTokens);
}

```

#### comptroller的作用

`liquidateBorrowAllowed` 会：

* 判断 borrower 是否已“超线”（借款/抵押比过高）；
* 调用 PriceOracle 获取实时价格；
* 如果确实超线，就允许清算；
* 计算清算奖励（closeFactor + liquidationIncentive）

| 步骤 | 发生的事 | 涉及合约 |
| --- | --- | --- |
| 1 | Liquidator 偿还 borrower 的部分债务 | cDAI |
| 2 | Comptroller 检查 borrower 是否超线 | Comptroller |
| 3 | 系统计算可没收的抵押资产数量 | Comptroller |
| 4 | Liquidator 获得抵押品（打折价） | cETH |

## cToken 关键函数与流程

* `mint(...)` / `mint()`（ETH）：
  1. 用户 approve（ERC20）或发送 ETH；
  2. cToken 调用 Comptroller 的 `mintAllowed`；
  3. `mintFresh` 更新账户余额与总额，按 `exchangeRate` 发布 cToken。
* `redeem(...)` / `redeemUnderlying(...)`：按当前 `exchangeRate` 赎回 underlying（检查合约可用资金）。
* `borrow(...)`：调用 Comptroller `borrowAllowed`，更新 `accountBorrows`，并转出 underlying。
* `repayBorrow(...)`：偿还借款并更新借款账本。
* `liquidateBorrow(...)`：liquidator 偿还欠款 -> seize borrower's collateral cTokens。
* `accrueInterest()`：按 rate model 与 block 数更新 `borrowIndex` / `totalBorrows`。见 `CToken.sol`。:contentReference\[oaicite:20]{index=20}

## 利率模型

## PriceOracle（重要，因为影响清算）

* Comptroller 从 PriceOracle 获取 `getUnderlyingPrice(cToken)`，将不同资产折算为统一计价单位（通常是 USD）。
* 常见实现：Chainlink feeds、UniswapAnchoredView（历史）等 — 注意返回单位与精度。:contentReference\[oaicite:21]{index=21}

## 阅读/实验建议

* 本地部署：先在本地用 Foundry/Hardhat 部署 Comptroller + one CErc20 + PriceOracle（简化版），写 3 个测试：mint -> borrow -> price drop -> liquidate。
* 重点看 `accrueInterest`, `borrowFresh`, `exchangeRateStored`, `getAccountLiquidity` 的实现。
* 画图：用户账户视角的资产负债表（Supply, Borrow, Collateral value, Borrow value）。

##


> 更新: 2025-11-08 14:56:09  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/gty0xg6ey69ehwms>