<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="https://s2.coinmarketcap.com/static/img/coins/64x64/7336.png" /></td>
        <td>
            <h1>Index Coop Report</h1>
            <h2>The Index Coop builds decentralized structured products</h2>
            <p>Prepared by: 0xVolodya, Independent Security Researcher</p>
            <p>Date: May 19 to Jun 14, 2023</p>
        </td>
    </tr>
</table>

# About Index Coop
The Index Coop building secure, accessible and simple to use DeFi products that make it easy for everyone from retail investors to institutions to gain exposure to the most important themes in DeFi.

# Summary of Findings

| &nbsp;&nbsp;&nbsp;ID&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Title                        | Severity | Fixed |
|----------------------------------------------------| ---------------------------- |----------| ----- |
| [H-01]                                             | AaveLeverageStrategyExtension doesn't work with turned on Efficiency Mode | High     |  ✓ |
| [M-01]                                             | Side effects of LTV = 0 assets: Index's users will not be able to withdraw (collateral), borrow | Medium   |  ✓ |
| [M-02]                                             |Some modules will not work with certain ERC20s reverting when trying to approve with allowance already >0| Medium   |  ✓ |
| [M-03]                                             |Deprecated chainlink oracle >0| Medium   |  ✓ |

# Detailed Findings

## [H-01] AaveLeverageStrategyExtension doesn't work with turned on Efficiency Mode

## Summary
AaveLeverageStrategyExtension doesn't work with turned-on Efficiency Mode. Incorrect

- LTV (Loan to value)
- Liquidation threshold
- Liquidation bonus
- A custom price oracle (optional)

## Vulnerability Detail
According to aave [docs](https://docs.aave.com/developers/whats-new/efficiency-mode-emode) whenever a pool in eMode it does how its own

- LTV (Loan to value)
- Liquidation threshold
- Liquidation bonus
- A custom price oracle (optional)

Whenever pool in eMode these params are not being fetched but instead using the same params as if eMode is not on which leads to the system having unexpected issues.

You can see from Morpho protocol that they are using different ltv and other params if pool in eMode
```solidity
 function _assetLiquidityData(address underlying, Types.LiquidityVars memory vars)
        internal
        view
        returns (uint256 underlyingPrice, uint256 ltv, uint256 liquidationThreshold, uint256 underlyingUnit)
    {
        DataTypes.ReserveConfigurationMap memory config = _pool.getConfiguration(underlying);

        bool isInEMode;
        (isInEMode, underlyingPrice, underlyingUnit) =
            _assetData(underlying, vars.oracle, config, vars.eModeCategory.priceSource);

        // If the LTV is 0 on Aave V3, the asset cannot be used as collateral to borrow upon a breaking withdraw.
        // In response, Morpho disables the asset as collateral and sets its liquidation threshold
        // to 0 and the governance should warn users to repay their debt.
        if (config.getLtv() == 0) return (underlyingPrice, 0, 0, underlyingUnit);

        if (isInEMode) {
            ltv = vars.eModeCategory.ltv;
            liquidationThreshold = vars.eModeCategory.liquidationThreshold;
        } else {
            ltv = config.getLtv();
            liquidationThreshold = config.getLiquidationThreshold();
        }
    }
```
[src/MorphoInternal.sol#L322](https://github.com/morpho-org/morpho-aave-v3/blob/0f494b8321d20789692e50305532b7f1b8fb23ef/src/MorphoInternal.sol#L322)

However, in Index eMode is not being used to determine ltv, liquidationThreshold and other params
```solidity
    function _calculateMaxBorrowCollateral(ActionInfo memory _actionInfo, bool _isLever) internal view returns(uint256) {
        
        // Retrieve collateral factor and liquidation threshold for the collateral asset in precise units (1e16 = 1%)
        ( , uint256 maxLtvRaw, uint256 liquidationThresholdRaw, , , , , , ,) = strategy.aaveProtocolDataProvider.getReserveConfigurationData(address(strategy.collateralAsset));

        // Normalize LTV and liquidation threshold to precise units. LTV is measured in 4 decimals in Aave which is why we must multiply by 1e14
        // for example ETH has an LTV value of 8000 which represents 80%
        if (_isLever) {
            uint256 netBorrowLimit = _actionInfo.collateralValue
                .preciseMul(maxLtvRaw.mul(10 ** 14))
                .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

            return netBorrowLimit
                .sub(_actionInfo.borrowValue)
                .preciseDiv(_actionInfo.collateralPrice);
        } else {
            uint256 netRepayLimit = _actionInfo.collateralValue
                .preciseMul(liquidationThresholdRaw.mul(10 ** 14))
                .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

            return _actionInfo.collateralBalance
                .preciseMul(netRepayLimit.sub(_actionInfo.borrowValue))
                .preciseDiv(netRepayLimit);
        }
    }

```
[adapters/AaveLeverageStrategyExtension.sol#L1095](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L1095)
## Impact
System will not work properly in eMode
## Code Snippet

## Tool used

Manual Review

## Recommendation
First, add global eModCategoryId variable which will be 0 by default
```diff
    function setEModeCategory(uint8 _categoryId) external onlyOperator {
+        eModCategoryId = _categoryId;
        _setEModeCategory(_categoryId);
    }
```
fetch data from eMode if its not 0, also do the same for oracle, like Morho is doing
```diff
    function _calculateMaxBorrowCollateral(ActionInfo memory _actionInfo, bool _isLever) internal view returns(uint256) {
        
        // Retrieve collateral factor and liquidation threshold for the collateral asset in precise units (1e16 = 1%)
        ( , uint256 maxLtvRaw, uint256 liquidationThresholdRaw, , , , , , ,) = strategy.aaveProtocolDataProvider.getReserveConfigurationData(address(strategy.collateralAsset));

+        if (eModCategoryId > 0) {
+            eModeCategory = _pool.getEModeCategoryData(eModCategoryId);
+            maxLtvRaw = eModeCategory.ltv;
+            liquidationThreshold = eModeCategory.liquidationThreshold;
+        }

        // Normalize LTV and liquidation threshold to precise units. LTV is measured in 4 decimals in Aave which is why we must multiply by 1e14
        // for example ETH has an LTV value of 8000 which represents 80%
        if (_isLever) {
            uint256 netBorrowLimit = _actionInfo.collateralValue
                .preciseMul(maxLtvRaw.mul(10 ** 14))
                .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

            return netBorrowLimit
                .sub(_actionInfo.borrowValue)
                .preciseDiv(_actionInfo.collateralPrice);
        } else {
            uint256 netRepayLimit = _actionInfo.collateralValue
                .preciseMul(liquidationThresholdRaw.mul(10 ** 14))
                .preciseMul(PreciseUnitMath.preciseUnit().sub(execution.unutilizedLeveragePercentage));

            return _actionInfo.collateralBalance
                .preciseMul(netRepayLimit.sub(_actionInfo.borrowValue))
                .preciseDiv(netRepayLimit);
        }
    }

```

## [M-01]  Side effects of LTV = 0 assets: Index's users will not be able to withdraw (collateral), borrow

## Summary
[Link to report from spearbit](https://solodit.xyz/issues/16216)
When an AToken has LTV = 0, Aave restricts the usage of some operations. In particular, if the user
owns at least one AToken as collateral that has LTV = 0, operations could revert.
1) Withdraw: if the asset withdrawn is collateral, the user is borrowing something, the operation will revert if the
   withdrawn collateral is an AToken with LTV > 0.
2) Transfer: if the from is using the asset as collateral, is borrowing something and the asset transferred is an
   AToken with LTV > 0 the operation will revert.
3) Set the reserve of an AToken as not collateral: if the AToken you are trying to set as non-collateral is an
   AToken with LTV > 0 the operation will revert.
   Note that all those checks are done on top of the "normal" checks that would usually prevent an operation, de-
   pending on the operation itself
## Vulnerability Detail
```solidity
    function _validateNewCollateralAsset(ISetToken _setToken, IERC20 _asset) internal view {
        require(!collateralAssetEnabled[_setToken][_asset], "Collateral already enabled");

        (address aToken, , ) = protocolDataProvider.getReserveTokensAddresses(address(_asset));
        require(address(underlyingToReserveTokens[_asset].aToken) == aToken, "Invalid aToken address");

        ( , , , , , bool usageAsCollateralEnabled, , , bool isActive, bool isFrozen) = protocolDataProvider.getReserveConfigurationData(address(_asset));
        // An active reserve is an alias for a valid reserve on Aave.
        // We are checking for the availability of the reserve directly on Aave rather than checking our internal `underlyingToReserveTokens` mappings,
        // because our mappings can be out-of-date if a new reserve is added to Aave
        require(isActive, "IAR");
        // A frozen reserve doesn't allow any new deposit, borrow or rate swap but allows repayments, liquidations and withdrawals
        require(!isFrozen, "FAR");
        require(usageAsCollateralEnabled, "CNE");
    }

```
[v1/AaveV3LeverageModule.sol#L1101](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AaveV3LeverageModule.sol#L1101)
## Impact
The Index protocol might stop working for users who would use those particular markets.
## Code Snippet

## Tool used

Manual Review

## Recommendation
Add restriction to those markets and add documentation to restrict those markets
E.x.

```diff
    function _validateNewCollateralAsset(ISetToken _setToken, IERC20 _asset) internal view {
        require(!collateralAssetEnabled[_setToken][_asset], "Collateral already enabled");

        (address aToken, , ) = protocolDataProvider.getReserveTokensAddresses(address(_asset));
        require(address(underlyingToReserveTokens[_asset].aToken) == aToken, "Invalid aToken address");

-        ( , , , , , bool usageAsCollateralEnabled, , , bool isActive, bool isFrozen) = protocolDataProvider.getReserveConfigurationData(address(_asset));
+        ( , uint256 ltv , , , , bool usageAsCollateralEnabled, , , bool isActive, bool isFrozen) = protocolDataProvider.getReserveConfigurationData(address(_asset));

        // An active reserve is an alias for a valid reserve on Aave.
        // We are checking for the availability of the reserve directly on Aave rather than checking our internal `underlyingToReserveTokens` mappings,
        // because our mappings can be out-of-date if a new reserve is added to Aave
        require(isActive, "IAR");
+      require(ltv != 0, "ltv should be non zero");

        // A frozen reserve doesn't allow any new deposit, borrow or rate swap but allows repayments, liquidations and withdrawals
        require(!isFrozen, "FAR");
        require(usageAsCollateralEnabled, "CNE");
    }

```

## [M-02] Some modules will not work with certain ERC20s reverting when trying to approve with allowance already >0

## Summary
Some ERC20 tokens like USDT require resetting the approval to 0 first before being able to reset it to another value.
## Vulnerability Detail
There are multiple files where using invokeApprove without resetting approve to 0.
There was 1 module where it was [fixed before](https://github.com/ckoopmann/set-protocol-v2/issues/3)
>Proposed Mitigation: Resetting allowance to 0 after every mint / redeem. This should also fix the issue of certain ERC20s reverting when trying to approve with allowance already >0

Here are files.
```solidity
    function _executeComponentApprovals(ActionInfo memory _actionInfo) internal {
        address spender = _actionInfo.ammAdapter.getSpenderAddress(_actionInfo.liquidityToken);

        // Loop through and approve total notional tokens to spender
        for (uint256 i = 0; i < _actionInfo.components.length ; i++) {
            _actionInfo.setToken.invokeApprove(
                _actionInfo.components[i],
                spender,
                _actionInfo.totalNotionalComponents[i]
            );
        }
    }

```
[v1/AmmModule.sol#L422](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AmmModule.sol#L422)

Other files:
contracts/protocol/modules/v1/WrapModuleV2.sol
contracts/protocol/modules/v1/StakingModule.sol
contracts/protocol/modules/v1/TradeModule.sol
contracts/protocol/modules/v1/AaveV3LeverageModule.sol
## Impact
Some modules will not work with certain ERC20s and there might be residual allowance left just like in this [issue](https://github.com/ckoopmann/set-protocol-v2/issues/3)
## Code Snippet

## Tool used

Manual Review

## Recommendation
Reset to 0 in those modules so certain ERC20 will work without problems

## [M-03] Deprecated chainlink oracle

## Summary
Deprecated chainlink oracle leads to incorrect and stale prices

## Vulnerability Detail
According to chainlink [docs](https://docs.chain.link/data-feeds/api-reference#latestanswer) latestAnswer shouldn't be used
> latestAnswer | (Deprecated - Do not use this function.)

latestAnswer is being used in the code and there is no check for stale prices
```solidity
    function _createActionInfo() internal view returns(ActionInfo memory) {
        ActionInfo memory rebalanceInfo;

        // Calculate prices from chainlink. Chainlink returns prices with 8 decimal places, but we need 36 - underlyingDecimals decimal places.
        // This is so that when the underlying amount is multiplied by the received price, the collateral valuation is normalized to 36 decimals.
        // To perform this adjustment, we multiply by 10^(36 - 8 - underlyingDecimals)
//      @audit latestAnswer	(Deprecated - Do not use this function.)
//      https://docs.chain.link/getting-started/consuming-data-feeds
        int256 rawCollateralPrice = strategy.collateralPriceOracle.latestAnswer();
        rebalanceInfo.collateralPrice = rawCollateralPrice.toUint256().mul(10 ** strategy.collateralDecimalAdjustment);
        int256 rawBorrowPrice = strategy.borrowPriceOracle.latestAnswer();
        rebalanceInfo.borrowPrice = rawBorrowPrice.toUint256().mul(10 ** strategy.borrowDecimalAdjustment);

        rebalanceInfo.collateralBalance = strategy.targetCollateralAToken.balanceOf(address(strategy.setToken));
        rebalanceInfo.borrowBalance = strategy.targetBorrowDebtToken.balanceOf(address(strategy.setToken));
        rebalanceInfo.collateralValue = rebalanceInfo.collateralPrice.preciseMul(rebalanceInfo.collateralBalance);
        rebalanceInfo.borrowValue = rebalanceInfo.borrowPrice.preciseMul(rebalanceInfo.borrowBalance);
        rebalanceInfo.setTotalSupply = strategy.setToken.totalSupply();

        return rebalanceInfo;
    }
```
[adapters/AaveLeverageStrategyExtension.sol#L889](https://github.com/sherlock-audit/2023-05-Index/blob/main/index-coop-smart-contracts/contracts/adapters/AaveLeverageStrategyExtension.sol#L895)
## Impact

## Code Snippet

## Tool used

Manual Review

## Recommendation
Use latestRoundData and check for stale prices

```solidity
            (uint80 roundId, int256 assetChainlinkPriceInt, , uint256 updatedAt, uint80 answeredInRound) = strategy.collateralPriceOracle.latestRoundData();
            require(answeredInRound >= roundId, "price is stale");
            require(updatedAt > 0, "round is incomplete");
            int256 rawCollateralPrice = assetChainlinkPriceInt;
```