<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="https://pbs.twimg.com/profile_images/1683495143876313088/GoAlrKp2_400x400.png" width="250" height="250" /></td>
        <td>
            <h1>Moonwell Audit Report</h1>
            <h2>An open lending and borrowing DeFi protocol built on Base, Moonbeam, and Moonriver.</h2>
            <p>Prepared by: 0xVolodya, Independent Security Researcher</p>
            <p>Date: July 25 to Aug 1, 2023</p>
        </td>
    </tr>
</table>

# About Venus
An open lending and borrowing DeFi protocol built on Base, Moonbeam, and Moonriver.

# Summary of Findings

| &nbsp;&nbsp;&nbsp;ID&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Title                                                                                  | Severity | Fixed |
|----------------------------------------------------------------|----------------------------------------------------------------------------------------|----------|------|
| [M-01]                                                         | One check is missing when proposal execution bypassing queuing | Medium   | ✓    |
| [M-02]                                                         | Its not possible to liquidate deprecated market | Medium   | ✓    |
| [M-03]                                                         | Unsafe use of transfer()/transferFrom() with IERC20 | Medium   | ✓    |
| [M-04]                                                         | Insufficient oracle validation | Medium   | ✓    |
| [M-05]                                                         | Missing checks for whether the L2 Sequencer is active | Medium   | ✓    |
| [M-06]                                                         | The `owner` is a single point of failure and a centralization risk | Medium   | ✓    |
| [M-07]                                                         | Some tokens may revert when zero value transfers are made | Medium   | ✓    |

# Detailed Findings

## [M-01]  One check is missing when proposal execution bypassing queuing
## Impact
Detailed description of the impact of this finding.
One check is missing when proposal execution bypassing queuing
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
In `TemporalGovernor`, `executeProposal` can be executed with a flag bypassing queuing, so there are the same checks as inside `queueProposal`, but there is one check that missing, and according to comments is important:
```solidity
  // Very important to check to make sure that the VAA we're processing is specifically designed
  // to be sent to this contract
  require(intendedRecipient == address(this), "TemporalGovernor: Incorrect destination");
```
[Governance/TemporalGovernor.sol#L311](https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Governance/TemporalGovernor.sol#L311)
so, the same check should be added to `executeProposal`
## Tools Used

## Recommended Mitigation Steps

```diff
 address[] memory targets; /// contracts to call
  uint256[] memory values; /// native token amount to send
  bytes[] memory calldatas; /// calldata to send
-        (, targets, values, calldatas) = abi.decode(
+        (intendedRecipient, targets, values, calldatas) = abi.decode(
      vm.payload,
      (address, address[], uint256[], bytes[])
  );

+        require(intendedRecipient == address(this), "TemporalGovernor: Incorrect destination");

  /// Interaction (s)

  _sanityCheckPayload(targets, values, calldatas);

```

## [M-02] Its not possible to liquidate deprecated market
## Impact
Detailed description of the impact of this finding.
Its not possible to liquidate deprecated market
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Currently in the code that is a function `_setBorrowPaused` that paused pause borrowing. In [origin compound code](https://github.com/compound-finance/compound-protocol/blob/a3214f67b73310d547e00fc578e8355911c9d376/contracts/Comptroller.sol#L1452) `borrowGuardianPaused` is used to do liquidate markets that are bad. So now there is not way to get rid of bad markets.
```solidity
    function liquidateBorrowAllowed(
        address mTokenBorrowed,
        address mTokenCollateral,
        address liquidator,
        address borrower,
        uint repayAmount) override external view returns (uint) {
        // Shh - currently unused
        liquidator;

        if (!markets[mTokenBorrowed].isListed || !markets[mTokenCollateral].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }

        /* The borrower must have shortfall in order to be liquidatable */
        (Error err, , uint shortfall) = getAccountLiquidityInternal(borrower);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }
        if (shortfall == 0) {
            return uint(Error.INSUFFICIENT_SHORTFALL);
        }

        /* The liquidator may not repay more than what is allowed by the closeFactor */
        uint borrowBalance = MToken(mTokenBorrowed).borrowBalanceStored(borrower);
        uint maxClose = mul_ScalarTruncate(Exp({mantissa: closeFactorMantissa}), borrowBalance);
        if (repayAmount > maxClose) {
            return uint(Error.TOO_MUCH_REPAY);
        }

        return uint(Error.NO_ERROR);
    }

```
[src/core/Comptroller.sol#L394](https://github.com/code-423n4/2023-07-moonwell/blob/fced18035107a345c31c9a9497d0da09105df4df/src/core/Comptroller.sol#L394)
## Tools Used

## Recommended Mitigation Steps
I think compound have a way to liquidate deprecated markets for a safety reason, so it needs to be restored
```diff
    function liquidateBorrowAllowed(
        address mTokenBorrowed,
        address mTokenCollateral,
        address liquidator,
        address borrower,
        uint repayAmount) override external view returns (uint) {
        // Shh - currently unused
        liquidator;
        /* allow accounts to be liquidated if the market is deprecated */
+        if (isDeprecated(CToken(cTokenBorrowed))) {
+            require(borrowBalance >= repayAmount, "Can not repay more than the total borrow");
+            return uint(Error.NO_ERROR);
+        }
        if (!markets[mTokenBorrowed].isListed || !markets[mTokenCollateral].isListed) {
            return uint(Error.MARKET_NOT_LISTED);
        }

        /* The borrower must have shortfall in order to be liquidatable */
        (Error err, , uint shortfall) = getAccountLiquidityInternal(borrower);
        if (err != Error.NO_ERROR) {
            return uint(err);
        }
        if (shortfall == 0) {
            return uint(Error.INSUFFICIENT_SHORTFALL);
        }

        /* The liquidator may not repay more than what is allowed by the closeFactor */
        uint borrowBalance = MToken(mTokenBorrowed).borrowBalanceStored(borrower);
        uint maxClose = mul_ScalarTruncate(Exp({mantissa: closeFactorMantissa}), borrowBalance);
        if (repayAmount > maxClose) {
            return uint(Error.TOO_MUCH_REPAY);
        }

        return uint(Error.NO_ERROR);
    }
+    function isDeprecated(CToken cToken) public view returns (bool) {
+        return
+        markets[address(cToken)].collateralFactorMantissa == 0 &&
+        borrowGuardianPaused[address(cToken)] == true &&
+        cToken.reserveFactorMantissa() == 1e18
+        ;
+    }

```

## [M-03]  Unsafe use of transfer()/transferFrom() with IERC20
Some tokens do not implement the ERC20 standard properly but are still accepted by most code that accepts ERC20 tokens.  For example Tether (USDT)'s `transfer()` and `transferFrom()` functions on L1 do not return booleans as the specification requires, and instead have no return value. When these sorts of tokens are cast to `IERC20`, their [function signatures](https://medium.com/coinmonks/missing-return-value-bug-at-least-130-tokens-affected-d67bf08521ca) do not match and therefore the calls made, revert (see [this](https://gist.github.com/IllIllI000/2b00a32e8f0559e8f386ea4f1800abc5) link for a test case). Use OpenZeppelin’s `SafeERC20`'s `safeTransfer()`/`safeTransferFrom()` instead

*There are 2 instances of this issue:*

```solidity
File: src/core/Comptroller.sol

965:              token.transfer(admin, token.balanceOf(address(this)));

967:              token.transfer(admin, _amount);

```
https://github.com/code-423n4/2023-07-moonwell/blob/4aaa7d6767da3bc42e31c18ea2e75736a4ea53d4/src/core/Comptroller.sol#L965

## [M-04]  Insufficient oracle validation
There is no freshness check on the timestamp of the prices fetched from the Chainlink oracle, so old prices may be used if [OCR](https://docs.chain.link/architecture-overview/off-chain-reporting) was unable to push an update in time. Add a staleness threshold number of seconds configuration parameter, and ensure that the price fetched is from within that time range.

*There are 2 instances of this issue:*

```solidity
File: src/core/Oracles/ChainlinkCompositeOracle.sol

183          (
184              uint80 roundId,
185              int256 price,
186              ,
187              ,
188              uint80 answeredInRound
189:         ) = AggregatorV3Interface(oracleAddress).latestRoundData();

```
https://github.com/code-423n4/2023-07-moonwell/blob/4aaa7d6767da3bc42e31c18ea2e75736a4ea53d4/src/core/Oracles/ChainlinkCompositeOracle.sol#L183-L189

```solidity
File: src/core/Oracles/ChainlinkOracle.sol

100          (, int256 answer, , uint256 updatedAt, ) = AggregatorV3Interface(feed)
101:             .latestRoundData();

```
https://github.com/code-423n4/2023-07-moonwell/blob/4aaa7d6767da3bc42e31c18ea2e75736a4ea53d4/src/core/Oracles/ChainlinkOracle.sol#L100-L101



## [M-05]  Missing checks for whether the L2 Sequencer is active
Chainlink recommends that users using price oracles, check whether the Arbitrum Sequencer is [active](https://docs.chain.link/data-feeds/l2-sequencer-feeds#arbitrum). If the sequencer goes down, the Chainlink oracles will have stale prices from before the downtime, until a new L2 OCR transaction goes through. Users who submit their transactions via the [L1 Dealyed Inbox](https://developer.arbitrum.io/tx-lifecycle#1b--or-from-l1-via-the-delayed-inbox) will be able to take advantage of these stale prices. Use a [Chainlink oracle](https://blog.chain.link/how-to-use-chainlink-price-feeds-on-arbitrum/#almost_done!_meet_the_l2_sequencer_health_flag) to determine whether the sequencer is offline or not, and don't allow operations to take place while the sequencer is offline.

*There are 2 instances of this issue:*

```solidity
File: src/core/Oracles/ChainlinkCompositeOracle.sol

189:         ) = AggregatorV3Interface(oracleAddress).latestRoundData();

```
https://github.com/code-423n4/2023-07-moonwell/blob/4aaa7d6767da3bc42e31c18ea2e75736a4ea53d4/src/core/Oracles/ChainlinkCompositeOracle.sol#L189-L189

```solidity
File: src/core/Oracles/ChainlinkOracle.sol

100          (, int256 answer, , uint256 updatedAt, ) = AggregatorV3Interface(feed)
101:             .latestRoundData();

```
https://github.com/code-423n4/2023-07-moonwell/blob/4aaa7d6767da3bc42e31c18ea2e75736a4ea53d4/src/core/Oracles/ChainlinkOracle.sol#L100-L101


## [M-06]  The `owner` is a single point of failure and a centralization risk
Having a single EOA as the only owner of contracts is a large centralization risk and a single point of failure. A single private key may be taken in a hack, or the sole holder of the key may become unable to retrieve the key when necessary. Consider changing to a multi-signature setup, or having a role-based authorization model.

*There are 2 instances of this issue:*

```solidity
File: src/core/Governance/TemporalGovernor.sol

266:     function fastTrackProposalExecution(bytes memory VAA) external onlyOwner {

274:     function togglePause() external onlyOwner {

```
https://github.com/code-423n4/2023-07-moonwell/blob/4aaa7d6767da3bc42e31c18ea2e75736a4ea53d4/src/core/Governance/TemporalGovernor.sol#L266-L266


## [M-07]  Some tokens may revert when zero value transfers are made
In spite of the fact that EIP-20 [states](https://github.com/ethereum/EIPs/blob/46b9b698815abbfa628cd1097311deee77dd45c5/EIPS/eip-20.md?plain=1#L116) that zero-valued transfers must be accepted, some tokens, such as LEND will revert if this is attempted, which may cause transactions that involve other tokens (such as batch operations) to fully revert. Consider skipping the transfer if the amount is zero, which will also save gas.

*There are 2 instances of this issue:*

```solidity
File: src/core/Comptroller.sol

964          if (_amount == type(uint).max) {
965              token.transfer(admin, token.balanceOf(address(this)));
966          } else {
967:             token.transfer(admin, _amount);

962          IERC20 token = IERC20(_tokenAddress);
963          // Similar to mTokens, if this is uint.max that means "transfer everything"
964          if (_amount == type(uint).max) {
965:             token.transfer(admin, token.balanceOf(address(this)));

```
https://github.com/code-423n4/2023-07-moonwell/blob/4aaa7d6767da3bc42e31c18ea2e75736a4ea53d4/src/core/Comptroller.sol#L964-L967