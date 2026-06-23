# CToken（cETH / cERC20）

* <code><font style="color:rgb(36, 36, 36);">CToken.sol</font></code><font style="color:rgb(36, 36, 36);">：抽象层，包含利息计算、总储备、exchangeRate 的通用逻辑。</font>
* <code><font style="color:rgb(36, 36, 36);">CErc20.sol</font></code><font style="color:rgb(36, 36, 36);">（或 CErc20Delegator/Delegate）：封装 ERC20 为 underlying 的实现，处理 </font><code><font style="color:rgb(36, 36, 36);">mint/borrow/repay/redeem</font></code><font style="color:rgb(36, 36, 36);"> 的 ERC20 路径。</font>
* <code><font style="color:rgb(36, 36, 36);">CEther.sol</font></code><font style="color:rgb(36, 36, 36);">：封装 ETH（payable）路径（mint 时直接发 ETH）</font>

# 函数

## `mint()`

```solidity
function mintInternal(uint mintAmount) internal nonReentrant {
        accrueInterest();
        // mintFresh emits the actual Mint event if successful and logs on errors, so we don't need to
        mintFresh(msg.sender, mintAmount);
    }
```

```solidity
 function mintFresh(address minter, uint mintAmount) internal {
        /* Fail if mint not allowed */
        uint allowed = comptroller.mintAllowed(address(this), minter, mintAmount);
        if (allowed != 0) {
            revert MintComptrollerRejection(allowed);
        }

        /* Verify market's block number equals current block number */
        if (accrualBlockNumber != getBlockNumber()) {
            revert MintFreshnessCheck();
        }

        Exp memory exchangeRate = Exp({mantissa: exchangeRateStoredInternal()});

        /////////////////////////
        // EFFECTS & INTERACTIONS
        // (No safe failures beyond this point)

        /*
         *  We call `doTransferIn` for the minter and the mintAmount.
         *  Note: The cToken must handle variations between ERC-20 and ETH underlying.
         *  `doTransferIn` reverts if anything goes wrong, since we can't be sure if
         *  side-effects occurred. The function returns the amount actually transferred,
         *  in case of a fee. On success, the cToken holds an additional `actualMintAmount`
         *  of cash.
         */
        uint actualMintAmount = doTransferIn(minter, mintAmount);

        /*
         * We get the current exchange rate and calculate the number of cTokens to be minted:
         *  mintTokens = actualMintAmount / exchangeRate
         */

        uint mintTokens = div_(actualMintAmount, exchangeRate);

        /*
         * We calculate the new total supply of cTokens and minter token balance, checking for overflow:
         *  totalSupplyNew = totalSupply + mintTokens
         *  accountTokensNew = accountTokens[minter] + mintTokens
         * And write them into storage
         */
        totalSupply = totalSupply + mintTokens;
        accountTokens[minter] = accountTokens[minter] + mintTokens;

        /* We emit a Mint event, and a Transfer event */
        emit Mint(minter, actualMintAmount, mintTokens);
        emit Transfer(address(this), minter, mintTokens);

        /* We call the defense hook */
        // unused function
        // comptroller.mintVerify(address(this), minter, actualMintAmount, mintTokens);
    }
```

* `mint(uint mintAmount)` / `mint()`（CEther 的 payable）：用户存入 underlying，合约 mint 出 cToken。流程：approve（ERC20）→ `mint` → 调用 `mintFresh`（内部）→ 更新 `totalCash` / `totalBorrows` / `accountTokens` → emit。文档有例子。[docs.compound.finance+1](https://docs.compound.finance/v2/ctokens/?utm_source=chatgpt.com)
* `redeem(uint redeemTokens)` / `redeemUnderlying(uint redeemAmount)`：用户赎回 underlying。内部会调用 `redeemFresh`，并检查合约流动性（是否有足够 underlying）。
* `borrow(uint borrowAmount)`：向用户放款（从池子转出 underlying），内部会检查 Comptroller 的 `borrowAllowed`，然后调用 `borrowFresh` 更新 `accountBorrows` 和 `totalBorrows` 并转出资金。
* `repayBorrow(uint repayAmount)` / `repayBorrowBehalf`：偿还借款，更新借款账本。
* `liquidateBorrow(address borrower, uint repayAmount, CTokenInterface cTokenCollateral)`：清算入口。liquidator 将代偿 borrower 的欠款，协议没收 borrower 抵押的 cTokenCollateral（并按折扣 seize）。
* `accrueInterest()`：关键——利息按时间累加。实现细节：按 `borrowIndex` （累积借款利率指标）来更新 `totalBorrows`、`interest` 分配等。理解 `exchangeRateStored()`、`exchangeRateCurrent()`（计算 cToken 与 underlying 的兑换率）对理解收益/份额非常重要。

## 利息计算（核心）

* 利率模型（interest rate model）是可插拔的合约（例如 `JumpRateModel`）：给定 `utilization = totalBorrows / (cash + totalBorrows - reserves)`，返回 `borrowRatePerBlock`；然后 `supplyRate` 基于 borrowRate、reserveFactor 等计算。
* `accrueInterest` 会用 `borrowRatePerBlock`、时间差 blocks 来计算新增利息并更新 `borrowIndex`。这决定了每个借款人的余额如何按指数增长（其实是按 borrowIndex 比例更新）。详看 `CToken` 的 `accrueInterest` 段落与利率模型合约。


> 更新: 2025-11-06 19:06:52  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/eb86nzr6rh0qcl3i>