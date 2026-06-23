# Comptroller

# 源码

```solidity
// SPDX-License-Identifier: BSD-3-Clause
pragma solidity ^0.8.10;

import "./CToken.sol";
import "./ErrorReporter.sol";
import "./PriceOracle.sol";
import "./ComptrollerInterface.sol";
import "./ComptrollerStorage.sol";
import "./Unitroller.sol";
import "./Governance/Comp.sol";

/**
 * @title Compound's Comptroller Contract
 * @author Compound
 */
contract Comptroller is ComptrollerV7Storage, ComptrollerInterface, ComptrollerErrorReporter, ExponentialNoError {
    /// @notice Emitted when an admin supports a market
    event MarketListed(CToken cToken);

    /// @notice Emitted when an account enters a market
    event MarketEntered(CToken cToken, address account);

    /// @notice Emitted when an account exits a market
    event MarketExited(CToken cToken, address account);

    /// @notice Emitted when close factor is changed by admin
    event NewCloseFactor(uint256 oldCloseFactorMantissa, uint256 newCloseFactorMantissa);

    /// @notice Emitted when a collateral factor is changed by admin
    event NewCollateralFactor(CToken cToken, uint256 oldCollateralFactorMantissa, uint256 newCollateralFactorMantissa);

    /// @notice Emitted when liquidation incentive is changed by admin
    event NewLiquidationIncentive(uint256 oldLiquidationIncentiveMantissa, uint256 newLiquidationIncentiveMantissa);

    /// @notice Emitted when price oracle is changed
    event NewPriceOracle(PriceOracle oldPriceOracle, PriceOracle newPriceOracle);

    /// @notice Emitted when pause guardian is changed
    event NewPauseGuardian(address oldPauseGuardian, address newPauseGuardian);

    /// @notice Emitted when an action is paused globally
    event ActionPaused(string action, bool pauseState);

    /// @notice Emitted when an action is paused on a market
    event ActionPaused(CToken cToken, string action, bool pauseState);

    /// @notice Emitted when a new borrow-side COMP speed is calculated for a market
    event CompBorrowSpeedUpdated(CToken indexed cToken, uint256 newSpeed);

    /// @notice Emitted when a new supply-side COMP speed is calculated for a market
    event CompSupplySpeedUpdated(CToken indexed cToken, uint256 newSpeed);

    /// @notice Emitted when a new COMP speed is set for a contributor
    event ContributorCompSpeedUpdated(address indexed contributor, uint256 newSpeed);

    /// @notice Emitted when COMP is distributed to a supplier
    event DistributedSupplierComp(
        CToken indexed cToken, address indexed supplier, uint256 compDelta, uint256 compSupplyIndex
    );

    /// @notice Emitted when COMP is distributed to a borrower
    event DistributedBorrowerComp(
        CToken indexed cToken, address indexed borrower, uint256 compDelta, uint256 compBorrowIndex
    );

    /// @notice Emitted when borrow cap for a cToken is changed
    event NewBorrowCap(CToken indexed cToken, uint256 newBorrowCap);

    /// @notice Emitted when borrow cap guardian is changed
    event NewBorrowCapGuardian(address oldBorrowCapGuardian, address newBorrowCapGuardian);

    /// @notice Emitted when COMP is granted by admin
    event CompGranted(address recipient, uint256 amount);

    /// @notice Emitted when COMP accrued for a user has been manually adjusted.
    event CompAccruedAdjusted(address indexed user, uint256 oldCompAccrued, uint256 newCompAccrued);

    /// @notice Emitted when COMP receivable for a user has been updated.
    event CompReceivableUpdated(address indexed user, uint256 oldCompReceivable, uint256 newCompReceivable);

    /// @notice The initial COMP index for a market
    uint224 public constant compInitialIndex = 1e36;

    // closeFactorMantissa must be strictly greater than this value
    uint256 internal constant closeFactorMinMantissa = 0.05e18; // 0.05

    // closeFactorMantissa must not exceed this value
    uint256 internal constant closeFactorMaxMantissa = 0.9e18; // 0.9

    // No collateralFactorMantissa may exceed this value
    uint256 internal constant collateralFactorMaxMantissa = 0.9e18; // 0.9

    constructor() {
        admin = msg.sender;
    }

    /**
     * Assets You Are In **
     */

    /**
     * @not？ice Returns the assets an account has entered
     * @param account The address of the account to pull assets for
     * @return A dynamic list with the assets the account has entered
     */
    /**
     * @notice 返回帐户输入的资产
     *
     * @param帐户 提取资产的帐户地址
     *
     * @return 包含帐户已输入的资产的动态列表
     */
    function getAssetsIn(address account) external view returns (CToken[] memory) {
        CToken[] memory assetsIn = accountAssets[account];

        return assetsIn;
    }

    /**
     * @notice Returns whether the given account is entered in the given asset
     * @param account The address of the account to check
     * @param cToken The cToken to check
     * @return True if the account is in the asset, otherwise false.
     */
    //     @notice 返回指定账户是否已经进入指定资产市场。
    // @param account 要检查的账户地址。
    // @param cToken 要检查的 cToken（资产市场）。
    // @return 如果该账户已进入该资产市场，则返回 true，否则返回 false。
    function checkMembership(address account, CToken cToken) external view returns (bool) {
        return markets[address(cToken)].accountMembership[account];
    }

    /**
     * @notice Add assets to be included in account liquidity calculation
     * @param cTokens The list of addresses of the cToken markets to be enabled
     * @return Success indicator for whether each corresponding market was entered
     */
    /**
     * @notice 添加资产以纳入账户流动性计算
     *
     * @param cTokens 要启用的cToken市场地址列表
     *
     * @Return 是否进入每个相应市场的成功指标
     */
    function enterMarkets(address[] memory cTokens) public override returns (uint256[] memory) {
        uint256 len = cTokens.length;

        uint256[] memory results = new uint256[](len);
        for (uint256 i = 0; i < len; i++) {
            CToken cToken = CToken(cTokens[i]);

            results[i] = uint256(addToMarketInternal(cToken, msg.sender));
        }

        return results;
    }

    /**
     * @notice Add the market to the borrower's "assets in" for liquidity calculations
     * @param cToken The market to enter
     * @param borrower The address of the account to modify
     * @return Success indicator for whether the market was entered
     */
    /**
     * @notice 将市场添加到借款人的“资产”中，以进行流动性计算
     *
     * @param cToken 进入的市场
     *
     * @param借款人 要修改的帐户地址
     *
     * @return 是否进入市场的成功指标
     */
    function addToMarketInternal(CToken cToken, address borrower) internal returns (Error) {
        Market storage marketToJoin = markets[address(cToken)];

        if (!marketToJoin.isListed) {
            // market is not listed, cannot join
            return Error.MARKET_NOT_LISTED;
        }

        if (marketToJoin.accountMembership[borrower] == true) {
            // already joined
            return Error.NO_ERROR;
        }

        // survived the gauntlet, add to list
        // NOTE: we store these somewhat redundantly as a significant optimization
        //  this avoids having to iterate through the list for the most common use cases
        //  that is, only when we need to perform liquidity checks
        //  and not whenever we want to check if an account is in a particular market
        //在挑战中幸存下来，添加到列表中

        // 注意：我们或多或少地存储这些作为显著的优化

        //这避免了在最常见的用例中迭代列表

        // 也就是说，只有当我们需要进行流动性检查时

        //不是每当我们想检查帐户是否在特定市场时
        marketToJoin.accountMembership[borrower] = true;
        accountAssets[borrower].push(cToken);

        emit MarketEntered(cToken, borrower);

        return Error.NO_ERROR;
    }

    /**
     * @notice Removes asset from sender's account liquidity calculation
     * @dev Sender must not have an outstanding borrow balance in the asset,
     *  or be providing necessary collateral for an outstanding borrow.
     * @param cTokenAddress The address of the asset to be removed
     * @return Whether or not the account successfully exited the market
     */
    /**
     * @notice从发件人的账户流动性计算中删除资产
     *
     * @dev 发件人资产中不得有未付的借款余额，
     *
     * 或为未偿借款提供必要的抵押品。
     *
     * @param cTokenAddress 要删除的资产的地址
     *
     * @return 该帐户是否成功退出市场
     */
    function exitMarket(address cTokenAddress) external override returns (uint256) {
        CToken cToken = CToken(cTokenAddress);
        /* Get sender tokensHeld and amountOwed underlying from the cToken */
        (uint256 oErr, uint256 tokensHeld, uint256 amountOwed,) = cToken.getAccountSnapshot(msg.sender);
        require(oErr == 0, "exitMarket: getAccountSnapshot failed"); // semi-opaque error code

        /* Fail if the sender has a borrow balance */
        if (amountOwed != 0) {
            return fail(Error.NONZERO_BORROW_BALANCE, FailureInfo.EXIT_MARKET_BALANCE_OWED);
        }

        /* Fail if the sender is not permitted to redeem all of their tokens */
        uint256 allowed = redeemAllowedInternal(cTokenAddress, msg.sender, tokensHeld);
        if (allowed != 0) {
            return failOpaque(Error.REJECTION, FailureInfo.EXIT_MARKET_REJECTION, allowed);
        }

        Market storage marketToExit = markets[address(cToken)];

        /* Return true if the sender is not already ‘in’ the market */
        if (!marketToExit.accountMembership[msg.sender]) {
            return uint256(Error.NO_ERROR);
        }

        /* Set cToken account membership to false */
        delete marketToExit.accountMembership[msg.sender];

        /* Delete cToken from the account’s list of assets */
        // load into memory for faster iteration
        CToken[] memory userAssetList = accountAssets[msg.sender];
        uint256 len = userAssetList.length;
        uint256 assetIndex = len;
        for (uint256 i = 0; i < len; i++) {
            if (userAssetList[i] == cToken) {
                assetIndex = i;
                break;
            }
        }

        // We *must* have found the asset in the list or our redundant data structure is broken
        assert(assetIndex < len);

        // copy last item in list to location of item to be removed, reduce length by 1
        CToken[] storage storedList = accountAssets[msg.sender];
        storedList[assetIndex] = storedList[storedList.length - 1];
        storedList.pop();

        emit MarketExited(cToken, msg.sender);

        return uint256(Error.NO_ERROR);
    }

    /**
     * Policy Hooks **
     */

    /**
     * @notice Checks if the account should be allowed to mint tokens in the given market
     * @param cToken The market to verify the mint against
     * @param minter The account which would get the minted tokens
     * @param mintAmount The amount of underlying being supplied to the market in exchange for tokens
     * @return 0 if the mint is allowed, otherwise a semi-opaque error code (See ErrorReporter.sol)
     */
    /**
     * @notice 检查是否应允许该帐户在给定市场中铸造代币
     *
     * @param cToken 验证铸币厂的市场
     *
     * @param minter 获得铸造代币的帐户
     *
     * @param mintAmount 向市场供应的标的金额以换取代币
     *
     * 如果允许mint，则@return 0，否则半不透明错误代码（见ErrorReporter.sol）
     */
    function mintAllowed(address cToken, address minter, uint256 mintAmount) external override returns (uint256) {
        // Pausing is a very serious situation - we revert to sound the alarms
        require(!mintGuardianPaused[cToken], "mint is paused");

        // Shh - currently unused
        minter;
        mintAmount;

        if (!markets[cToken].isListed) {
            return uint256(Error.MARKET_NOT_LISTED);
        }

        // Keep the flywheel moving
        updateCompSupplyIndex(cToken);
        distributeSupplierComp(cToken, minter);

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Validates mint and reverts on rejection. May emit logs.
     * @param cToken Asset being minted
     * @param minter The address minting the tokens
     * @param actualMintAmount The amount of the underlying asset being minted
     * @param mintTokens The number of tokens being minted
     */
    /**
     * @notice验证铸币厂，并在拒绝时恢复。可能会发出日志。
     *
     * @param cToken 资产正在铸造中
     *
     * @param minter 铸造代币的地址
     *
     * @param actualMintAmount 铸造的标的资产金额
     *
     * @param mintTokens 铸造的代币数量
     */
    function mintVerify(address cToken, address minter, uint256 actualMintAmount, uint256 mintTokens)
        external
        override
    {
        // Shh - currently unused
        cToken;
        minter;
        actualMintAmount;
        mintTokens;

        // Shh - we don't ever want this hook to be marked pure
        if (false) {
            maxAssets = maxAssets;
        }
    }

    /**
     * @notice Checks if the account should be allowed to redeem tokens in the given market
     * @param cToken The market to verify the redeem against
     * @param redeemer The account which would redeem the tokens
     * @param redeemTokens The number of cTokens to exchange for the underlying asset in the market
     * @return 0 if the redeem is allowed, otherwise a semi-opaque error code (See ErrorReporter.sol)
     */
    /**
     * @notice 检查该帐户是否应该被允许在给定市场中兑换代币
     *
     * @param cToken 验证兑换的市场
     *
     * @param兑换者 兑换代币的帐户
     *
     * @param redeemTokens 兑换市场上标的资产的cTokens数量
     *
     * 如果允许兑换，则为@return 0，否则为半不透明的错误代码（请参阅ErrorReporter.sol）
     */
    function redeemAllowed(address cToken, address redeemer, uint256 redeemTokens)
        external
        override
        returns (uint256)
    {
        uint256 allowed = redeemAllowedInternal(cToken, redeemer, redeemTokens);
        if (allowed != uint256(Error.NO_ERROR)) {
            return allowed;
        }

        // Keep the flywheel moving
        updateCompSupplyIndex(cToken);
        distributeSupplierComp(cToken, redeemer);

        return uint256(Error.NO_ERROR);
    }

    function redeemAllowedInternal(address cToken, address redeemer, uint256 redeemTokens)
        internal
        view
        returns (uint256)
    {
        if (!markets[cToken].isListed) {
            return uint256(Error.MARKET_NOT_LISTED);
        }

        /* If the redeemer is not 'in' the market, then we can bypass the liquidity check */
        if (!markets[cToken].accountMembership[redeemer]) {
            return uint256(Error.NO_ERROR);
        }

        /* Otherwise, perform a hypothetical liquidity check to guard against shortfall */
        (Error err,, uint256 shortfall) =
            getHypotheticalAccountLiquidityInternal(redeemer, CToken(cToken), redeemTokens, 0);
        if (err != Error.NO_ERROR) {
            return uint256(err);
        }
        if (shortfall > 0) {
            return uint256(Error.INSUFFICIENT_LIQUIDITY);
        }

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Validates redeem and reverts on rejection. May emit logs.
     * @param cToken Asset being redeemed
     * @param redeemer The address redeeming the tokens
     * @param redeemAmount The amount of the underlying asset being redeemed
     * @param redeemTokens The number of tokens being redeemed
     */
    /**
     * @notice 验证兑换，并在被拒绝时恢复。可能会发出日志。
     *
     * @param cToken 资产正在兑换中
     *
     * @param兑换者 兑换代币的地址
     *
     * @param redeemAmount 被赎回的标的资产金额
     *
     * @param redeemTokens 兑换的代币数量
     */
    function redeemVerify(address cToken, address redeemer, uint256 redeemAmount, uint256 redeemTokens)
        external
        override
    {
        // Shh - currently unused
        cToken;
        redeemer;

        // Require tokens is zero or amount is also zero
        if (redeemTokens == 0 && redeemAmount > 0) {
            revert("redeemTokens zero");
        }
    }

    /**
     * @notice Checks if the account should be allowed to borrow the underlying asset of the given market
     * @param cToken The market to verify the borrow against
     * @param borrower The account which would borrow the asset
     * @param borrowAmount The amount of underlying the account would borrow
     * @return 0 if the borrow is allowed, otherwise a semi-opaque error code (See ErrorReporter.sol)
     */
    /**
     * @notice 检查是否应该允许该账户借入给定市场的标的资产
     *
     * @param cToken 验证借款的市场
     *
     * @param借款人 借入资产的账户
     *
     * @param borrowAmount 账户将借入的底金金额
     *
     * 如果允许借用，则@return 0，否则为半不透明的错误代码（请参阅ErrorReporter.sol）
     */
    function borrowAllowed(address cToken, address borrower, uint256 borrowAmount)
        external
        override
        returns (uint256)
    {
        // Pausing is a very serious situation - we revert to sound the alarms
        require(!borrowGuardianPaused[cToken], "borrow is paused");

        if (!markets[cToken].isListed) {
            return uint256(Error.MARKET_NOT_LISTED);
        }

        if (!markets[cToken].accountMembership[borrower]) {
            // only cTokens may call borrowAllowed if borrower not in market
            require(msg.sender == cToken, "sender must be cToken");

            // attempt to add borrower to the market
            Error err = addToMarketInternal(CToken(msg.sender), borrower);
            if (err != Error.NO_ERROR) {
                return uint256(err);
            }

            // it should be impossible to break the important invariant
            assert(markets[cToken].accountMembership[borrower]);
        }

        if (oracle.getUnderlyingPrice(CToken(cToken)) == 0) {
            return uint256(Error.PRICE_ERROR);
        }

        uint256 borrowCap = borrowCaps[cToken];
        // Borrow cap of 0 corresponds to unlimited borrowing
        if (borrowCap != 0) {
            uint256 totalBorrows = CToken(cToken).totalBorrows();
            uint256 nextTotalBorrows = add_(totalBorrows, borrowAmount);
            require(nextTotalBorrows < borrowCap, "market borrow cap reached");
        }

        (Error err,, uint256 shortfall) =
            getHypotheticalAccountLiquidityInternal(borrower, CToken(cToken), 0, borrowAmount);
        if (err != Error.NO_ERROR) {
            return uint256(err);
        }
        if (shortfall > 0) {
            return uint256(Error.INSUFFICIENT_LIQUIDITY);
        }

        // Keep the flywheel moving
        Exp memory borrowIndex = Exp({mantissa: CToken(cToken).borrowIndex()});
        updateCompBorrowIndex(cToken, borrowIndex);
        distributeBorrowerComp(cToken, borrower, borrowIndex);

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Validates borrow and reverts on rejection. May emit logs.
     * @param cToken Asset whose underlying is being borrowed
     * @param borrower The address borrowing the underlying
     * @param borrowAmount The amount of the underlying asset requested to borrow
     */
    /**
     * @notice验证借用并在拒绝时恢复。可能会发出日志。
     *
     * @param cToken 资产，其基础是借入的
     *
     * @param借款人 借用基础的地址
     *
     * @param borrowAmount 要求借款的标的资产金额
     */
    function borrowVerify(address cToken, address borrower, uint256 borrowAmount) external override {
        // Shh - currently unused
        cToken;
        borrower;
        borrowAmount;

        // Shh - we don't ever want this hook to be marked pure
        if (false) {
            maxAssets = maxAssets;
        }
    }

    /**
     * @notice Checks if the account should be allowed to repay a borrow in the given market
     * @param cToken The market to verify the repay against
     * @param payer The account which would repay the asset
     * @param borrower The account which would borrowed the asset
     * @param repayAmount The amount of the underlying asset the account would repay
     * @return 0 if the repay is allowed, otherwise a semi-opaque error code (See ErrorReporter.sol)
     */
    /**
     * @notice 检查是否应允许该帐户在给定市场中偿还借款
     *
     * @param cToken 验证还款的市场
     *
     * @param payer 偿还资产的账户
     *
     * @param借款人 借入资产的账户
     *
     * @param repayAmount 账户将偿还的标的资产金额
     *
     * 如果允许还款，则@return 0，否则半不透明错误代码（见ErrorReporter.sol）
     */
    function repayBorrowAllowed(address cToken, address payer, address borrower, uint256 repayAmount)
        external
        override
        returns (uint256)
    {
        // Shh - currently unused
        payer;
        borrower;
        repayAmount;

        if (!markets[cToken].isListed) {
            return uint256(Error.MARKET_NOT_LISTED);
        }

        // Keep the flywheel moving
        Exp memory borrowIndex = Exp({mantissa: CToken(cToken).borrowIndex()});
        updateCompBorrowIndex(cToken, borrowIndex);
        distributeBorrowerComp(cToken, borrower, borrowIndex);

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Validates repayBorrow and reverts on rejection. May emit logs.
     * @param cToken Asset being repaid
     * @param payer The address repaying the borrow
     * @param borrower The address of the borrower
     * @param actualRepayAmount The amount of underlying being repaid
     */
    /**
     * @notice验证repayBorrow，并在拒绝时恢复。可能会发出日志。
     *
     * @param cToken 资产正在偿还
     *
     * @param 付款人 偿还借款的地址
     *
     * @param 借款人 借款人的地址
     *
     * @param actualRepayAmount 被偿还的标的金额
     */
    function repayBorrowVerify(
        address cToken,
        address payer,
        address borrower,
        uint256 actualRepayAmount,
        uint256 borrowerIndex
    ) external override {
        // Shh - currently unused
        cToken;
        payer;
        borrower;
        actualRepayAmount;
        borrowerIndex;

        // Shh - we don't ever want this hook to be marked pure
        if (false) {
            maxAssets = maxAssets;
        }
    }

    /**
     * @notice Checks if the liquidation should be allowed to occur
     * @param cTokenBorrowed Asset which was borrowed by the borrower
     * @param cTokenCollateral Asset which was used as collateral and will be seized
     * @param liquidator The address repaying the borrow and seizing the collateral
     * @param borrower The address of the borrower
     * @param repayAmount The amount of underlying being repaid
     */
    /**
     * @notice 检查是否应该允许清算发生
     *
     * @param cToken借款人借入的资产
     *
     * @param cTokenCollateral资产被用作抵押品，将被扣押
     *
     * @param liquidator 偿还借款和扣押抵押品的地址
     *
     * @param 借款人 借款人的地址
     *
     * @param repayAmount 偿还的标的金额
     */
    function liquidateBorrowAllowed(
        address cTokenBorrowed,
        address cTokenCollateral,
        address liquidator,
        address borrower,
        uint256 repayAmount
    ) external override returns (uint256) {
        // Shh - currently unused
        liquidator;

        if (!markets[cTokenBorrowed].isListed || !markets[cTokenCollateral].isListed) {
            return uint256(Error.MARKET_NOT_LISTED);
        }

        uint256 borrowBalance = CToken(cTokenBorrowed).borrowBalanceStored(borrower);

        /* allow accounts to be liquidated if the market is deprecated */
        if (isDeprecated(CToken(cTokenBorrowed))) {
            require(borrowBalance >= repayAmount, "Can not repay more than the total borrow");
        } else {
            /* The borrower must have shortfall in order to be liquidatable */
            (Error err,, uint256 shortfall) = getAccountLiquidityInternal(borrower);
            if (err != Error.NO_ERROR) {
                return uint256(err);
            }

            if (shortfall == 0) {
                return uint256(Error.INSUFFICIENT_SHORTFALL);
            }

            /* The liquidator may not repay more than what is allowed by the closeFactor */
            uint256 maxClose = mul_ScalarTruncate(Exp({mantissa: closeFactorMantissa}), borrowBalance);
            if (repayAmount > maxClose) {
                return uint256(Error.TOO_MUCH_REPAY);
            }
        }
        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Validates liquidateBorrow and reverts on rejection. May emit logs.
     * @param cTokenBorrowed Asset which was borrowed by the borrower
     * @param cTokenCollateral Asset which was used as collateral and will be seized
     * @param liquidator The address repaying the borrow and seizing the collateral
     * @param borrower The address of the borrower
     * @param actualRepayAmount The amount of underlying being repaid
     */
    /**
     * @notice 验证 liquidateBorrow，并在拒绝时恢复。可能会发出日志。
     *
     * @param cToken借款人借入的资产
     *
     * @param cTokenCollateral资产被用作抵押品，将被扣押
     *
     * @param liquidator 偿还借款和扣押抵押品的地址
     *
     * @param 借款人 借款人的地址
     *
     * @param actualRepayAmount 被偿还的标的金额
     */
    function liquidateBorrowVerify(
        address cTokenBorrowed,
        address cTokenCollateral,
        address liquidator,
        address borrower,
        uint256 actualRepayAmount,
        uint256 seizeTokens
    ) external override {
        // Shh - currently unused
        cTokenBorrowed;
        cTokenCollateral;
        liquidator;
        borrower;
        actualRepayAmount;
        seizeTokens;

        // Shh - we don't ever want this hook to be marked pure
        if (false) {
            maxAssets = maxAssets;
        }
    }

    /**
     * @notice Checks if the seizing of assets should be allowed to occur
     * @param cTokenCollateral Asset which was used as collateral and will be seized
     * @param cTokenBorrowed Asset which was borrowed by the borrower
     * @param liquidator The address repaying the borrow and seizing the collateral
     * @param borrower The address of the borrower
     * @param seizeTokens The number of collateral tokens to seize
     */
    /**
     * @notice 检查是否应该允许资产被扣押
     *
     * @param cTokenCollateral资产被用作抵押品，将被扣押
     *
     * @param cToken借款人借入的资产
     *
     * @param liquidator 偿还借款和扣押抵押品的地址
     *
     * @param 借款人 借款人的地址
     *
     * @param seizeTokens 要扣押的抵押代币数量
     */
    function seizeAllowed(
        address cTokenCollateral,
        address cTokenBorrowed,
        address liquidator,
        address borrower,
        uint256 seizeTokens
    ) external override returns (uint256) {
        // Pausing is a very serious situation - we revert to sound the alarms
        require(!seizeGuardianPaused, "seize is paused");

        // Shh - currently unused
        seizeTokens;

        if (!markets[cTokenCollateral].isListed || !markets[cTokenBorrowed].isListed) {
            return uint256(Error.MARKET_NOT_LISTED);
        }

        if (CToken(cTokenCollateral).comptroller() != CToken(cTokenBorrowed).comptroller()) {
            return uint256(Error.COMPTROLLER_MISMATCH);
        }

        // Keep the flywheel moving
        updateCompSupplyIndex(cTokenCollateral);
        distributeSupplierComp(cTokenCollateral, borrower);
        distributeSupplierComp(cTokenCollateral, liquidator);

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Validates seize and reverts on rejection. May emit logs.
     * @param cTokenCollateral Asset which was used as collateral and will be seized
     * @param cTokenBorrowed Asset which was borrowed by the borrower
     * @param liquidator The address repaying the borrow and seizing the collateral
     * @param borrower The address of the borrower
     * @param seizeTokens The number of collateral tokens to seize
     */
    /**
     * @notice 验证扣押并恢复拒绝。可能会发出日志。
     *
     * @param cTokenCollateral资产被用作抵押品，将被扣押
     *
     * @param cToken借款人借入的资产
     *
     * @param liquidator 偿还借款和扣押抵押品的地址
     *
     * @param 借款人 借款人的地址
     *
     * @param seizeTokens 要扣押的抵押代币数量
     */
    function seizeVerify(
        address cTokenCollateral,
        address cTokenBorrowed,
        address liquidator,
        address borrower,
        uint256 seizeTokens
    ) external override {
        // Shh - currently unused
        cTokenCollateral;
        cTokenBorrowed;
        liquidator;
        borrower;
        seizeTokens;

        // Shh - we don't ever want this hook to be marked pure
        if (false) {
            maxAssets = maxAssets;
        }
    }

    /**
     * @notice Checks if the account should be allowed to transfer tokens in the given market
     * @param cToken The market to verify the transfer against
     * @param src The account which sources the tokens
     * @param dst The account which receives the tokens
     * @param transferTokens The number of cTokens to transfer
     * @return 0 if the transfer is allowed, otherwise a semi-opaque error code (See ErrorReporter.sol)
     */
    /**
     * @notice 检查该帐户是否应该被允许在给定市场中转移代币
     *
     * @param cToken 验证转让的市场
     *
     * @param src 获取代币的帐户
     *
     * @param dst 接收代币的帐户
     *
     * @param transferTokens 要传输的cTokens数量
     *
     * 如果允许传输，则@return 0，否则为半不透明错误代码（见ErrorReporter.sol）
     */
    function transferAllowed(address cToken, address src, address dst, uint256 transferTokens)
        external
        override
        returns (uint256)
    {
        // Pausing is a very serious situation - we revert to sound the alarms
        require(!transferGuardianPaused, "transfer is paused");

        // Currently the only consideration is whether or not
        //  the src is allowed to redeem this many tokens
        uint256 allowed = redeemAllowedInternal(cToken, src, transferTokens);
        if (allowed != uint256(Error.NO_ERROR)) {
            return allowed;
        }

        // Keep the flywheel moving
        updateCompSupplyIndex(cToken);
        distributeSupplierComp(cToken, src);
        distributeSupplierComp(cToken, dst);

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Validates transfer and reverts on rejection. May emit logs.
     * @param cToken Asset being transferred
     * @param src The account which sources the tokens
     * @param dst The account which receives the tokens
     * @param transferTokens The number of cTokens to transfer
     */
    /**
     * @notice验证转让，并在拒绝时恢复。可能会发出日志。
     *
     * @param cToken 资产正在转让
     *
     * @param src 获取代币的帐户
     *
     * @param dst 接收代币的帐户
     *
     * @param transferTokens 要传输的cTokens数量
     */
    function transferVerify(address cToken, address src, address dst, uint256 transferTokens) external override {
        // Shh - currently unused
        cToken;
        src;
        dst;
        transferTokens;

        // Shh - we don't ever want this hook to be marked pure
        if (false) {
            maxAssets = maxAssets;
        }
    }

    /**
     * Liquidity/Liquidation Calculations **
     */

    /**
     * @dev Local vars for avoiding stack-depth limits in calculating account liquidity.
     *  Note that `cTokenBalance` is the number of cTokens the account owns in the market,
     *  whereas `borrowBalance` is the amount of underlying that the account has borrowed.
     */
    /**
     * @dev 本地变量，用于在计算账户流动性时避免堆叠深度限制。
     *
     * 请注意，“cTokenBalance”是该帐户在市场上拥有的cTokens数量，
     *
     * 而“借款余额”是该账户借款的标的金额。
     */
    struct AccountLiquidityLocalVars {
        uint256 sumCollateral;
        uint256 sumBorrowPlusEffects;
        uint256 cTokenBalance;
        uint256 borrowBalance;
        uint256 exchangeRateMantissa;
        uint256 oraclePriceMantissa;
        Exp collateralFactor;
        Exp exchangeRate;
        Exp oraclePrice;
        Exp tokensToDenom;
    }

    /**
     * @notice Determine the current account liquidity wrt collateral requirements
     * @return (possible error code (semi-opaque),
     *             account liquidity in excess of collateral requirements,
     *          account shortfall below collateral requirements)
     */
    /**
     * @notice 确定活期账户流动性wrt抵押品要求
     *
     * @return（可能的错误代码（半不透明），
     *
     * 账户流动性超过抵押品要求，
     *
     * 账户短缺低于抵押品要求）
     */
    function getAccountLiquidity(address account) public view returns (uint256, uint256, uint256) {
        (Error err, uint256 liquidity, uint256 shortfall) =
            getHypotheticalAccountLiquidityInternal(account, CToken(address(0)), 0, 0);

        return (uint256(err), liquidity, shortfall);
    }

    /**
     * @notice Determine the current account liquidity wrt collateral requirements
     * @return (possible error code,
     *             account liquidity in excess of collateral requirements,
     *          account shortfall below collateral requirements)
     */
    /**
     * @notice 确定活期账户流动性wrt抵押品要求
     *
     * @return（可能的错误代码，
     *
     * 账户流动性超过抵押品要求，
     *
     * 账户短缺低于抵押品要求）
     */
    function getAccountLiquidityInternal(address account) internal view returns (Error, uint256, uint256) {
        return getHypotheticalAccountLiquidityInternal(account, CToken(address(0)), 0, 0);
    }

    /**
     * @notice Determine what the account liquidity would be if the given amounts were redeemed/borrowed
     * @param cTokenModify The market to hypothetically redeem/borrow in
     * @param account The account to determine liquidity for
     * @param redeemTokens The number of tokens to hypothetically redeem
     * @param borrowAmount The amount of underlying to hypothetically borrow
     * @return (possible error code (semi-opaque),
     *             hypothetical account liquidity in excess of collateral requirements,
     *          hypothetical account shortfall below collateral requirements)
     */
    /**
     * @notice 确定如果给定金额被赎回/借款，账户流动性会是多少
     *
     * @param cTokenModify 假设赎回/借款的市场
     *
     * @param帐户 确定流动性的帐户
     *
     * @param redeemTokens 假设赎回的代币数量
     *
     * @param borrowAmount 假设借入的标的金额
     *
     * @return（可能的错误代码（半不透明），
     *
     * 假设账户流动性超过抵押品要求，
     *
     * 假设账户短缺低于抵押品要求）
     */
    function getHypotheticalAccountLiquidity(
        address account,
        address cTokenModify,
        uint256 redeemTokens,
        uint256 borrowAmount
    ) public view returns (uint256, uint256, uint256) {
        (Error err, uint256 liquidity, uint256 shortfall) =
            getHypotheticalAccountLiquidityInternal(account, CToken(cTokenModify), redeemTokens, borrowAmount);
        return (uint256(err), liquidity, shortfall);
    }

    /**
     * @notice Determine what the account liquidity would be if the given amounts were redeemed/borrowed
     * @param cTokenModify The market to hypothetically redeem/borrow in
     * @param account The account to determine liquidity for
     * @param redeemTokens The number of tokens to hypothetically redeem
     * @param borrowAmount The amount of underlying to hypothetically borrow
     * @dev Note that we calculate the exchangeRateStored for each collateral cToken using stored data,
     *  without calculating accumulated interest.
     * @return (possible error code,
     *             hypothetical account liquidity in excess of collateral requirements,
     *          hypothetical account shortfall below collateral requirements)
     */
    /**
     * @notice 确定如果给定金额被赎回/借款，账户流动性会是多少
     *
     * @param cTokenModify 假设赎回/借款的市场
     *
     * @param帐户 确定流动性的帐户
     *
     * @param redeemTokens 假设赎回的代币数量
     *
     * @param borrowAmount 假设借入的标的金额
     *
     * @dev 请注意，我们使用存储的数据计算每个抵押品cToken的交换率存储，
     *
     * 不计算累计利息。
     *
     * @return（可能的错误代码，
     *
     * 假设账户流动性超过抵押品要求，
     *
     * 假设账户短缺低于抵押品要求）
     */
    function getHypotheticalAccountLiquidityInternal(
        address account,
        CToken cTokenModify,
        uint256 redeemTokens,
        uint256 borrowAmount
    ) internal view returns (Error, uint256, uint256) {
        AccountLiquidityLocalVars memory vars; // Holds all our calculation results
        uint256 oErr;

        // For each asset the account is in
        CToken[] memory assets = accountAssets[account];
        for (uint256 i = 0; i < assets.length; i++) {
            CToken asset = assets[i];

            // Read the balances and exchange rate from the cToken
            (oErr, vars.cTokenBalance, vars.borrowBalance, vars.exchangeRateMantissa) =
                asset.getAccountSnapshot(account);
            if (oErr != 0) {
                // semi-opaque error code, we assume NO_ERROR == 0 is invariant between upgrades
                return (Error.SNAPSHOT_ERROR, 0, 0);
            }
            vars.collateralFactor = Exp({mantissa: markets[address(asset)].collateralFactorMantissa});
            vars.exchangeRate = Exp({mantissa: vars.exchangeRateMantissa});

            // Get the normalized price of the asset
            vars.oraclePriceMantissa = oracle.getUnderlyingPrice(asset);
            if (vars.oraclePriceMantissa == 0) {
                return (Error.PRICE_ERROR, 0, 0);
            }
            vars.oraclePrice = Exp({mantissa: vars.oraclePriceMantissa});

            // Pre-compute a conversion factor from tokens -> ether (normalized price value)
            vars.tokensToDenom = mul_(mul_(vars.collateralFactor, vars.exchangeRate), vars.oraclePrice);

            // sumCollateral += tokensToDenom * cTokenBalance
            vars.sumCollateral = mul_ScalarTruncateAddUInt(vars.tokensToDenom, vars.cTokenBalance, vars.sumCollateral);

            // sumBorrowPlusEffects += oraclePrice * borrowBalance
            vars.sumBorrowPlusEffects =
                mul_ScalarTruncateAddUInt(vars.oraclePrice, vars.borrowBalance, vars.sumBorrowPlusEffects);

            // Calculate effects of interacting with cTokenModify
            if (asset == cTokenModify) {
                // redeem effect
                // sumBorrowPlusEffects += tokensToDenom * redeemTokens
                vars.sumBorrowPlusEffects =
                    mul_ScalarTruncateAddUInt(vars.tokensToDenom, redeemTokens, vars.sumBorrowPlusEffects);

                // borrow effect
                // sumBorrowPlusEffects += oraclePrice * borrowAmount
                vars.sumBorrowPlusEffects =
                    mul_ScalarTruncateAddUInt(vars.oraclePrice, borrowAmount, vars.sumBorrowPlusEffects);
            }
        }

        // These are safe, as the underflow condition is checked first
        if (vars.sumCollateral > vars.sumBorrowPlusEffects) {
            return (Error.NO_ERROR, vars.sumCollateral - vars.sumBorrowPlusEffects, 0);
        } else {
            return (Error.NO_ERROR, 0, vars.sumBorrowPlusEffects - vars.sumCollateral);
        }
    }

    /**
     * @notice Calculate number of tokens of collateral asset to seize given an underlying amount
     * @dev Used in liquidation (called in cToken.liquidateBorrowFresh)
     * @param cTokenBorrowed The address of the borrowed cToken
     * @param cTokenCollateral The address of the collateral cToken
     * @param actualRepayAmount The amount of cTokenBorrowed underlying to convert into cTokenCollateral tokens
     * @return (errorCode, number of cTokenCollateral tokens to be seized in a liquidation)
     */
    /**
     * @notice 计算要扣押给定基础金额的抵押资产的代币数量
     *
     * @dev用于清算（称为cToken.liquidateBorrowFresh）
     *
     * @param cTokenBorrowed 借来的cToken的地址
     *
     * @param cTokenCollateral 抵押品cToken的地址
     *
     * @param actualRepayAmount cTokenBorrowed的基础金额转换为cTokenCollateral代币
     *
     * @return（错误代码，清算中要扣押的cTokenCollateral代币数量）
     */
    function liquidateCalculateSeizeTokens(address cTokenBorrowed, address cTokenCollateral, uint256 actualRepayAmount)
        external
        view
        override
        returns (uint256, uint256)
    {
        /* Read oracle prices for borrowed and collateral markets */
        uint256 priceBorrowedMantissa = oracle.getUnderlyingPrice(CToken(cTokenBorrowed));
        uint256 priceCollateralMantissa = oracle.getUnderlyingPrice(CToken(cTokenCollateral));
        if (priceBorrowedMantissa == 0 || priceCollateralMantissa == 0) {
            return (uint256(Error.PRICE_ERROR), 0);
        }

        /*
         * Get the exchange rate and calculate the number of collateral tokens to seize:
         *  seizeAmount = actualRepayAmount * liquidationIncentive * priceBorrowed / priceCollateral
         *  seizeTokens = seizeAmount / exchangeRate
         *   = actualRepayAmount * (liquidationIncentive * priceBorrowed) / (priceCollateral * exchangeRate)
         */
        /*

        *获取汇率并计算要扣押的抵押代币数量：

        *扣押金额=实际还款金额*清算激励*价格借款/价格抵押品

        *扣押代币=扣押金额/汇率

        * = 实际还款金额*（清偿激励*借款价格）/（价格抵押品*汇率）

        */
        uint256 exchangeRateMantissa = CToken(cTokenCollateral).exchangeRateStored(); // Note: reverts on error
        uint256 seizeTokens;
        Exp memory numerator;
        Exp memory denominator;
        Exp memory ratio;

        numerator = mul_(Exp({mantissa: liquidationIncentiveMantissa}), Exp({mantissa: priceBorrowedMantissa}));
        denominator = mul_(Exp({mantissa: priceCollateralMantissa}), Exp({mantissa: exchangeRateMantissa}));
        ratio = div_(numerator, denominator);

        seizeTokens = mul_ScalarTruncate(ratio, actualRepayAmount);

        return (uint256(Error.NO_ERROR), seizeTokens);
    }

    /**
     * Admin Functions **
     */

    /**
     * @notice Sets a new price oracle for the comptroller
     * @dev Admin function to set a new price oracle
     * @return uint 0=success, otherwise a failure (see ErrorReporter.sol for details)
     */
    /**
     * @notice 为主计长设定了新的价格
     *
     * @dev 设置新价格甲骨文的管理功能
     *
     * @return uint 0=成功，否则失败（详情见ErrorReporter.sol）
     */
    function _setPriceOracle(PriceOracle newOracle) public returns (uint256) {
        // Check caller is admin
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SET_PRICE_ORACLE_OWNER_CHECK);
        }

        // Track the old oracle for the comptroller
        PriceOracle oldOracle = oracle;

        // Set comptroller's oracle to newOracle
        oracle = newOracle;

        // Emit NewPriceOracle(oldOracle, newOracle)
        emit NewPriceOracle(oldOracle, newOracle);

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Sets the closeFactor used when liquidating borrows
     * @dev Admin function to set closeFactor
     * @param newCloseFactorMantissa New close factor, scaled by 1e18
     * @return uint 0=success, otherwise a failure
     */
    /**
     * @notice 设置清算借款时使用的closeFactor
     *
     * 设置closeFactor的@dev管理功能
     *
     * @param newCloseFactorMantissa 新的接近系数，缩放为1e18
     *
     * @return uint 0=成功，否则失败
     */
    function _setCloseFactor(uint256 newCloseFactorMantissa) external returns (uint256) {
        // Check caller is admin
        require(msg.sender == admin, "only admin can set close factor");

        uint256 oldCloseFactorMantissa = closeFactorMantissa;
        closeFactorMantissa = newCloseFactorMantissa;
        emit NewCloseFactor(oldCloseFactorMantissa, closeFactorMantissa);

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Sets the collateralFactor for a market
     * @dev Admin function to set per-market collateralFactor
     * @param cToken The market to set the factor on
     * @param newCollateralFactorMantissa The new collateral factor, scaled by 1e18
     * @return uint 0=success, otherwise a failure. (See ErrorReporter for details)
     */
    /**
     * @notice 为市场设置抵押因素
     *
     * @Dev管理功能设置每个市场的抵押因素
     *
     * @param cToken 市场设定因素
     *
     * @param newCollateralFactorMantissa 新的附带因素，缩放为1e18
     *
     * @return uint 0=成功，否则失败。（详情请参阅ErrorReporter）
     */
    function _setCollateralFactor(CToken cToken, uint256 newCollateralFactorMantissa) external returns (uint256) {
        // Check caller is admin
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SET_COLLATERAL_FACTOR_OWNER_CHECK);
        }

        // Verify market is listed
        Market storage market = markets[address(cToken)];
        if (!market.isListed) {
            return fail(Error.MARKET_NOT_LISTED, FailureInfo.SET_COLLATERAL_FACTOR_NO_EXISTS);
        }

        Exp memory newCollateralFactorExp = Exp({mantissa: newCollateralFactorMantissa});

        // Check collateral factor <= 0.9
        Exp memory highLimit = Exp({mantissa: collateralFactorMaxMantissa});
        if (lessThanExp(highLimit, newCollateralFactorExp)) {
            return fail(Error.INVALID_COLLATERAL_FACTOR, FailureInfo.SET_COLLATERAL_FACTOR_VALIDATION);
        }

        // If collateral factor != 0, fail if price == 0
        if (newCollateralFactorMantissa != 0 && oracle.getUnderlyingPrice(cToken) == 0) {
            return fail(Error.PRICE_ERROR, FailureInfo.SET_COLLATERAL_FACTOR_WITHOUT_PRICE);
        }

        // Set market's collateral factor to new collateral factor, remember old value
        uint256 oldCollateralFactorMantissa = market.collateralFactorMantissa;
        market.collateralFactorMantissa = newCollateralFactorMantissa;

        // Emit event with asset, old collateral factor, and new collateral factor
        emit NewCollateralFactor(cToken, oldCollateralFactorMantissa, newCollateralFactorMantissa);

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Sets liquidationIncentive
     * @dev Admin function to set liquidationIncentive
     * @param newLiquidationIncentiveMantissa New liquidationIncentive scaled by 1e18
     * @return uint 0=success, otherwise a failure. (See ErrorReporter for details)
     */
    /**
     * @notice设置清算激励
     *
     * @dev管理功能设置清算激励
     *
     * @param newLiquidationIncentiveMantissa 新清算激励扩大了1e18
     *
     * @return uint 0=成功，否则失败。 （详情请参阅ErrorReporter）
     */
    function _setLiquidationIncentive(uint256 newLiquidationIncentiveMantissa) external returns (uint256) {
        // Check caller is admin
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SET_LIQUIDATION_INCENTIVE_OWNER_CHECK);
        }

        // Save current value for use in log
        uint256 oldLiquidationIncentiveMantissa = liquidationIncentiveMantissa;

        // Set liquidation incentive to new incentive
        liquidationIncentiveMantissa = newLiquidationIncentiveMantissa;

        // Emit event with old incentive, new incentive
        emit NewLiquidationIncentive(oldLiquidationIncentiveMantissa, newLiquidationIncentiveMantissa);

        return uint256(Error.NO_ERROR);
    }

    /**
     * @notice Add the market to the markets mapping and set it as listed
     * @dev Admin function to set isListed and add support for the market
     * @param cToken The address of the market (token) to list
     * @return uint 0=success, otherwise a failure. (See enum Error for details)
     */
    /**
     * @notice 将市场添加到市场映射中，并将其设置为上市
     *
     * @dev管理功能设置isListed并添加对市场的支持
     *
     * @param cToken 上市的市场地址（代币）
     *
     * @return uint 0=成功，否则失败。（详情见枚列错误）
     */
    function _supportMarket(CToken cToken) external returns (uint256) {
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SUPPORT_MARKET_OWNER_CHECK);
        }

        if (markets[address(cToken)].isListed) {
            return fail(Error.MARKET_ALREADY_LISTED, FailureInfo.SUPPORT_MARKET_EXISTS);
        }

        cToken.isCToken(); // Sanity check to make sure its really a CToken

        // Note that isComped is not in active use anymore
        Market storage newMarket = markets[address(cToken)];
        newMarket.isListed = true;
        newMarket.isComped = false;
        newMarket.collateralFactorMantissa = 0;

        _addMarketInternal(address(cToken));
        _initializeMarket(address(cToken));

        emit MarketListed(cToken);

        return uint256(Error.NO_ERROR);
    }

    function _addMarketInternal(address cToken) internal {
        for (uint256 i = 0; i < allMarkets.length; i++) {
            require(allMarkets[i] != CToken(cToken), "market already added");
        }
        allMarkets.push(CToken(cToken));
    }

    function _initializeMarket(address cToken) internal {
        uint32 blockNumber = safe32(getBlockNumber(), "block number exceeds 32 bits");

        CompMarketState storage supplyState = compSupplyState[cToken];
        CompMarketState storage borrowState = compBorrowState[cToken];

        /*
         * Update market state indices
         */
        if (supplyState.index == 0) {
            // Initialize supply state index with default value
            supplyState.index = compInitialIndex;
        }

        if (borrowState.index == 0) {
            // Initialize borrow state index with default value
            borrowState.index = compInitialIndex;
        }

        /*
         * Update market state block numbers
         */
        supplyState.block = borrowState.block = blockNumber;
    }

    /**
     * @notice Set the given borrow caps for the given cToken markets. Borrowing that brings total borrows to or above borrow cap will revert.
     * @dev Admin or borrowCapGuardian function to set the borrow caps. A borrow cap of 0 corresponds to unlimited borrowing.
     * @param cTokens The addresses of the markets (tokens) to change the borrow caps for
     * @param newBorrowCaps The new borrow cap values in underlying to be set. A value of 0 corresponds to unlimited borrowing.
     */
    /**
     * @notice 为给定的cToken市场设置给定的借款上限。使借款总额达到或超过借款上限的借款将恢复。
     *
     * @dev Admin或borneCapGuardian功能来设置借用上限。借款上限为0对应于无限借款。
     *
     * @param cTokens 改变借款上限的市场地址（代币）
     *
     * @param newBorrowCaps 底层的新借款上限值待设置。0的值对应于无限制的借款。
     */
    function _setMarketBorrowCaps(CToken[] calldata cTokens, uint256[] calldata newBorrowCaps) external {
        require(
            msg.sender == admin || msg.sender == borrowCapGuardian,
            "only admin or borrow cap guardian can set borrow caps"
        );

        uint256 numMarkets = cTokens.length;
        uint256 numBorrowCaps = newBorrowCaps.length;

        require(numMarkets != 0 && numMarkets == numBorrowCaps, "invalid input");

        for (uint256 i = 0; i < numMarkets; i++) {
            borrowCaps[address(cTokens[i])] = newBorrowCaps[i];
            emit NewBorrowCap(cTokens[i], newBorrowCaps[i]);
        }
    }

    /**
     * @notice Admin function to change the Borrow Cap Guardian
     * @param newBorrowCapGuardian The address of the new Borrow Cap Guardian
     */
    function _setBorrowCapGuardian(address newBorrowCapGuardian) external {
        require(msg.sender == admin, "only admin can set borrow cap guardian");

        // Save current value for inclusion in log
        address oldBorrowCapGuardian = borrowCapGuardian;

        // Store borrowCapGuardian with value newBorrowCapGuardian
        borrowCapGuardian = newBorrowCapGuardian;

        // Emit NewBorrowCapGuardian(OldBorrowCapGuardian, NewBorrowCapGuardian)
        emit NewBorrowCapGuardian(oldBorrowCapGuardian, newBorrowCapGuardian);
    }

    /**
     * @notice Admin function to change the Pause Guardian
     * @param newPauseGuardian The address of the new Pause Guardian
     * @return uint 0=success, otherwise a failure. (See enum Error for details)
     */
    /**
     * @notice 更改暂停守护者的管理功能
     *
     * @param newPauseGuardian 新Pause Guardian的地址
     *
     * @return uint 0=成功，否则失败。（详情见枚列错误）
     */
    function _setPauseGuardian(address newPauseGuardian) public returns (uint256) {
        if (msg.sender != admin) {
            return fail(Error.UNAUTHORIZED, FailureInfo.SET_PAUSE_GUARDIAN_OWNER_CHECK);
        }

        // Save current value for inclusion in log
        address oldPauseGuardian = pauseGuardian;

        // Store pauseGuardian with value newPauseGuardian
        pauseGuardian = newPauseGuardian;

        // Emit NewPauseGuardian(OldPauseGuardian, NewPauseGuardian)
        emit NewPauseGuardian(oldPauseGuardian, pauseGuardian);

        return uint256(Error.NO_ERROR);
    }

    function _setMintPaused(CToken cToken, bool state) public returns (bool) {
        require(markets[address(cToken)].isListed, "cannot pause a market that is not listed");
        require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
        require(msg.sender == admin || state == true, "only admin can unpause");

        mintGuardianPaused[address(cToken)] = state;
        emit ActionPaused(cToken, "Mint", state);
        return state;
    }

    function _setBorrowPaused(CToken cToken, bool state) public returns (bool) {
        require(markets[address(cToken)].isListed, "cannot pause a market that is not listed");
        require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
        require(msg.sender == admin || state == true, "only admin can unpause");

        borrowGuardianPaused[address(cToken)] = state;
        emit ActionPaused(cToken, "Borrow", state);
        return state;
    }

    function _setTransferPaused(bool state) public returns (bool) {
        require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
        require(msg.sender == admin || state == true, "only admin can unpause");

        transferGuardianPaused = state;
        emit ActionPaused("Transfer", state);
        return state;
    }

    function _setSeizePaused(bool state) public returns (bool) {
        require(msg.sender == pauseGuardian || msg.sender == admin, "only pause guardian and admin can pause");
        require(msg.sender == admin || state == true, "only admin can unpause");

        seizeGuardianPaused = state;
        emit ActionPaused("Seize", state);
        return state;
    }

    function _become(Unitroller unitroller) public {
        require(msg.sender == unitroller.admin(), "only unitroller admin can change brains");
        require(unitroller._acceptImplementation() == 0, "change not authorized");
    }

    /// @notice Delete this function after proposal 65 is executed
    function fixBadAccruals(address[] calldata affectedUsers, uint256[] calldata amounts) external {
        require(msg.sender == admin, "Only admin can call this function"); // Only the timelock can call this function
        require(!proposal65FixExecuted, "Already executed this one-off function"); // Require that this function is only called once
        require(affectedUsers.length == amounts.length, "Invalid input");

        // Loop variables
        address user;
        uint256 currentAccrual;
        uint256 amountToSubtract;
        uint256 newAccrual;

        // Iterate through all affected users
        for (uint256 i = 0; i < affectedUsers.length; ++i) {
            user = affectedUsers[i];
            currentAccrual = compAccrued[user];

            amountToSubtract = amounts[i];

            // The case where the user has claimed and received an incorrect amount of COMP.
            // The user has less currently accrued than the amount they incorrectly received.
            if (amountToSubtract > currentAccrual) {
                // Amount of COMP the user owes the protocol
                uint256 accountReceivable = amountToSubtract - currentAccrual; // Underflow safe since amountToSubtract > currentAccrual

                uint256 oldReceivable = compReceivable[user];
                uint256 newReceivable = add_(oldReceivable, accountReceivable);

                // Accounting: record the COMP debt for the user
                compReceivable[user] = newReceivable;

                emit CompReceivableUpdated(user, oldReceivable, newReceivable);

                amountToSubtract = currentAccrual;
            }

            if (amountToSubtract > 0) {
                // Subtract the bad accrual amount from what they have accrued.
                // Users will keep whatever they have correctly accrued.
                compAccrued[user] = newAccrual = sub_(currentAccrual, amountToSubtract);

                emit CompAccruedAdjusted(user, currentAccrual, newAccrual);
            }
        }

        proposal65FixExecuted = true; // Makes it so that this function cannot be called again
    }

    /**
     * @notice Checks caller is admin, or this contract is becoming the new implementation
     */
    function adminOrInitializing() internal view returns (bool) {
        return msg.sender == admin || msg.sender == comptrollerImplementation;
    }

    /**
     * Comp Distribution **
     */

    /**
     * @notice Set COMP speed for a single market
     * @param cToken The market whose COMP speed to update
     * @param supplySpeed New supply-side COMP speed for market
     * @param borrowSpeed New borrow-side COMP speed for market
     */
    /**
     * @notice 为单一市场设置COMP速度
     *
     * @param cToken 其COMP速度更新的市场
     *
     * @param supplySpeed 市场新的供应方COMP速度
     *
     * @param borrowSpeed 市场新的借款方COMP速度
     */
    function setCompSpeedInternal(CToken cToken, uint256 supplySpeed, uint256 borrowSpeed) internal {
        Market storage market = markets[address(cToken)];
        require(market.isListed, "comp market is not listed");

        if (compSupplySpeeds[address(cToken)] != supplySpeed) {
            // Supply speed updated so let's update supply state to ensure that
            //  1. COMP accrued properly for the old speed, and
            //  2. COMP accrued at the new speed starts after this block.
            updateCompSupplyIndex(address(cToken));

            // Update speed and emit event
            compSupplySpeeds[address(cToken)] = supplySpeed;
            emit CompSupplySpeedUpdated(cToken, supplySpeed);
        }

        if (compBorrowSpeeds[address(cToken)] != borrowSpeed) {
            // Borrow speed updated so let's update borrow state to ensure that
            //  1. COMP accrued properly for the old speed, and
            //  2. COMP accrued at the new speed starts after this block.
            Exp memory borrowIndex = Exp({mantissa: cToken.borrowIndex()});
            updateCompBorrowIndex(address(cToken), borrowIndex);

            // Update speed and emit event
            compBorrowSpeeds[address(cToken)] = borrowSpeed;
            emit CompBorrowSpeedUpdated(cToken, borrowSpeed);
        }
    }

    /**
     * @notice Accrue COMP to the market by updating the supply index
     * @param cToken The market whose supply index to update
     * @dev Index is a cumulative sum of the COMP per cToken accrued.
     */
    /**
     * @notice 通过更新供应指数向市场累积COMP
     *
     * @param cToken 供应指数更新的市场
     *
     * @dev指数是每cToken累积的COMP的累积总和。
     */
    function updateCompSupplyIndex(address cToken) internal {
        CompMarketState storage supplyState = compSupplyState[cToken];
        uint256 supplySpeed = compSupplySpeeds[cToken];
        uint32 blockNumber = safe32(getBlockNumber(), "block number exceeds 32 bits");
        uint256 deltaBlocks = sub_(uint256(blockNumber), uint256(supplyState.block));
        if (deltaBlocks > 0 && supplySpeed > 0) {
            uint256 supplyTokens = CToken(cToken).totalSupply();
            uint256 compAccrued = mul_(deltaBlocks, supplySpeed);
            Double memory ratio = supplyTokens > 0 ? fraction(compAccrued, supplyTokens) : Double({mantissa: 0});
            supplyState.index =
                safe224(add_(Double({mantissa: supplyState.index}), ratio).mantissa, "new index exceeds 224 bits");
            supplyState.block = blockNumber;
        } else if (deltaBlocks > 0) {
            supplyState.block = blockNumber;
        }
    }

    /**
     * @notice Accrue COMP to the market by updating the borrow index
     * @param cToken The market whose borrow index to update
     * @dev Index is a cumulative sum of the COMP per cToken accrued.
     */
    /**
     * @notice 通过更新借款指数向市场累积COMP
     *
     * @param cToken 借款指数更新的市场
     *
     * @dev指数是每cToken累积的COMP的累积总和。
     */
    function updateCompBorrowIndex(address cToken, Exp memory marketBorrowIndex) internal {
        CompMarketState storage borrowState = compBorrowState[cToken];
        uint256 borrowSpeed = compBorrowSpeeds[cToken];
        uint32 blockNumber = safe32(getBlockNumber(), "block number exceeds 32 bits");
        uint256 deltaBlocks = sub_(uint256(blockNumber), uint256(borrowState.block));
        if (deltaBlocks > 0 && borrowSpeed > 0) {
            uint256 borrowAmount = div_(CToken(cToken).totalBorrows(), marketBorrowIndex);
            uint256 compAccrued = mul_(deltaBlocks, borrowSpeed);
            Double memory ratio = borrowAmount > 0 ? fraction(compAccrued, borrowAmount) : Double({mantissa: 0});
            borrowState.index =
                safe224(add_(Double({mantissa: borrowState.index}), ratio).mantissa, "new index exceeds 224 bits");
            borrowState.block = blockNumber;
        } else if (deltaBlocks > 0) {
            borrowState.block = blockNumber;
        }
    }

    /**
     * @notice Calculate COMP accrued by a supplier and possibly transfer it to them
     * @param cToken The market in which the supplier is interacting
     * @param supplier The address of the supplier to distribute COMP to
     */
    /**
     * @notice 计算供应商应计的COMP，并可能将其转让给他们
     *
     * @param cToken 供应商互动的市场
     *
     * @param供应商将COMP分发给的供应商地址
     */
    function distributeSupplierComp(address cToken, address supplier) internal {
        // TODO: Don't distribute supplier COMP if the user is not in the supplier market.
        // This check should be as gas efficient as possible as distributeSupplierComp is called in many places.
        // - We really don't want to call an external contract as that's quite expensive.
        // TODO：如果用户不在供应商市场，请不要分发供应商COMP。

        //这种检查应该尽可能的气体效率，因为在许多地方被称为distributeSupplierComp。

        // - 我们真的不想调用external合同，因为那相当昂贵。

        CompMarketState storage supplyState = compSupplyState[cToken];
        uint256 supplyIndex = supplyState.index;
        uint256 supplierIndex = compSupplierIndex[cToken][supplier];

        // Update supplier's index to the current index since we are distributing accrued COMP
        compSupplierIndex[cToken][supplier] = supplyIndex;

        if (supplierIndex == 0 && supplyIndex >= compInitialIndex) {
            // Covers the case where users supplied tokens before the market's supply state index was set.
            // Rewards the user with COMP accrued from the start of when supplier rewards were first
            // set for the market.
            supplierIndex = compInitialIndex;
        }

        // Calculate change in the cumulative sum of the COMP per cToken accrued
        Double memory deltaIndex = Double({mantissa: sub_(supplyIndex, supplierIndex)});

        uint256 supplierTokens = CToken(cToken).balanceOf(supplier);

        // Calculate COMP accrued: cTokenAmount * accruedPerCToken
        uint256 supplierDelta = mul_(supplierTokens, deltaIndex);

        uint256 supplierAccrued = add_(compAccrued[supplier], supplierDelta);
        compAccrued[supplier] = supplierAccrued;

        emit DistributedSupplierComp(CToken(cToken), supplier, supplierDelta, supplyIndex);
    }

    /**
     * @notice Calculate COMP accrued by a borrower and possibly transfer it to them
     * @dev Borrowers will not begin to accrue until after the first interaction with the protocol.
     * @param cToken The market in which the borrower is interacting
     * @param borrower The address of the borrower to distribute COMP to
     */
    /**
     * @notice 计算借款人应计的COMP，并可能将其转让给他们
     *
     * @Dev借款人在与协议的首次交互后才会开始累积。
     *
     * @param cToken 借款人正在互动的市场
     *
     * @param借款人 分配COMP的借款人的地址
     */
    function distributeBorrowerComp(address cToken, address borrower, Exp memory marketBorrowIndex) internal {
        // TODO: Don't distribute supplier COMP if the user is not in the borrower market.
        // This check should be as gas efficient as possible as distributeBorrowerComp is called in many places.
        // - We really don't want to call an external contract as that's quite expensive.

        CompMarketState storage borrowState = compBorrowState[cToken];
        uint256 borrowIndex = borrowState.index;
        uint256 borrowerIndex = compBorrowerIndex[cToken][borrower];

        // Update borrowers's index to the current index since we are distributing accrued COMP
        compBorrowerIndex[cToken][borrower] = borrowIndex;

        if (borrowerIndex == 0 && borrowIndex >= compInitialIndex) {
            // Covers the case where users borrowed tokens before the market's borrow state index was set.
            // Rewards the user with COMP accrued from the start of when borrower rewards were first
            // set for the market.
            //涵盖用户在设定市场借款状态指数之前借入代币的情况。

            //从借款人奖励开始，用累积的COMP奖励用户

            //为市场设置。
            borrowerIndex = compInitialIndex;
        }

        // Calculate change in the cumulative sum of the COMP per borrowed unit accrued
        Double memory deltaIndex = Double({mantissa: sub_(borrowIndex, borrowerIndex)});

        uint256 borrowerAmount = div_(CToken(cToken).borrowBalanceStored(borrower), marketBorrowIndex);

        // Calculate COMP accrued: cTokenAmount * accruedPerBorrowedUnit
        uint256 borrowerDelta = mul_(borrowerAmount, deltaIndex);

        uint256 borrowerAccrued = add_(compAccrued[borrower], borrowerDelta);
        compAccrued[borrower] = borrowerAccrued;

        emit DistributedBorrowerComp(CToken(cToken), borrower, borrowerDelta, borrowIndex);
    }

    /**
     * @notice Calculate additional accrued COMP for a contributor since last accrual
     * @param contributor The address to calculate contributor rewards for
     */
    /**
     * @notice 计算自上次应计以来贡献者的额外应计 COMP
     *
     * @param 贡献者 计算贡献者奖励的地址
     */
    function updateContributorRewards(address contributor) public {
        uint256 compSpeed = compContributorSpeeds[contributor];
        uint256 blockNumber = getBlockNumber();
        uint256 deltaBlocks = sub_(blockNumber, lastContributorBlock[contributor]);
        if (deltaBlocks > 0 && compSpeed > 0) {
            uint256 newAccrued = mul_(deltaBlocks, compSpeed);
            uint256 contributorAccrued = add_(compAccrued[contributor], newAccrued);

            compAccrued[contributor] = contributorAccrued;
            lastContributorBlock[contributor] = blockNumber;
        }
    }

    /**
     * @notice Claim all the comp accrued by holder in all markets
     * @param holder The address to claim COMP for
     */
    /**
     * @notice 在所有市场中索要持有人应计的所有补偿
     *
     * @param持有人 申请COMP的地址
     */
    function claimComp(address holder) public {
        return claimComp(holder, allMarkets);
    }

    /**
     * @notice Claim all the comp accrued by holder in the specified markets
     * @param holder The address to claim COMP for
     * @param cTokens The list of markets to claim COMP in
     */
    /**
     * @notice 领取持有人在指定市场中累积的所有补偿
     *
     * @param持有人 申请COMP的地址
     *
     * @param cTokens 申请COMP的市场列表
     */
    function claimComp(address holder, CToken[] memory cTokens) public {
        address[] memory holders = new address[](1);
        holders[0] = holder;
        claimComp(holders, cTokens, true, true);
    }

    /**
     * @notice Claim all comp accrued by the holders
     * @param holders The addresses to claim COMP for
     * @param cTokens The list of markets to claim COMP in
     * @param borrowers Whether or not to claim COMP earned by borrowing
     * @param suppliers Whether or not to claim COMP earned by supplying
     */
    /**
     * @notice 索赔持有人应计的所有补偿
     *
     * @param持有人申请COMP的地址
     *
     * @param cTokens 申请COMP的市场列表
     *
     * @param 借款人 是否申请通过借款获得的 COMP
     *
     * @param供应商是否要求通过供应获得的COMP
     */
    function claimComp(address[] memory holders, CToken[] memory cTokens, bool borrowers, bool suppliers) public {
        for (uint256 i = 0; i < cTokens.length; i++) {
            CToken cToken = cTokens[i];
            require(markets[address(cToken)].isListed, "market must be listed");
            if (borrowers == true) {
                Exp memory borrowIndex = Exp({mantissa: cToken.borrowIndex()});
                updateCompBorrowIndex(address(cToken), borrowIndex);
                for (uint256 j = 0; j < holders.length; j++) {
                    distributeBorrowerComp(address(cToken), holders[j], borrowIndex);
                }
            }
            if (suppliers == true) {
                updateCompSupplyIndex(address(cToken));
                for (uint256 j = 0; j < holders.length; j++) {
                    distributeSupplierComp(address(cToken), holders[j]);
                }
            }
        }
        for (uint256 j = 0; j < holders.length; j++) {
            compAccrued[holders[j]] = grantCompInternal(holders[j], compAccrued[holders[j]]);
        }
    }

    /**
     * @notice Transfer COMP to the user
     * @dev Note: If there is not enough COMP, we do not perform the transfer all.
     * @param user The address of the user to transfer COMP to
     * @param amount The amount of COMP to (possibly) transfer
     * @return The amount of COMP which was NOT transferred to the user
     */
    /**
     * @notice 将COMP传输给用户
     *
     * @dev 注意：如果没有足够的COMP，我们不会执行全部传输。
     *
     * @param用户将COMP传输到的用户地址
     *
     * @param金额 COMP（可能）转移的金额
     *
     * @return 未转让给用户的COMP金额
     */
    function grantCompInternal(address user, uint256 amount) internal returns (uint256) {
        Comp comp = Comp(getCompAddress());
        uint256 compRemaining = comp.balanceOf(address(this));
        if (amount > 0 && amount <= compRemaining) {
            comp.transfer(user, amount);
            return 0;
        }
        return amount;
    }

    /**
     * Comp Distribution Admin **
     */

    /**
     * @notice Transfer COMP to the recipient
     * @dev Note: If there is not enough COMP, we do not perform the transfer all.
     * @param recipient The address of the recipient to transfer COMP to
     * @param amount The amount of COMP to (possibly) transfer
     */
    /**
     * @notice 将COMP转让给收件人
     *
     * @dev 注意：如果没有足够的COMP，我们不会执行全部传输。
     *
     * @param收件人 接收者将COMP转移到的地址
     *
     * @param金额 COMP（可能）转移的金额
     */
    function _grantComp(address recipient, uint256 amount) public {
        require(adminOrInitializing(), "only admin can grant comp");
        uint256 amountLeft = grantCompInternal(recipient, amount);
        require(amountLeft == 0, "insufficient comp for grant");
        emit CompGranted(recipient, amount);
    }

    /**
     * @notice Set COMP borrow and supply speeds for the specified markets.
     * @param cTokens The markets whose COMP speed to update.
     * @param supplySpeeds New supply-side COMP speed for the corresponding market.
     * @param borrowSpeeds New borrow-side COMP speed for the corresponding market.
     */
    /**
     * @notice 为指定市场设置 COMP 借款和供应速度。
     *
     * @param cTokens 其COMP速度更新的市场。
     *
     * @param supplySpeeds 相应市场的新供应方COMP速度。
     *
     * @param borrowSpeeds 相应市场的新借款方COMP速度。
     */
    function _setCompSpeeds(CToken[] memory cTokens, uint256[] memory supplySpeeds, uint256[] memory borrowSpeeds)
        public
    {
        require(adminOrInitializing(), "only admin can set comp speed");

        uint256 numTokens = cTokens.length;
        require(
            numTokens == supplySpeeds.length && numTokens == borrowSpeeds.length,
            "Comptroller::_setCompSpeeds invalid input"
        );

        for (uint256 i = 0; i < numTokens; ++i) {
            setCompSpeedInternal(cTokens[i], supplySpeeds[i], borrowSpeeds[i]);
        }
    }

    /**
     * @notice Set COMP speed for a single contributor
     * @param contributor The contributor whose COMP speed to update
     * @param compSpeed New COMP speed for contributor
     */
    /**
     * @notice 为单个贡献者设置COMP速度
     *
     * @param贡献者 贡献者，其COMP更新速度
     *
     * @param compSpeed 贡献者的新COMP速度
     */
    function _setContributorCompSpeed(address contributor, uint256 compSpeed) public {
        require(adminOrInitializing(), "only admin can set comp speed");

        // note that COMP speed could be set to 0 to halt liquidity rewards for a contributor
        updateContributorRewards(contributor);
        if (compSpeed == 0) {
            // release storage
            delete lastContributorBlock[contributor];
        } else {
            lastContributorBlock[contributor] = getBlockNumber();
        }
        compContributorSpeeds[contributor] = compSpeed;

        emit ContributorCompSpeedUpdated(contributor, compSpeed);
    }

    /**
     * @notice Return all of the markets
     * @dev The automatic getter may be used to access an individual market.
     * @return The list of market addresses
     */
    /**
     * @notice 返回所有市场
     *
     * @dev 自动获取器可用于访问单个市场。
     *
     * @return 市场地址列表
     */
    function getAllMarkets() public view returns (CToken[] memory) {
        return allMarkets;
    }

    /**
     * @notice Returns true if the given cToken market has been deprecated
     * @dev All borrows in a deprecated cToken market can be immediately liquidated
     * @param cToken The market to check if deprecated
     */
    /**
     * 如果給定的cToken市場已被棄用，@notice返回true
     *
     * @dev 弃用的cToken市场中的所有借款都可以立即清算
     *
     * @param cToken 市场检查是否已弃用
     */
    function isDeprecated(CToken cToken) public view returns (bool) {
        return markets[address(cToken)].collateralFactorMantissa == 0 && borrowGuardianPaused[address(cToken)] == true
            && cToken.reserveFactorMantissa() == 1e18;
    }

    function getBlockNumber() public view virtual returns (uint256) {
        return block.number;
    }

    /**
     * @notice Return the address of the COMP token
     * @return The address of COMP
     */
    /**
     * @notice 返回COMP令牌的地址
     *
     * @return COMP的地址
     */
    function getCompAddress() public view virtual returns (address) {
        return 0xc00e94Cb662C3520282E6f5717214004A7f26888;
    }
}

```

* markets：记录哪些 cToken 被支持，每个 market 的抵押因子（collateral factor）等。
* user membership：哪个用户进入了哪些 market（enterMarkets / exitMarket）。
* 风险检查回调：cToken 在 mint/redeem/borrow/repay/liquidate 等关键操作里会调用 Comptroller 的 `*Allowed` 和 `*Verify` 函数来询问是否允许操作（权限/风险/限额检查）。

# 函数

## `mintAllowed()`

被所有 cToken 调用，用于在**用户执行 mint() 前风控检查**

```solidity
function mintAllowed(address cToken, address minter, uint256 mintAmount) external override returns (uint256) {
        // Pausing is a very serious situation - we revert to sound the alarms
        require(!mintGuardianPaused[cToken], "mint is paused");
        //Guardian 模式。
        //在 Compound 升级中，增加了一个“紧急开关”（guardian），当出现漏洞或风险时，可以暂停某个市场的 mint 行为

        // Shh - currently unused
        minter;
        mintAmount;//用来“避免未使用变量警告”，这里暂时没逻辑意义

        if (!markets[cToken].isListed) {
            //检查市场是否被列入
            return uint256(Error.MARKET_NOT_LISTED);
        }

        // Keep the flywheel moving
        updateCompSupplyIndex(cToken); //更新该市场的 COMP 奖励累计指数；
        distributeSupplierComp(cToken, minter); //给当前存钱的用户 minter 发放他应得的 COMP

        return uint256(Error.NO_ERROR);
        //只要没被暂停、市场存在、分发成功，就返回 0（NO_ERROR）。
        //否则，cToken 合约那边会收到错误码并中止执行
    }
```

## `redeemAllowed()`

* `borrowAllowed(address cToken, address borrower, uint borrowAmount)` — 借款前的关键检查：是否有足够抵押、是否超过借款上限、市场状态等。
* `repayBorrowAllowed(...)` — 偿还借款的授权检查（通常允许）。
* `exitMarket(address cTokenAddress)` — 退出某个市场（取消作为抵押）。
  * `enterMarkets(address[] calldata cTokens)` — 用户把某 cToken 标记为用于抵押（影响可借额）。
* `liquidateBorrowAllowed(address cTokenBorrowed, address cTokenCollateral, address liquidator, address borrower, uint repayAmount)` — 清算是否被允许（是否欠息太多，是否超过清算门槛）。
* `seizeAllowed(...)` — 清算完成后，cToken 之间没收抵押资产的检查。
* `getAccountLiquidity(address account)`（或内部实现） — 计算用户可借余量和是否欠维持保证金，核心的风险数学在这里或被多处调用。

# 边角料函数

# 🟩 PART 1 — COMP 分发机制（奖励系统）

这部分就是 Compound V2 的 **流动性奖励系统（Liquidity Mining）**。\
关键点：\
用户会因为 **借款** 和 **供应** 获得 COMP。

对应数据结构：

* 每个市场有 supplyIndex、borrowIndex（全局累计的奖励指数）
* 每个用户对每个市场有自己的 index（记录他上次领取时的“进度”）
* compAccrued 存“已累积但还没转给用户”的 COMP

## 🍀 1. `distributeBorrowerComp`

**一句话：给借款人根据 borrowIndex 分发 COMP。**

流程步骤（非常重要）：

1. 取出市场的 borrowState.index\
   → 这是“全局奖励进度条”
2. 取出 borrower 上次的 borrowIndex\
   → 这是“用户上次领奖时的进度条”
3. 把用户 index 更新为最新：

```plain
compBorrowerIndex[cToken][borrower] = borrowIndex
```

4. 特殊情况：\
   borrowerIndex == 0 且 borrowIndex >= compInitialIndex\
   → 则 borrowerIndex = 初始奖励点（补发）
5. 计算 deltaIndex

```plain
deltaIndex = borrowIndex - borrowerIndex
```

6. 把借款余额 normalize：

```plain
borrowerAmount = borrowBalanceStored / marketBorrowIndex
```

为什么要这么算？\
因为 borrowIndex 会一直增长，需要把“当前借款量”换算成“对应在奖励系统的单位”。

7. 计算奖励：

```plain
borrowerDelta = borrowerAmount * deltaIndex
```

8. 存入 compAccrued：

```plain
compAccrued[borrower] += borrowerDelta
```

👉 **注意**：这里只是“登记奖励”，不会立刻转给用户。

***

## 🍀 2. `updateContributorRewards`

**一句话：给协议贡献者（不是借/贷者）按每区块速度发 COMP。**

贡献者是可以自定义的，比如预留给开发者团队。

流程：

1. 读取上一次领取的块号
2. 算出 deltaBlocks
3. 奖励 = deltaBlocks \* speed
4. 加进 compAccrued

***

## 🍀 3. `claimComp(holder)`

**一句话：领取自己所有市场的 COMP。**

它直接调用：

```plain
claimComp(holder, allMarkets)
```

***

## 🍀 4. `claimComp(holder, cTokens)`

**一句话：只领取某些市场的 COMP。**

***

## 🍀 5. 最根本的 `claimComp(holders, cTokens, borrowers, suppliers)`

**一句话：批量领取奖励 + 分发供应/借款奖励 + 最终转账。**

流程精华：

### 第 1 层：遍历每个市场

对每个 cToken：

* 如果 borrowers == true：
  * 更新市场 borrowIndex
  * 给每个 holder 分发 borrower 奖励
* 如果 suppliers == true：
  * 更新 supplyIndex
  * 给每个 holder 分发 supplier 奖励

### 第 2 层：实际把奖励转给用户

```plain
compAccrued[user] = grantCompInternal(user, compAccrued[user])
```

全部奖励最终通过 `grantCompInternal` 发出去。

***

## 🍀 6. `grantCompInternal`

**一句话：检查余额够不够 → 给用户转 COMP。**

如果余额不足 → 不发，返回未发出的部分。

***

## 🍀 7. `_grantComp`

**Admin 手动给某人发 COMP。**

***

## 🍀 8. `_setCompSpeeds`

**一句话：设置每个市场的 supplySpeed / borrowSpeed（奖励速度）。**

每个市场的速度控制了：

* 每区块发多少 COMP 给借款人
* 每区块发多少 COMP 给供应者

这是 Liquidity Mining 激励机制的核心参数。

***

## 🍀 9. `_setContributorCompSpeed`

**一句话：给某个 contributor 设置每区块奖励速度。**

与 updateContributorRewards 配套使用。

***

## 🍀 10. `getAllMarkets`

返回所有市场。

***

## 🍀 11. `isDeprecated(cToken)`

市场废弃的判断条件（都是硬指标）：

* collateralFactor = 0（不能再作抵押了）
* borrowGuardianPaused = true（借款暂停）
* reserveFactor = 1e18（100% 收入全部进储备，市场不再运作）

废弃市场内的借款可以被立即清算。

***

## 🍀 12. getBlockNumber / getCompAddress

简单工具函数。

***

# 🟦 PART 2 — 现在把整个 Comptroller 全部功能做一份「顶级总览图」

我把你这几天学的所有函数全部分类、串联成一份 **超级清晰的 map** ↓

***

# 🧩 **Comptroller —— 全局“权限 + 流动性 + 奖励控制中心”**

## 🔵 1. 市场管理（抵押品加入/退出）

| 功能 | 函数 |
| --- | --- |
| 加入多个抵押品市场 | enterMarkets |
| 加入单个抵押品市场（内部） | addToMarketInternal |
| 退出市场 | exitMarket |
| 获取某用户的抵押品市场列表 | getAssetsIn |
| 判断某市场是否已加入 | checkMembership |

***

## 🔴 2. 流动性与清算逻辑

| 功能 | 函数 |
| --- | --- |
| 流动性计算（最核心） | getAccountLiquidity / getHypotheticalAccountLiquidity |
| 清算前验证 | liquidateBorrowAllowed |
| 清算执行校验 | liquidateBorrowVerify |
| 计算 seizeTokens | liquidateCalculateSeizeTokens |
| seize 前权限检查 | seizeAllowed |
| seize 后 hook | seizeVerify |

***

## 🟡 3. 转账（cToken transfer）限制

| 功能 | 函数 |
| --- | --- |
| 判断是否允许转出（是否破坏抵押） | transferAllowed |
| 完成后 hook | transferVerify |

***

## 🟢 4. COMP 激励系统（你最新这整段）

| 功能 | 函数 |
| --- | --- |
| 供应奖励发放 | distributeSupplierComp |
| 借款奖励发放 | distributeBorrowerComp |
| 更新 supply / borrow index | updateCompSupplyIndex / updateCompBorrowIndex |
| 手动领取 COMP | claimComp |
| 批量领取 COMP | claimComp(holders, markets) |
| 设置速度（市场 reward） | \_setCompSpeeds |
| 设置速度（contributor reward） | \_setContributorCompSpeed |
| 发放奖励内核 | grantCompInternal |
| 更新 contributor 奖励累计 | updateContributorRewards |

***

## 🟣 5. 其它管理功能

| 功能 | 函数 |
| --- | --- |
| 是否是废弃市场 | isDeprecated |
| 获取所有市场 | getAllMarkets |
| block number 工具 | getBlockNumber |
| COMP token 地址 | getCompAddress |

***

# ⭐ 总结：你现在已经完整掌握 Comptroller 的“整个世界”

你现在已经把 Comptroller 这台“巨大引擎”的所有齿轮全部看过：

* 抵押品管理
* 转账限制
* 借贷检查
* 清算检查
* 激励系统（供应/借款/贡献者）
* 市场管理
* 特殊状态（deprecated）

这是 Compound V2 最复杂的一部分，你已经吃完了 100% 内容（真的很猛 🎉）。

# 1

# 📘 Compound V2 — COMP 奖励分发机制完整解析（Markdown 笔记）

本笔记总结 Comptroller 中负责 *COMP 挖矿/奖励分发* 的全部逻辑，包括：借款奖励、存款奖励、贡献者奖励、claim 流程以及参数管理。

***

# 1. COMP 分发设计总览

Compound 会根据区块数持续发放 COMP，发放对象包括：

* **供应者（Suppliers）**
* **借款者（Borrowers）**
* **贡献者（Contributor） — 特殊角色**

整个奖励分发机制依靠三个大的核心概念：

## 1.1 market index（市场指数）

每个市场都会维护两个指数：

* `compSupplyState[cToken].index`
* `compBorrowState[cToken].index`

它们记录「单位供应/单位借款累计奖励」。

## 1.2 user index（用户指数）

用户也有自己上次结算时的 index：

* `compSupplierIndex[cToken][user]`
* `compBorrowerIndex[cToken][user]`

## 1.3 compAccrued\[user]

记录用户的未领取 COMP。

公式核心：

```plain
用户新增奖励 = 持仓数量 × (市场当前指数 - 用户上次指数)
```

***

# 2. distributeBorrowerComp：分发借款者的 COMP

```plain
function distributeBorrowerComp(address cToken, address borrower, Exp memory marketBorrowIndex)
```

流程：

1. **读取市场当前 borrow 指数**
2. **读取 borrower 上次结算的 borrow 指数**
3. 如果 borrowerIndex=0（用户很早就借过），替换为 `compInitialIndex`
4. 计算本次新增指数 `deltaIndex`

```plain
deltaIndex = borrowIndex - borrowerIndex
```

5. 计算用户借款余额换算成 “借款单位”

```plain
borrowerAmount = borrowBalanceStored / marketBorrowIndex
```

6. 计算新增 COMP：

```plain
borrowerDelta = borrowerAmount × deltaIndex
```

7. 更新 `compAccrued[borrower]`

***

# 3. updateContributorRewards：贡献者奖励

```plain
function updateContributorRewards(address contributor)
```

机制：

* 贡献者具有自己的速度 `compContributorSpeeds`
* 每个区块获得固定的 COMP
* 根据区块差计算新增奖励：

```plain
newAccrued = deltaBlocks × compSpeed
```

* 写入 compAccrued\[contributor]

和借款/供应不同，不依赖 index，完全是固定速率发 COMP。

***

# 4. claimComp：领取奖励（多种模式）

Compound 提供四层调用：

### 4.1 单地址领取全部市场

```plain
claimComp(holder)
```

### 4.2 单地址领取指定市场

```plain
claimComp(holder, cTokens)
```

### 4.3 多地址领取（内部核心）

```plain
claimComp(holders, cTokens, borrowers, suppliers)
```

这个函数是实现核心：

在遍历每个市场时：

### 4.3.1 如果 borrowers=true：

1. 读取 borrowIndex
2. `updateCompBorrowIndex()` 更新市场指数
3. 给所有 holders 执行 distributeBorrowerComp

### 4.3.2 如果 suppliers=true：

1. `updateCompSupplyIndex()` 更新供应指数
2. 给所有 holders 执行 distributeSupplierComp

### 4.3.3 发放 COMP

最后统一执行：

```plain
compAccrued[holder] = grantCompInternal(holder, compAccrued[holder]);
```

***

# 5. grantCompInternal：实际转账

```plain
function grantCompInternal(address user, uint amount)
```

逻辑：

1. 如果 amount ≤ 本合约持有的 COMP，则执行 transfer
2. 否则全部失败，返回 amount（表示未发放）

这个函数用于 **真正把 COMP 转给用户**。

***

# 6. 管理员操作（Admin Only）

## 6.1 \_grantComp：管理员主动发 COMP

管理员可以向任意账户发 COMP，用于：

* 合作奖励
* 手工补偿
* 活动空投

## 6.2 \_setCompSpeeds：设置供应/借款挖矿速度

设置：

* supplySpeeds
* borrowSpeeds

这是最常见的 COMP 激励调参接口。

## 6.3 \_setContributorCompSpeed：设置贡献者速度

如果 speed=0，清除 lastContributorBlock。

***

# 7. getAllMarkets / isDeprecated

辅助函数：

## 7.1 getAllMarkets()

返回全市场列表。

## 7.2 isDeprecated(cToken)

如果满足：

* collateralFactor = 0
* borrowGuardianPaused = true
* reserveFactor = 1e18

则市场被标记为 deprecated，可被**立即清算**。

***

# 8. getBlockNumber / getCompAddress

简单返回：

* 当前区块号
* COMP token 地址

***

# 📌 总结（超简）

| 功能 | 关键函数 |
| --- | --- |
| 供应者奖励 | updateCompSupplyIndex / distributeSupplierComp |
| 借款者奖励 | updateCompBorrowIndex / distributeBorrowerComp |
| 贡献者奖励 | updateContributorRewards |
| 用户领取 | claimComp / grantCompInternal |
| 设置奖励速度 | \_setCompSpeeds / \_setContributorCompSpeed |
| 管理员手工发COMP | \_grantComp |

整个奖励系统核心就是「指数差 × 用户持仓量」。

# 1

# Compound V2 — COMP 分发逐行注释（包含源码片段）

> 说明：本文件包含 Comptroller 中与 COMP 分发（奖励）相关的关键函数源码片段，并对每一行或每一小段进行了逐行注释，便于深度学习与审计。注释紧跟源码行后，使用 `//` 风格以贴近 Solidity 注释习惯。

***

## 1. distributeBorrowerComp

```solidity
function distributeBorrowerComp(address cToken, address borrower, Exp memory marketBorrowIndex) internal {
    // 入口：给指定市场的借款者计算并记录应得的 COMP 奖励（不立即转账）
    CompMarketState storage borrowState = compBorrowState[cToken];
    // 读取该 cToken 对应的全局借款奖励状态（包含 index 和 block）
    uint256 borrowIndex = borrowState.index;
    // 把当前市场的 borrow 索引（奖励累计索引）赋值给局部变量
    uint256 borrowerIndex = compBorrowerIndex[cToken][borrower];
    // 读取该借款人在该市场上次结算时的索引（如果没有则为 0）

    // Update borrowers's index to the current index since we are distributing accrued COMP
    compBorrowerIndex[cToken][borrower] = borrowIndex;
    // 将借款人的索引更新为最新的全局 borrowIndex（表示已结算到当前为止）

    if (borrowerIndex == 0 && borrowIndex >= compInitialIndex) {
        // 如果用户之前没有索引记录（borrowerIndex == 0），但市场索引已经被初始化过
        borrowerIndex = compInitialIndex;
        // 则把用户的起始索引视作 compInitialIndex，防止忽略已发生的奖励段
    }

    // Calculate change in the cumulative sum of the COMP per borrowed unit accrued
    Double memory deltaIndex = Double({mantissa: sub_(borrowIndex, borrowerIndex)});
    // 计算两次索引之差 deltaIndex，代表用户自上次结算以来每“借款单位”应获得的 COMP 量

    uint256 borrowerAmount = div_(CToken(cToken).borrowBalanceStored(borrower), marketBorrowIndex);
    // 将借款者的借款余额转成“奖励计量单位”
    // borrowBalanceStored 返回的是 underlying 借款量
    // marketBorrowIndex 是传入的 cToken.borrowIndex()，这里做除法是为与 borrowIndex 的计量单位对齐

    // Calculate COMP accrued: cTokenAmount * accruedPerBorrowedUnit
    uint256 borrowerDelta = mul_(borrowerAmount, deltaIndex);
    // borrowerDelta = borrowerAmount * deltaIndex -> 本次应得 COMP

    uint256 borrowerAccrued = add_(compAccrued[borrower], borrowerDelta);
    // 将本次应得分配累加到 compAccrued（记录尚未领取的 COMP）

    compAccrued[borrower] = borrowerAccrued;
    // 写回存储，更新用户未领取的 COMP 数量

    emit DistributedBorrowerComp(CToken(cToken), borrower, borrowerDelta, borrowIndex);
    // 事件记录：市场、用户、本次新增 COMP、市场最新借款索引
}
2. updateContributorRewards
solidity
复制代码
function updateContributorRewards(address contributor) public {
    // 对于特定的“贡献者”按固定速度发 COMP（按区块计数）
    uint256 compSpeed = compContributorSpeeds[contributor];
    // 读取 contributor 的速度（每区块多少 COMP）
    uint256 blockNumber = getBlockNumber();
    // 当前区块号（可 override，便于测试）
    uint256 deltaBlocks = sub_(blockNumber, lastContributorBlock[contributor]);
    // 距离上次结算过去了多少区块
    if (deltaBlocks > 0 && compSpeed > 0) {
        // 只有在有区块差且速度大于 0 的情况下才计算奖励
        uint256 newAccrued = mul_(deltaBlocks, compSpeed);
        // 新增奖励 = 区块差 * 速度
        uint256 contributorAccrued = add_(compAccrued[contributor], newAccrued);
        // 累加到之前的未领取奖励

        compAccrued[contributor] = contributorAccrued;
        // 写回 compAccrued
        lastContributorBlock[contributor] = blockNumber;
        // 更新最后结算区块为当前块
    }
}
3. claimComp（各种重载）
3.1 单地址领取所有市场（包装函数）
solidity
复制代码
function claimComp(address holder) public {
    return claimComp(holder, allMarkets);
}
只是把单地址请求转为调用 claimComp(holder, allMarkets)，方便使用。

3.2 单地址领取指定市场（包装）
solidity
复制代码
function claimComp(address holder, CToken[] memory cTokens) public {
    address;
    holders[0] = holder;
    claimComp(holders, cTokens, true, true);
}
将单个 holder 包装成 holders 数组，默认同时领取 borrow 与 supply 的奖励。

3.3 核心：批量领取（可选择借款/供应奖励）
solidity
复制代码
function claimComp(address[] memory holders, CToken[] memory cTokens, bool borrowers, bool suppliers) public {
    for (uint256 i = 0; i < cTokens.length; i++) {
        CToken cToken = cTokens[i];
        require(markets[address(cToken)].isListed, "market must be listed");
        // 要保证市场已上架才允许领取

        if (borrowers == true) {
            Exp memory borrowIndex = Exp({mantissa: cToken.borrowIndex()});
            updateCompBorrowIndex(address(cToken));
            // 先更新该市场的 borrow 索引（会根据 borrowSpeed 与区块差调整）
            for (uint256 j = 0; j < holders.length; j++) {
                distributeBorrowerComp(address(cToken), holders[j], borrowIndex);
            }
            // 然后为每个 holder 调用分发函数，计算并累加 compAccrued
        }
        if (suppliers == true) {
            updateCompSupplyIndex(address(cToken));
            // 更新供应方索引
            for (uint256 j = 0; j < holders.length; j++) {
                distributeSupplierComp(address(cToken), holders[j]);
            }
            // 为每个 holder 累计供应端奖励（distributeSupplierComp 内部做类似的 index 差值计算）
        }
    }
    for (uint256 j = 0; j < holders.length; j++) {
        compAccrued[holders[j]] = grantCompInternal(holders[j], compAccrued[holders[j]]);
    }
    // 最后将 compAccrued 中的奖励尝试通过 grantCompInternal 转账给用户（如果余额足够）
}
4. grantCompInternal（实际转账尝试）
solidity
复制代码
function grantCompInternal(address user, uint256 amount) internal returns (uint256) {
    Comp comp = Comp(getCompAddress());
    // 获取 COMP token 合约实例（getCompAddress 可 override）
    uint256 compRemaining = comp.balanceOf(address(this));
    // 查询 Comptroller 合约自身持有的 COMP 余额
    if (amount > 0 && amount <= compRemaining) {
        comp.transfer(user, amount);
        // 如果余额充足，则转账给 user
        return 0;
        // 返回 0，表示没有未发放的金额
    }
    return amount;
    // 如果余额不足或 amount==0，则返回未发放的金额（通常调用端会把返回值写回 compAccrued）
}
5. 管理员直接发放与参数设置（片段）
_grantComp（管理员发放）
solidity
复制代码
function _grantComp(address recipient, uint256 amount) public {
    require(adminOrInitializing(), "only admin can grant comp");
    uint256 amountLeft = grantCompInternal(recipient, amount);
    require(amountLeft == 0, "insufficient comp for grant");
    emit CompGranted(recipient, amount);
}
管理员接口，直接转账 COMP，若合约内余额不足则 revert。

_setCompSpeeds（设置市场奖励速度）
solidity
复制代码
function _setCompSpeeds(CToken[] memory cTokens, uint256[] memory supplySpeeds, uint256[] memory borrowSpeeds) public {
    require(adminOrInitializing(), "only admin can set comp speed");
    uint256 numTokens = cTokens.length;
    require(numTokens == supplySpeeds.length && numTokens == borrowSpeeds.length, "Comptroller::_setCompSpeeds invalid input");
    for (uint256 i = 0; i < numTokens; ++i) {
        setCompSpeedInternal(cTokens[i], supplySpeeds[i], borrowSpeeds[i]);
    }
}
批量设置每个市场的供应/借款速度，实际写入逻辑在 setCompSpeedInternal。

_setContributorCompSpeed（设置贡献者速度）
solidity
复制代码
function _setContributorCompSpeed(address contributor, uint256 compSpeed) public {
    require(adminOrInitializing(), "only admin can set comp speed");
    updateContributorRewards(contributor);
    if (compSpeed == 0) {
        delete lastContributorBlock[contributor];
    } else {
        lastContributorBlock[contributor] = getBlockNumber();
    }
    compContributorSpeeds[contributor] = compSpeed;
    emit ContributorCompSpeedUpdated(contributor, compSpeed);
}
管理员设置 contributor 的速度，并结算到当前区块。

6. 其它辅助函数
getAllMarkets
solidity
复制代码
function getAllMarkets() public view returns (CToken[] memory) {
    return allMarkets;
}
isDeprecated
solidity
复制代码
function isDeprecated(CToken cToken) public view returns (bool) {
    return markets[address(cToken)].collateralFactorMantissa == 0 && borrowGuardianPaused[address(cToken)] == true
        && cToken.reserveFactorMantissa() == 1e18;
}
用于判断市场是否彻底废弃（不能作为抵押，借款暂停，储备因子为 100%）。

getBlockNumber / getCompAddress
solidity
复制代码
function getBlockNumber() public view virtual returns (uint256) {
    return block.number;
}
function getCompAddress() public view virtual returns (address) {
    return 0xc00e94Cb662C3520282E6f5717214004A7f26888;
}
getBlockNumber 可被测试合约重写，getCompAddress 返回主网 COMP 地址（可 override）。
```


> 更新: 2025-11-17 21:36:24  
> 原文: <https://www.yuque.com/xiaoyuhushenfu/yzin4n/hy7icnlv77al5g5e>