<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="https://audits.sherlock.xyz/_next/image?url=https%3A%2F%2Fsherlock-files.ams3.digitaloceanspaces.com%2Fcontests%2Ftokemak.jpg&w=96&q=75" width="250" height="250" /></td>
        <td>
            <h1>Tokemak Audit Report</h1>
            <h2>Generating sustainable liquidity for the tokenized world. Eliminating inefficiencies and helping LPs to deploy liquidity where it can do the most work is exactly why we are building Tokemak v2!
</h2>
            <p>Prepared by: 0xVolodya, Independent Security Researcher</p>
            <p>Date: Aug 1 to Aug 7, 2023</p>
        </td>
    </tr>
</table>

# About Venus
Generating sustainable liquidity for the tokenized world. Eliminating inefficiencies and helping LPs to deploy liquidity where it can do the most work is exactly why we are building Tokemak v2!

# Summary of Findings

| &nbsp;&nbsp;&nbsp;ID&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Title                                                                                  | Severity | Fixed |
|----------------------------------------------------------------|----------------------------------------------------------------------------------------|----------|------|
| [H-01]                                                         | Liquidations sometimes will not work due to incorrect logic inside queueNewRewards | High     | ✓    |
| [H-02]                                                         | Incentive Pricing will not provide a robust estimate of incentive pricing to the LMP due to incorrect scaling | High     | ✓    |
| [H-03]                                                         | Curve pool reentrancy check doesn't work for some pools which lead to draining of funds | High     | ✓    |

# Detailed Findings

## [H-01]  Liquidations sometimes will not work due to incorrect logic inside queueNewRewards

## Vulnerability Detail
Whenever system conducts the liquidation process at some point there will be a call to queueNewRewards, lets note that approve is to amount
```solidity
            // approve main rewarder to pull the tokens
            LibAdapter._approve(IERC20(params.buyTokenAddress), address(mainRewarder), amount);
            mainRewarder.queueNewRewards(amount);
```
[src/liquidation/LiquidationRow.sol#L276](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/liquidation/LiquidationRow.sol#L276)

Let's look at queueNewRewards at some point startingQueuedRewards will not be 0 thus
newRewards = newRewards + startingQueuedRewards
and rewarded will try to request those funds from liquidationRow, but we remember that allowance is only newRewards and not newRewards + startingQueuedRewards, so this will fail
`IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards)`

```solidity

  function queueNewRewards(uint256 newRewards) external onlyWhitelisted {
        uint256 startingQueuedRewards = queuedRewards;
        uint256 startingNewRewards = newRewards;

        newRewards += startingQueuedRewards;

        if (block.number >= periodInBlockFinish) {
            notifyRewardAmount(newRewards);
            queuedRewards = 0;
        } else {
            uint256 elapsedBlock = block.number - (periodInBlockFinish - durationInBlock);
            uint256 currentAtNow = rewardRate * elapsedBlock;
            uint256 queuedRatio = currentAtNow * 1000 / newRewards;

            if (queuedRatio < newRewardRatio) {
                notifyRewardAmount(newRewards);
                queuedRewards = 0;
            } else {
                queuedRewards = newRewards;
            }
        }

        emit QueuedRewardsUpdated(startingQueuedRewards, startingNewRewards, queuedRewards);

        // Transfer the new rewards from the caller to this contract.
        IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards); // @audit should be newRewards - startingQueuedRewards
    }
```
[src/rewarders/AbstractRewarder.sol#L239](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/rewarders/AbstractRewarder.sol#L239)

## Recommendation
There suppose to be a subtraction because we already transferred startingQueuedRewards in previous calls

```solidity

    function queueNewRewards(uint256 newRewards) external onlyWhitelisted {
        uint256 startingQueuedRewards = queuedRewards;
        uint256 startingNewRewards = newRewards;

        newRewards += startingQueuedRewards;

        if (block.number >= periodInBlockFinish) {
            notifyRewardAmount(newRewards);
            queuedRewards = 0;
        } else {
            uint256 elapsedBlock = block.number - (periodInBlockFinish - durationInBlock);
            uint256 currentAtNow = rewardRate * elapsedBlock;
            uint256 queuedRatio = currentAtNow * 1000 / newRewards;

            if (queuedRatio < newRewardRatio) {
                notifyRewardAmount(newRewards);
                queuedRewards = 0;
            } else {
                queuedRewards = newRewards;
            }
        }

        emit QueuedRewardsUpdated(startingQueuedRewards, startingNewRewards, queuedRewards);

        // Transfer the new rewards from the caller to this contract.
-        IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards);
+        IERC20(rewardToken).safeTransferFrom(msg.sender, address(this), newRewards - startingQueuedRewards);
  }
```

## [H-02]  Incentive Pricing will not provide a robust estimate of incentive pricing to the LMP due to incorrect scaling

## Vulnerability Detail
After the initialization phase first INIT_SAMPLE_COUNT prices will be summed and scaled * 1e18 while getting the average.
But when the next time we will get existing.slowFilterPrice and existing.fastFilterPrice the current price will not be scaled

```solidity

        uint256 price = pricer.getPriceInEth(token);
        existing.lastSnapshot = uint40(block.timestamp);

        if (existing._initComplete) {
            existing.slowFilterPrice = Stats.getFilteredValue(SLOW_ALPHA, existing.slowFilterPrice, price); // price is not scaled by 1e18
            existing.fastFilterPrice = Stats.getFilteredValue(FAST_ALPHA, existing.fastFilterPrice, price);// price is not scaled by 1e18
        } else {
            // still the initialization phase
            existing._initCount += 1;
            existing._initAcc += price;

            if (existing._initCount == INIT_SAMPLE_COUNT) {
                existing._initComplete = true;
                uint256 averagePrice = existing._initAcc * 1e18 / INIT_SAMPLE_COUNT;// price scaled by 1e18
                existing.fastFilterPrice = averagePrice;
                existing.slowFilterPrice = averagePrice;
            }
        }
```
[calculators/IncentivePricingStats.sol#L156](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/stats/calculators/IncentivePricingStats.sol#L156)

This is how priorValue and currentValue look like in the test inside getFilteredValue function, which seems like scaling is not correct

```solidity
    function getFilteredValue(
        uint256 alpha,
        uint256 priorValue,
        uint256 currentValue
    ) internal view returns (uint256) {
        if (alpha > 1e18 || alpha == 0) revert Errors.InvalidParam("alpha");
        console.log("--------------------------------");
        console.log(priorValue);
        console.log(currentValue);
        return ((priorValue * (1e18 - alpha)) + (currentValue * alpha)) / 1e18;
    }
```

```solidity
  --------------------------------
  4288888888888888888888888888888888888
  10000000000
```

## [H-03]  Curve pool reentrancy check doesn't work for some pools which lead to draining of funds

## Vulnerability Detail
From there [post mortem](https://medium.com/@ConicFinance/post-mortem-eth-and-crvusd-omnipool-exploits-c9c7fa213a3d)

> Our assumption was that Curve v2 pools using ETH have the ETH address (0xeee…eee) as one of their coins. However, they instead have the WETH address. This led to _isETH returning false, and in turn, to the reentrancy guard of the rETH pool being bypassed.

E.x. in the readme there is

> Curve stETH/ETH concentrated: 0x828b154032950C8ff7CF8085D841723Db2696056

if you will go to [contract](https://etherscan.io/address/0x828b154032950c8ff7cf8085d841723db2696056#readContract) and see coins
it will use 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2 instead of 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE

Same for this 0x6c38cE8984a890F5e46e6dF6117C26b3F1EcfC9C and maybe some others.

```solidity
            if (iToken == LibAdapter.CURVE_REGISTRY_ETH_ADDRESS_POINTER) {
                if (poolInfo.checkReentrancy == 1) {
                    // This will fail in reentrancy
                    ICurveOwner(pool.owner()).withdraw_admin_fees(address(pool));
                }
            }
```
[CurveV1StableEthOracle.sol#L132](https://github.com/sherlock-audit/2023-06-tokemak/blob/main/v2-core-audit-2023-07-14/src/oracles/providers/CurveV1StableEthOracle.sol#L132)