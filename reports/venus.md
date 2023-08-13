<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="https://s2.coinmarketcap.com/static/img/coins/200x200/7288.png" width="250" height="250" /></td>
        <td>
            <h1>Venus Protocol Audit Report</h1>
            <h2>Algorithmic money market</h2>
            <p>Prepared by: 0xVolodya, Independent Security Researcher</p>
            <p>Date: May 9 to May 16, 2023</p>
        </td>
    </tr>
</table>

# About Venus
Venus is a decentralized finance (DeFi) algorithmic money market protocol on BNB Chain.

Decentralized lending pools are very similar to traditional lending services offered by banks, except that they are offered by P2P decentralized platforms. Users can leverage assets by borrowing and lending assets listed in a pool. Lending pools help crypto holders earn a substantial income through interest paid on their supplied assets and access assets they don't currently own without selling any of their portfolio.

# Summary of Findings
Not yet available

| &nbsp;&nbsp;&nbsp;ID&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Title                                                                                  | Severity | Fixed |
|----------------------------------|----------------------------------------------------------------------------------------|----------|------|
| [H-01]                           | blocksPerYear is not sync like it supposed to in bsc chain | High     | ✓    |
| [M-01]                           | Users can borrow the borrowCap amount, but they should borrow less than that and not equal                                      | Medium   | ✓    |
| [M-02]                           | Sometimes calculateBorrowerReward and calculateSupplierReward return incorrect results                                      | Medium   | ✓    |
| [M-03]                           | First Deposit Bug                                      | Medium   | ✓    |
| [M-04]                           | Inconsistent scaling of USD in bad debt in the project.                                      | Medium   | ✓    |
| [M-05]                           | There are no incentives for users to start bidding after the auction restarts.                                      | Medium   | x    |
| [M-06]                           | There are no restriction on auction duration                                      | Medium   | x    |
| [M-07]                           | The Oracle returns incorrect prices because it does not call updatePrice before calling getUnderlyingPrice                                      | Medium   |   ✓   |
| [M-08]                           | Repayments Paused While Liquidations Enabled                                      | Medium   |   ✓   |

# Detailed Findings

## [H-01]  blocksPerYear is not sync like it supposed to in bsc chain
## Impact
Detailed description of the impact of this finding.
The variable calculations in WhitePaperInterestRateModel are incorrect compared to BaseJumpRateModelV2 due to a lack of synchronization in blocksPerYear.
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
blocksPerYear should be 10512000 just like in WhitePaperInterestRateModel for bsc chain
```solidity
/**
 * @notice The approximate number of blocks per year that is assumed by the interest rate model
 */
uint256 public constant blocksPerYear = 2102400;
```
[contracts/WhitePaperInterestRateModel.sol#L17](https://github.com/VenusProtocol/isolated-pools/blob/f075e8256a5215d438ff610f34cd3e25eea7c79d/contracts/WhitePaperInterestRateModel.sol#L17)


```solidity
/**
 * @notice The approximate number of blocks per year that is assumed by the interest rate model
 */
uint256 public constant blocksPerYear = 10512000;
```
[contracts/BaseJumpRateModelV2.sol#L23](https://github.com/VenusProtocol/isolated-pools/blob/f075e8256a5215d438ff610f34cd3e25eea7c79d/contracts/BaseJumpRateModelV2.sol#L23)

There will be invalid `baseRatePerBlock` and `multiplierPerBlock`
```solidity
constructor(uint256 baseRatePerYear, uint256 multiplierPerYear) {
    baseRatePerBlock = baseRatePerYear / blocksPerYear;
    multiplierPerBlock = multiplierPerYear / blocksPerYear;

    emit NewInterestParams(baseRatePerBlock, multiplierPerBlock);
}

```
[contracts/WhitePaperInterestRateModel.sol#L36](https://github.com/VenusProtocol/isolated-pools/blob/f075e8256a5215d438ff610f34cd3e25eea7c79d/contracts/WhitePaperInterestRateModel.sol#L36)
## Tools Used
Manual
## Recommended Mitigation Steps
Change to 10512000

## [M-01]  Users can borrow the borrowCap amount, but they should borrow less than that and not equal
## Impact
Detailed description of the impact of this finding.
Users can borrow up to the borrowCap amount, but they should borrow less than that.
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
According to the docs from the code function `Borrowing that brings total borrows to borrow cap will revert`
```solidity
   /**
    * @notice Set the given borrow caps for the given vToken markets. Borrowing that brings total borrows to or above borrow cap will revert.
...
```
[contracts/Comptroller.sol#L839](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Comptroller.sol#L839)

this means that whevener user should not be able to hit borrowCap and revert if `nextTotalBorrows == borrowCap` but it doesn't
```solidity
       if (borrowCap != type(uint256).max) {
           uint256 totalBorrows = VToken(vToken).totalBorrows();
           uint256 nextTotalBorrows = totalBorrows + borrowAmount;
           //          @audit should be, users can borrow more than allowed
           //            if (nextTotalBorrows >= borrowCap) {
           if (nextTotalBorrows > borrowCap) {
               revert BorrowCapExceeded(vToken, borrowCap);
           }
       }
```
[contracts/Comptroller.sol#L354](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Comptroller.sol#L354)
## Tools Used

## Recommended Mitigation Steps

```diff
       if (borrowCap != type(uint256).max) {
           uint256 totalBorrows = VToken(vToken).totalBorrows();
           uint256 nextTotalBorrows = totalBorrows + borrowAmount;
-            if (nextTotalBorrows > borrowCap) {
+            if (nextTotalBorrows >= borrowCap) {
               revert BorrowCapExceeded(vToken, borrowCap);
           }
       }
```

## [M-02]  Sometimes calculateBorrowerReward and calculateSupplierReward return incorrect results

## Impact
Detailed description of the impact of this finding.
Sometimes calculateBorrowerReward and calculateSupplierReward return incorrect results
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Whenever user wants to know his pending rewards he calls `getPendingRewards` sometimes it returns incorrect results.

There is a bug inside `calculateBorrowerReward` and `calculateSupplierReward`
```solidity
function calculateBorrowerReward(
    address vToken,
    RewardsDistributor rewardsDistributor,
    address borrower,
    RewardTokenState memory borrowState,
    Exp memory marketBorrowIndex
) internal view returns (uint256) {
    Double memory borrowIndex = Double({ mantissa: borrowState.index });
    Double memory borrowerIndex = Double({
        mantissa: rewardsDistributor.rewardTokenBorrowerIndex(vToken, borrower)
    });
//      @audit
//        if (borrowerIndex.mantissa == 0 && borrowIndex.mantissa >= rewardsDistributor.rewardTokenInitialIndex()) {
    if (borrowerIndex.mantissa == 0 && borrowIndex.mantissa > 0) {
        // Covers the case where users borrowed tokens before the market's borrow state index was set
        borrowerIndex.mantissa = rewardsDistributor.rewardTokenInitialIndex();
    }
    Double memory deltaIndex = sub_(borrowIndex, borrowerIndex);
    uint256 borrowerAmount = div_(VToken(vToken).borrowBalanceStored(borrower), marketBorrowIndex);
    uint256 borrowerDelta = mul_(borrowerAmount, deltaIndex);
    return borrowerDelta;
}

```
[contracts/Lens/PoolLens.sol#L495](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Lens/PoolLens.sol#L495)

```solidity
function calculateSupplierReward(
    address vToken,
    RewardsDistributor rewardsDistributor,
    address supplier,
    RewardTokenState memory supplyState
) internal view returns (uint256) {
    Double memory supplyIndex = Double({ mantissa: supplyState.index });
    Double memory supplierIndex = Double({
        mantissa: rewardsDistributor.rewardTokenSupplierIndex(vToken, supplier)
    });
//      @audit
//        if (supplierIndex.mantissa == 0 && supplyIndex.mantissa  >= rewardsDistributor.rewardTokenInitialIndex()) {
    if (supplierIndex.mantissa == 0 && supplyIndex.mantissa > 0) {
        // Covers the case where users supplied tokens before the market's supply state index was set
        supplierIndex.mantissa = rewardsDistributor.rewardTokenInitialIndex();
    }
    Double memory deltaIndex = sub_(supplyIndex, supplierIndex);
    uint256 supplierTokens = VToken(vToken).balanceOf(supplier);
    uint256 supplierDelta = mul_(supplierTokens, deltaIndex);
    return supplierDelta;
}

```
[contracts/Lens/PoolLens.sol#L516](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Lens/PoolLens.sol#L516)

Inside rewardsDistributor original functions written likes this
```solidity
function _distributeSupplierRewardToken(address vToken, address supplier) internal {
...
    if (supplierIndex == 0 && supplyIndex >= rewardTokenInitialIndex) {
        // Covers the case where users supplied tokens before the market's supply state index was set.
        // Rewards the user with REWARD TOKEN accrued from the start of when supplier rewards were first
        // set for the market.
        supplierIndex = rewardTokenInitialIndex;
    }
...
}
```
[contracts/Rewards/RewardsDistributor.sol#L340](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Rewards/RewardsDistributor.sol#L340)

```solidity
function _distributeBorrowerRewardToken(
    address vToken,
    address borrower,
    Exp memory marketBorrowIndex
) internal {
...
    if (borrowerIndex == 0 && borrowIndex >= rewardTokenInitialIndex) {
        // Covers the case where users borrowed tokens before the market's borrow state index was set.
        // Rewards the user with REWARD TOKEN accrued from the start of when borrower rewards were first
        // set for the market.
        borrowerIndex = rewardTokenInitialIndex;
    }
...
}
```
[Rewards/RewardsDistributor.sol#L374](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Rewards/RewardsDistributor.sol#L374)
## Tools Used

## Recommended Mitigation Steps

```diff
function calculateSupplierReward(
    address vToken,
    RewardsDistributor rewardsDistributor,
    address supplier,
    RewardTokenState memory supplyState
) internal view returns (uint256) {
    Double memory supplyIndex = Double({ mantissa: supplyState.index });
    Double memory supplierIndex = Double({
        mantissa: rewardsDistributor.rewardTokenSupplierIndex(vToken, supplier)
    });
-        if (supplierIndex.mantissa == 0 && supplyIndex.mantissa > 0) {
+        if (supplierIndex.mantissa == 0 && supplyIndex.mantissa  >= rewardsDistributor.rewardTokenInitialIndex()) {
        // Covers the case where users supplied tokens before the market's supply state index was set
        supplierIndex.mantissa = rewardsDistributor.rewardTokenInitialIndex();
    }
    Double memory deltaIndex = sub_(supplyIndex, supplierIndex);
    uint256 supplierTokens = VToken(vToken).balanceOf(supplier);
    uint256 supplierDelta = mul_(supplierTokens, deltaIndex);
    return supplierDelta;
}
```
```diff
function calculateBorrowerReward(
    address vToken,
    RewardsDistributor rewardsDistributor,
    address borrower,
    RewardTokenState memory borrowState,
    Exp memory marketBorrowIndex
) internal view returns (uint256) {
    Double memory borrowIndex = Double({ mantissa: borrowState.index });
    Double memory borrowerIndex = Double({
        mantissa: rewardsDistributor.rewardTokenBorrowerIndex(vToken, borrower)
    });
-        if (borrowerIndex.mantissa == 0 && borrowIndex.mantissa > 0) {
+        if (borrowerIndex.mantissa == 0 && borrowIndex.mantissa >= rewardsDistributor.rewardTokenInitialIndex()) {
        // Covers the case where users borrowed tokens before the market's borrow state index was set
        borrowerIndex.mantissa = rewardsDistributor.rewardTokenInitialIndex();
    }
    Double memory deltaIndex = sub_(borrowIndex, borrowerIndex);
    uint256 borrowerAmount = div_(VToken(vToken).borrowBalanceStored(borrower), marketBorrowIndex);
    uint256 borrowerDelta = mul_(borrowerAmount, deltaIndex);
    return borrowerDelta;
}
```

## [M-03]  First Deposit Bug

## Vulnerability details

The CToken is a yield bearing asset which is minted when any user deposits some units of
`underlying` tokens. The amount of CTokens minted to a user is calculated based upon
the amount of `underlying` tokens user is depositing.

As per the implementation of CToken contract, there exist two cases for CToken amount calculation:

1. First deposit - when `VToken.totalSupply()` is `0`.
2. All subsequent deposits.

Here is the actual CToken code (extra code and comments clipped for better reading):

```solidity
function _exchangeRateStored() internal view virtual returns (uint256) {
    uint256 _totalSupply = totalSupply;
    if (_totalSupply == 0) {
        /*
         * If there are no tokens minted:
         *  exchangeRate = initialExchangeRate
         */
        return initialExchangeRateMantissa;
    } else {
        /*
         * Otherwise:
         *  exchangeRate = (totalCash + totalBorrows + badDebt - totalReserves) / totalSupply
         */
        uint256 totalCash = _getCashPrior();
        uint256 cashPlusBorrowsMinusReserves = totalCash + totalBorrows + badDebt - totalReserves;
        uint256 exchangeRate = (cashPlusBorrowsMinusReserves * expScale) / _totalSupply;

        return exchangeRate;
    }
}

function _mintFresh(
    address payer,
    address minter,
    uint256 mintAmount
) internal {
    /* Fail if mint not allowed */
    comptroller.preMintHook(address(this), minter, mintAmount);

    /* Verify market's block number equals current block number */
    if (accrualBlockNumber != _getBlockNumber()) {
        revert MintFreshnessCheck();
    }

    Exp memory exchangeRate = Exp({ mantissa: _exchangeRateStored() });
...
```
[/contracts/VToken.sol#L1463](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/VToken.sol#L1463)

## Impact
A sophisticated attack can impact all user deposits until the lending protocols owners and users are notified and contracts are paused. Since this attack is a replicable attack it can be performed continuously to steal the deposits of all depositors that try to deposit into the CToken contract.

The loss amount will be the sum of all deposits done by users into the CToken multiplied by the underlying token's price.

Suppose there are `10` users and each of them tries to deposit `1,000,000` underlying tokens into the CToken contract. Price of underlying token is `$1`.

`Total loss (in $) = $10,000,000`


## The Fix
The fix to prevent this issue would be to enforce a minimum deposit that cannot be withdrawn. This can be done by minting small amount of CToken units to `0x00` address on the first deposit.

Instead of a fixed `1000` value an admin controlled parameterized value can also be used to control the burn amount on a per CToken basis.

## [M-04]  Inconsistent scaling of USD in bad debt in the project.

## Impact
Detailed description of the impact of this finding.
Inconsistent scaling of USD in bad debt in the project.

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
usdvalue is being calculated differently in project. This is how its calculated for auction, As you can see its being scaled down by  `/ 1e18`. Its the only place in project where `getUnderlyingPrice * tokenAmount` is being scaled down.
```solidity
  for (uint256 i; i < marketsCount; ++i) {
      uint256 marketBadDebt = vTokens[i].badDebt();

      priceOracle.updatePrice(address(vTokens[i]));
      uint256 usdValue = (priceOracle.getUnderlyingPrice(address(vTokens[i])) * marketBadDebt) / 1e18;

      poolBadDebt = poolBadDebt + usdValue;
      auction.markets[i] = vTokens[i];
      auction.marketDebt[vTokens[i]] = marketBadDebt;
      marketsDebt[i] = marketBadDebt;
  }

```
[contracts/Shortfall/Shortfall.sol#L393](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Shortfall/Shortfall.sol#L393)

This is how badDebtUsd is being calculated inside poollens
```solidity
  for (uint256 i; i < markets.length; ++i) {
      BadDebt memory badDebt;
      badDebt.vTokenAddress = address(markets[i]);
      badDebt.badDebtUsd =
          VToken(address(markets[i])).badDebt() *
          priceOracle.getUnderlyingPrice(address(markets[i]));
      badDebtSummary.badDebts[i] = badDebt;
      totalBadDebtUsd = totalBadDebtUsd + badDebt.badDebtUsd;
  }
```
[contracts/Lens/PoolLens.sol#L268](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Lens/PoolLens.sol#L268)

## Tools Used
Manual
## Recommended Mitigation Steps
As I understood, scaling down should be removed inside shortfall due the only place in the project where `getUnderlyingPrice * tokenAmount` is being scaled down

```diff
  for (uint256 i; i < marketsCount; ++i) {
      uint256 marketBadDebt = vTokens[i].badDebt();

      priceOracle.updatePrice(address(vTokens[i]));
-            uint256 usdValue = (priceOracle.getUnderlyingPrice(address(vTokens[i])) * marketBadDebt) / 1e18;
+            uint256 usdValue = (priceOracle.getUnderlyingPrice(address(vTokens[i])) * marketBadDebt);

      poolBadDebt = poolBadDebt + usdValue;
      auction.markets[i] = vTokens[i];
      auction.marketDebt[vTokens[i]] = marketBadDebt;
      marketsDebt[i] = marketBadDebt;
  }

```

## [M-05]  There are no incentives for users to start bidding after the auction restarts.


## Impact
Detailed description of the impact of this finding.
There are no incentives for users to start bidding after the auction restarts this means that auction might restarts forever.
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
If the auction becomes stale without receiving any bids, anyone can restart the auction by calling restartAuction. If nobody is placing bids this means that its not profitable for users, so protocol should make some changes to auction to incentive users to start placing bids. E.x. [makerdao](https://docs.makerdao.com/smart-contract-modules/system-stabilizer-module/flop-detailed-documentation) increasing lot(funds) by 50% on the restart.
> If the auction expires without receiving any bids, anyone can restart the auction by calling tick(uint auction_id). This will do two things:
1. It resets bids[id].end to now + tau
2. It resets bids[id].lot to bids[id].lot * pad / ONE

## Tools Used

## Recommended Mitigation Steps
Make it appealing for users to start bidding after the auction becomes stale. For example, you can consider calling `swapPoolsAssets` to increase the `riskFundBalance` inside the `_startAuction` function


## [M-06]  There are no restriction on auction duration

## Impact
Detailed description of the impact of this finding.
There are no restriction on auction duration, if nextBidderBlockLimit will be changed then auction might never ends
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
There are no restriction on auction duration, current length of auction is `nextBidderBlockLimit * MAX_BPS = 10*10000`  10*10000 / 28753 ~ 3.5 days. That's the maximum amount of time a user can prolong the auction, its depens on `auction.highestBidBlock + nextBidderBlockLimit` inside `closeAuction`

```solidity
function closeAuction(address comptroller) external nonReentrant {
    ...
    require(
        block.number > auction.highestBidBlock + nextBidderBlockLimit && auction.highestBidder != address(0),
        "waiting for next bidder. cannot close auction"
    );
...
```
[contracts/Shortfall/Shortfall.sol#L214](https://github.com/VenusProtocol/isolated-pools/blob/f075e8256a5215d438ff610f34cd3e25eea7c79d/contracts/Shortfall/Shortfall.sol#L214)

According to the code nextBidderBlockLimit can be changed, e.x. admin would like to change it to the same as [makerdao](https://docs.makerdao.com/keepers/the-auctions-of-the-maker-protocol) to 6 hours ~ 7188 blocks

> ttl: Bid duration (for example, 6 hours). The auction ends if no new bid is placed during this time.

```sodliity
function updateNextBidderBlockLimit(uint256 _nextBidderBlockLimit) external {
    _checkAccessAllowed("updateNextBidderBlockLimit(uint256)");
    require(_nextBidderBlockLimit != 0, "_nextBidderBlockLimit must not be 0");
    uint256 oldNextBidderBlockLimit = nextBidderBlockLimit;
    nextBidderBlockLimit = _nextBidderBlockLimit;
    emit NextBidderBlockLimitUpdated(oldNextBidderBlockLimit, _nextBidderBlockLimit);
}
```
[contracts/Shortfall/Shortfall.sol#L293](https://github.com/VenusProtocol/isolated-pools/blob/f075e8256a5215d438ff610f34cd3e25eea7c79d/contracts/Shortfall/Shortfall.sol#L293)
That would mean that max lenght of auction would be `nextBidderBlockLimit * MAX_BPS = 7188*10000` 7188 *10000 / 28753 ~  2499 days
Which is too much, without any way to restrict an auction duration.

## Tools Used

## Recommended Mitigation Steps
Introduce auction time limit variable so the system will be for flexible for the admin without never ending auction just like [makerDao](https://docs.makerdao.com/keepers/the-auctions-of-the-maker-protocol) has `tau`

> tau: Auction duration (for example, 24 hours). The auction ends after this period under all circumstances.

## [M-07]  The Oracle returns incorrect prices because it does not call updatePrice before calling getUnderlyingPrice

## Impact
Detailed description of the impact of this finding.
The Oracle returns incorrect prices because it does not call updatePrice before calling getUnderlyingPrice
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

According to the [comments](https://github.com/VenusProtocol/oracle/blob/develop/contracts/ResilientOracle.sol#L175) from oracle file. updatePrice should always be called before calling getUnderlyingPrice.
```solidity
/**
 * @notice Updates the TWAP pivot oracle price.
 * @dev This function should always be called before calling getUnderlyingPrice
 * @param vToken vToken address
 */
function updatePrice(address vToken) external override {
    (address pivotOracle, bool pivotOracleEnabled) = getOracle(vToken, OracleRole.PIVOT);
    if (pivotOracle != address(0) && pivotOracleEnabled) {
        //if pivot oracle is not TwapOracle it will revert so we need to catch the revert
        try TwapInterface(pivotOracle).updateTwap(vToken) {} catch {}
    }
}
```
[/contracts/ResilientOracle.sol#L175](https://github.com/VenusProtocol/oracle/blob/develop/contracts/ResilientOracle.sol#L175)

There are functions in the code that do not call updatePrice before calling getUnderlyingPrice while some functions call updatePrice. E.x. this function is not calling updatePrice
calls _checkRedeemAllowed -> calls _getHypotheticalLiquiditySnapshot -> calls _safeGetUnderlyingPrice(asset)
```solidity
function exitMarket(address vTokenAddress) external override returns (uint256) {
    _checkActionPauseState(vTokenAddress, Action.EXIT_MARKET);
    VToken vToken = VToken(vTokenAddress);
    /* Get sender tokensHeld and amountOwed underlying from the vToken */
    (uint256 tokensHeld, uint256 amountOwed, ) = _safeGetAccountSnapshot(vToken, msg.sender);

    /* Fail if the sender has a borrow balance */
    if (amountOwed != 0) {
        revert NonzeroBorrowBalance();
    }

    /* Fail if the sender is not permitted to redeem all of their tokens */
    _checkRedeemAllowed(vTokenAddress, msg.sender, tokensHeld);

    Market storage marketToExit = markets[address(vToken)];

    /* Return true if the sender is not already ‘in’ the market */
    if (!marketToExit.accountMembership[msg.sender]) {
        return NO_ERROR;
    }

    /* Set vToken account membership to false */
    delete marketToExit.accountMembership[msg.sender];

    /* Delete vToken from the account’s list of assets */
    // load into memory for faster iteration
    VToken[] memory userAssetList = accountAssets[msg.sender];
    uint256 len = userAssetList.length;

    uint256 assetIndex = len;
    for (uint256 i; i < len; ++i) {
        if (userAssetList[i] == vToken) {
            assetIndex = i;
            break;
        }
    }

    // We *must* have found the asset in the list or our redundant data structure is broken
    assert(assetIndex < len);

    // copy last item in list to location of item to be removed, reduce length by 1
    VToken[] storage storedList = accountAssets[msg.sender];
    storedList[assetIndex] = storedList[storedList.length - 1];
    storedList.pop();

    emit MarketExited(vToken, msg.sender);

    return NO_ERROR;
}

```
[contracts/Comptroller.sol#L199](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Comptroller.sol#L199)

The same way you can track that these functions doesn't call updatePrice before calling getUnderlyingPrice
`liquidateCalculateSeizeTokens` calls _safeGetUnderlyingPrice -> getUnderlyingPrice
`getHypotheticalAccountLiquidity` calls _getHypotheticalLiquiditySnapshot -> _safeGetUnderlyingPrice -> getUnderlyingPrice
`setCollateralFactor`
inside Comptroller.sol

Inside PoolLens.sol
```solidity
function getPoolBadDebt(address comptrollerAddress) external view returns (BadDebtSummary memory) {
    uint256 totalBadDebtUsd;

    // Get every market in the pool
    ComptrollerViewInterface comptroller = ComptrollerViewInterface(comptrollerAddress);
    VToken[] memory markets = comptroller.getAllMarkets();
    PriceOracle priceOracle = comptroller.oracle();

    BadDebt[] memory badDebts = new BadDebt[](markets.length);

    BadDebtSummary memory badDebtSummary;
    badDebtSummary.comptroller = comptrollerAddress;
    badDebtSummary.badDebts = badDebts;

    // // Calculate the bad debt is USD per market
    for (uint256 i; i < markets.length; ++i) {
        BadDebt memory badDebt;
        badDebt.vTokenAddress = address(markets[i]);
        badDebt.badDebtUsd =
            VToken(address(markets[i])).badDebt() *
            priceOracle.getUnderlyingPrice(address(markets[i]));
        badDebtSummary.badDebts[i] = badDebt;
        totalBadDebtUsd = totalBadDebtUsd + badDebt.badDebtUsd;
    }

    badDebtSummary.totalBadDebtUsd = totalBadDebtUsd;

    return badDebtSummary;
}
```
[contracts/Lens/PoolLens.sol#L268](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Lens/PoolLens.sol#L268)

```solidity
function vTokenUnderlyingPrice(VToken vToken) public view returns (VTokenUnderlyingPrice memory) {
    ComptrollerViewInterface comptroller = ComptrollerViewInterface(address(vToken.comptroller()));
    PriceOracle priceOracle = comptroller.oracle();

    return
        VTokenUnderlyingPrice({
            vToken: address(vToken),
            underlyingPrice: priceOracle.getUnderlyingPrice(address(vToken))
        });
}
```
[contracts/Lens/PoolLens.sol#L408](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Lens/PoolLens.sol#L408)
## Tools Used

## Recommended Mitigation Steps
I think it will be cleaner to place update price inside _safeGetUnderlyingPrice instead of inside every function like its now. Code wil be cleaner as well.
Add updatePrice to other functions inside PoolLens as well

```diff
function _safeGetUnderlyingPrice(VToken asset) internal view returns (uint256) {
+       oracle.updatePrice(address(asset));
    uint256 oraclePriceMantissa = oracle.getUnderlyingPrice(address(asset));
    if (oraclePriceMantissa == 0) {
        revert PriceError(address(asset));
    }
    return oraclePriceMantissa;
}
```

## [M-08]  Repayments Paused While Liquidations Enabled

## Impact
Detailed description of the impact of this finding.
It is possible to have repayments Paused While Liquidations Enabled
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Lending & Borrowing DeFi platforms should never be able to enter a state where repayments are paused but liquidations are enabled, since this would unfairly prevent Borrowers from making their repayments while still allowing them to be liquidated. If repayments can be paused then liquidations must also be paused at the same time.

Right now if `_checkActionPauseState(vToken, Action.REPAY)` is true and `_checkActionPauseState(vTokenBorrowed, Action.LIQUIDATE);` is false than Liquidations is enable but repayment not.

```solidity
function preRepayHook(address vToken, address borrower) external override {
    _checkActionPauseState(vToken, Action.REPAY);

    oracle.updatePrice(vToken);

    if (!markets[vToken].isListed) {
        revert MarketNotListed(address(vToken));
    }

    // Keep the flywheel moving
    uint256 rewardDistributorsCount = rewardsDistributors.length;

    for (uint256 i; i < rewardDistributorsCount; ++i) {
        Exp memory borrowIndex = Exp({ mantissa: VToken(vToken).borrowIndex() });
        rewardsDistributors[i].updateRewardTokenBorrowIndex(vToken, borrowIndex);
        rewardsDistributors[i].distributeBorrowerRewardToken(vToken, borrower, borrowIndex);
    }
}

```
[contracts/Comptroller.sol#L390](https://github.com/code-423n4/2023-05-venus/blob/9853f6f4fe906b635e214b22de9f627c6a17ba5b/contracts/Comptroller.sol#L390)
## Tools Used

## Recommended Mitigation Steps

```diff
function preLiquidateHook(
    address vTokenBorrowed,
    address vTokenCollateral,
    address borrower,
    uint256 repayAmount,
    bool skipLiquidityCheck
) external override {
    // Pause Action.LIQUIDATE on BORROWED TOKEN to prevent liquidating it.
    // If we want to pause liquidating to vTokenCollateral, we should pause
    // Action.SEIZE on it
    _checkActionPauseState(vTokenBorrowed, Action.LIQUIDATE);
+        _checkActionPauseState(vTokenBorrowed, Action.REPAY);

    oracle.updatePrice(vTokenBorrowed);
    oracle.updatePrice(vTokenCollateral);

    if (!markets[vTokenBorrowed].isListed) {
        revert MarketNotListed(address(vTokenBorrowed));
    }
    if (!markets[vTokenCollateral].isListed) {
        revert MarketNotListed(address(vTokenCollateral));
    }

    uint256 borrowBalance = VToken(vTokenBorrowed).borrowBalanceStored(borrower);

    /* Allow accounts to be liquidated if the market is deprecated or it is a forced liquidation */
    if (skipLiquidityCheck || isDeprecated(VToken(vTokenBorrowed))) {
        if (repayAmount > borrowBalance) {
            revert TooMuchRepay();
        }
        return;
    }

    /* The borrower must have shortfall and collateral > threshold in order to be liquidatable */
    AccountLiquiditySnapshot memory snapshot = _getCurrentLiquiditySnapshot(borrower, _getLiquidationThreshold);

    if (snapshot.totalCollateral <= minLiquidatableCollateral) {
        /* The liquidator should use either liquidateAccount or healAccount */
        revert MinimalCollateralViolated(minLiquidatableCollateral, snapshot.totalCollateral);
    }

    if (snapshot.shortfall == 0) {
        revert InsufficientShortfall();
    }

    /* The liquidator may not repay more than what is allowed by the closeFactor */
    uint256 maxClose = mul_ScalarTruncate(Exp({ mantissa: closeFactorMantissa }), borrowBalance);
    if (repayAmount > maxClose) {
        revert TooMuchRepay();
    }
}

```