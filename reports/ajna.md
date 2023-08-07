<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="https://repository-images.githubusercontent.com/633575987/9597b876-a4bb-434d-86b7-2e9ce368bd4c" /></td>
        <td>
            <h1>Ajna Audit Report</h1>
            <h2>A peer to peer, oracleless, permissionless lending protocol with no governance, accepting both fungible and non fungible tokens as collateral.</h2>
            <p>Prepared by: 0xVolodya, Independent Security Researcher</p>
            <p>Date: May 3 to May 11, 2023</p>
        </td>
    </tr>
</table>

# About Ajna
The Ajna protocol is a non-custodial, peer-to-peer, permissionless lending, borrowing and trading system that requires no governance or external price feeds to function. The protocol consists of pools: pairings of quote tokens provided by lenders and collateral tokens provided by borrowers. Ajna is capable of accepting fungible tokens as quote tokens and both fungible and non-fungible tokens as collateral tokens.# About **0xVolodya**

# Summary of Findings

| ID     | Title                                                                | Severity | Fixed |
|--------|----------------------------------------------------------------------|----------| ----- |
| [H-01] | User can avoid bankruptcy for his position inside PositionManager    | High     |  ✓ |
| [L-01] | Find the status of proposal sometimes returns incorrect status       | Low      |  ✓ |
| [L-02] | Use of transferFrom() rather than safeTransferFrom() for NFTs| Low      |  ✓ |
| [L-03] | _safeMint() should be used rather than _mint() wherever possible| Low      |  ✓ |
| [L-04] | Users can claim delegate reward at the time they are not supposed to | Medium   |  ✓ |

## [H-01]  User can avoid bankruptcy for his position inside PositionManager

## Impact
Detailed description of the impact of this finding.
User can avoid bankruptcy for his position inside PositionManager.
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Whenever user would like to redeem his position, there is a check that position is not bankrupt.
```solidity
function reedemPositions(
    RedeemPositionsParams calldata params_
) external override mayInteract(params_.pool, params_.tokenId) {
...
        if (_bucketBankruptAfterDeposit(pool, index, position.depositTime)) revert BucketBankrupt();
...
```
A user can revive their position (rather than going bankrupt) by creating a new position and transferring its liquidity to the desired position that they want to recover. As you can see that whenever position`s liquidity being moved to another position its position is being rewritten, instead of taking minimun from both of them

```solidity
function moveLiquidity(
    MoveLiquidityParams calldata params_
) external override mayInteract(params_.pool, params_.tokenId) nonReentrant {
...
    // update position LP state
    fromPosition.lps -= vars.lpbAmountFrom;
    toPosition.lps   += vars.lpbAmountTo;
    // update position deposit time to the from bucket deposit time
//      @audit
//        toPosition.depositTime = Math.max(vars.depositTime, toPosition.depositTime);
    toPosition.depositTime = vars.depositTime;

    emit MoveLiquidity(
        ownerOf(params_.tokenId),
        params_.tokenId,
        params_.fromIndex,
        params_.toIndex,
        vars.lpbAmountFrom,
        vars.lpbAmountTo
    );
}

```

## Tools Used

## Recommended Mitigation Steps
Assign minimum deposition time to interacted position
```diff
function moveLiquidity(
    MoveLiquidityParams calldata params_
) external override mayInteract(params_.pool, params_.tokenId) nonReentrant {
    Position storage fromPosition = positions[params_.tokenId][params_.fromIndex];

    MoveLiquidityLocalVars memory vars;
    vars.depositTime = fromPosition.depositTime;

    // handle the case where owner attempts to move liquidity after they've already done so
    if (vars.depositTime == 0) revert RemovePositionFailed();

    // ensure bucketDeposit accounts for accrued interest
    IPool(params_.pool).updateInterest();

    // retrieve info of bucket from which liquidity is moved
    (
        vars.bucketLP,
        vars.bucketCollateral,
        vars.bankruptcyTime,
        vars.bucketDeposit,
    ) = IPool(params_.pool).bucketInfo(params_.fromIndex);

    // check that bucket hasn't gone bankrupt since memorialization
    if (vars.depositTime <= vars.bankruptcyTime) revert BucketBankrupt();

    // calculate the max amount of quote tokens that can be moved, given the tracked LP
    vars.maxQuote = _lpToQuoteToken(
        vars.bucketLP,
        vars.bucketCollateral,
        vars.bucketDeposit,
        fromPosition.lps,
        vars.bucketDeposit,
        _priceAt(params_.fromIndex)
    );

    EnumerableSet.UintSet storage positionIndex = positionIndexes[params_.tokenId];

    // remove bucket index from which liquidity is moved from tracked positions
    if (!positionIndex.remove(params_.fromIndex)) revert RemovePositionFailed();

    // update bucket set at which a position has liquidity
    // slither-disable-next-line unused-return
    positionIndex.add(params_.toIndex);

    // move quote tokens in pool
    (
        vars.lpbAmountFrom,
        vars.lpbAmountTo,
    ) = IPool(params_.pool).moveQuoteToken(
        vars.maxQuote,
        params_.fromIndex,
        params_.toIndex,
        params_.expiry
    );

    Position storage toPosition = positions[params_.tokenId][params_.toIndex];

    // update position LP state
    fromPosition.lps -= vars.lpbAmountFrom;
    toPosition.lps   += vars.lpbAmountTo;
    // update position deposit time to the from bucket deposit time
+        if(toPosition.depositTime ==0){
+            toPosition.depositTime = vars.depositTime;
+        } else{
+            toPosition.depositTime = vars.depositTime < toPosition.depositTime ? vars.depositTime :toPosition.depositTime;
+        }
    emit MoveLiquidity(
        ownerOf(params_.tokenId),
        params_.tokenId,
        params_.fromIndex,
        params_.toIndex,
        vars.lpbAmountFrom,
        vars.lpbAmountTo
    );
}

```


## [L-01]  Find the status of a given proposal returns incorrect status for some standard proposals
## Impact
Detailed description of the impact of this finding.
Find the status of a given proposal returns incorrect status for some standard proposals
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Whevener user would like to know a state of standard proposal, he request `state` from GrantFund
```solidity
function state(
    uint256 proposalId_
) external view override returns (ProposalState) {
    FundingMechanism mechanism = findMechanismOfProposal(proposalId_);

    return mechanism == FundingMechanism.Standard ? _standardProposalState(proposalId_) : _getExtraordinaryProposalState(proposalId_);
}
```
[ajna-grants/src/grants/GrantFund.sol#L45](https://github.com/code-423n4/2023-05-ajna/blob/76c254c0085e7520edd24cd2f8b79cbb61d7706c/ajna-grants/src/grants/GrantFund.sol#L45)

which calls _standardProposalState where is a bug when ProposalState.Active
```solidity
function _standardProposalState(uint256 proposalId_) internal view returns (ProposalState) {
    Proposal memory proposal = _standardFundingProposals[proposalId_];

    if (proposal.executed)                                                     return ProposalState.Executed;
    else if (_distributions[proposal.distributionId].endBlock >= block.number) return ProposalState.Active;
    else if (_standardFundingVoteSucceeded(proposalId_))                      return ProposalState.Succeeded;
    else                                                                       return ProposalState.Defeated;
}

```
[src/grants/base/StandardFunding.sol#L509](https://github.com/code-423n4/2023-05-ajna/blob/76c254c0085e7520edd24cd2f8b79cbb61d7706c/ajna-grants/src/grants/base/StandardFunding.sol#L509)

if should be `block.number > screeningStageEndBlock && _distributions[proposal.distributionId].endBlock >= block.number` just like in `fundingVote`
```solidity
    function fundingVote(
    FundingVoteParams[] memory voteParams_
) external override returns (uint256 votesCast_) {
...
    uint256 screeningStageEndBlock = _getScreeningStageEndBlock(endBlock);

    // check that the funding stage is active
    if (block.number <= screeningStageEndBlock || block.number > endBlock) revert InvalidVote();
...

```
[/grants/base/StandardFunding.sol#L532](https://github.com/code-423n4/2023-05-ajna/blob/76c254c0085e7520edd24cd2f8b79cbb61d7706c/ajna-grants/src/grants/base/StandardFunding.sol#L532)
## Tools Used

## Recommended Mitigation Steps
Change to this
```diff
function _standardProposalState(uint256 proposalId_) internal view returns (ProposalState) {
    Proposal memory proposal = _standardFundingProposals[proposalId_];
+       uint24 currentDistributionId = _currentDistributionId;
+       uint256 endBlock = _distributions[currentDistributionId].endBlock;
+       uint256 screeningStageEndBlock = _getScreeningStageEndBlock(endBlock);

    if (proposal.executed)                                                     return ProposalState.Executed;
-        else if (_distributions[proposal.distributionId].endBlock >= block.number) return ProposalState.Active;
+        else if (block.number > screeningStageEndBlock && _distributions[proposal.distributionId].endBlock >= block.number) return ProposalState.Active;
    else if (_standardFundingVoteSucceeded(proposalId_))                      return ProposalState.Succeeded;
    else                                                                       return ProposalState.Defeated;
}

```

## [L-02]  Use of transferFrom() rather than safeTransferFrom() for NFTs in will lead to the loss of NFTs

The EIP-721 standard says the following about `transferFrom()`:
```solidity
    /// @notice Transfer ownership of an NFT -- THE CALLER IS RESPONSIBLE
    ///  TO CONFIRM THAT `_to` IS CAPABLE OF RECEIVING NFTS OR ELSE
    ///  THEY MAY BE PERMANENTLY LOST
    /// @dev Throws unless `msg.sender` is the current owner, an authorized
    ///  operator, or the approved address for this NFT. Throws if `_from` is
    ///  not the current owner. Throws if `_to` is the zero address. Throws if
    ///  `_tokenId` is not a valid NFT.
    /// @param _from The current owner of the NFT
    /// @param _to The new owner
    /// @param _tokenId The NFT to transfer
    function transferFrom(address _from, address _to, uint256 _tokenId) external payable;
```
https://github.com/ethereum/EIPs/blob/78e2c297611f5e92b6a5112819ab71f74041ff25/EIPS/eip-721.md?plain=1#L103-L113
Code must use the `safeTransferFrom()` flavor if it hasn't otherwise verified that the receiving address can handle it

*There is one instance of this issue:*

```solidity
File: ajna-core/src/RewardsManager.sol

302:          IERC721(address(positionManager)).transferFrom(address(this), msg.sender, tokenId_);

```
https://github.com/code-423n4/2023-05-ajna/blob/6995f24bdf9244fa35880dda21519ffc131c905c/ajna-core/src/RewardsManager.sol#L302

## [L-03]  _safeMint() should be used rather than _mint() wherever possible
`_mint()` is [discouraged](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d4d8d2ed9798cc3383912a23b5e8d5cb602f7d4b/contracts/token/ERC721/ERC721.sol#L271) in favor of `_safeMint()` which ensures that the recipient is either an EOA or implements `IERC721Receiver`. Both [OpenZeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/d4d8d2ed9798cc3383912a23b5e8d5cb602f7d4b/contracts/token/ERC721/ERC721.sol#L238-L250) and [solmate](https://github.com/Rari-Capital/solmate/blob/4eaf6b68202e36f67cab379768ac6be304c8ebde/src/tokens/ERC721.sol#L180) have versions of this function

*There is one instance of this issue:*

```solidity
File: ajna-core/src/PositionManager.sol

238:          _mint(params_.recipient, tokenId_);

```
https://github.com/code-423n4/2023-05-ajna/blob/6995f24bdf9244fa35880dda21519ffc131c905c/ajna-core/src/PositionManager.sol#L238


## [L-04]  Users can claim delegate reward at the time they are not supposed to
## Impact
Detailed description of the impact of this finding.
Users can claim delegate reward at the time they are not supposed to.
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

Whenever a user wants to claim their delegate reward, they call the `claimDelegateReward` function inside StandardFunding. The function includes a check to verify if the period is active or not.
```solidity
function claimDelegateReward(
    uint24 distributionId_
) external override returns(uint256 rewardClaimed_) {
    // Revert if delegatee didn't vote in screening stage
    if(screeningVotesCast[distributionId_][msg.sender] == 0) revert DelegateRewardInvalid();

    QuarterlyDistribution memory currentDistribution = _distributions[distributionId_];

    // Check if Challenge Period is still active
    if(block.number < _getChallengeStageEndBlock(currentDistribution.endBlock)) revert ChallengePeriodNotEnded();
...
```
[src/grants/base/StandardFunding.sol#L245](https://github.com/code-423n4/2023-05-ajna/blob/76c254c0085e7520edd24cd2f8b79cbb61d7706c/ajna-grants/src/grants/base/StandardFunding.sol#L245)
That check suppose to be `block.number <= _getChallengeStageEndBlock(currentDistribution.endBlock` becuase if we will look at how that check holds thoughout the file we will see it.

```
        if (currentDistributionId > 0 && (block.number > _getChallengeStageEndBlock(currentDistributionEndBlock))) {
            // Add unused funds from last distribution to treasury
            _updateTreasury(currentDistributionId);
        }
```
[ajna-grants/src/grants/base/StandardFunding.sol#L129](https://github.com/code-423n4/2023-05-ajna/blob/76c254c0085e7520edd24cd2f8b79cbb61d7706c/ajna-grants/src/grants/base/StandardFunding.sol#L129)
```
    if (block.number <= _getChallengeStageEndBlock(_distributions[distributionId].endBlock)) revert ExecuteProposalInvalid();
```
```
    if (block.number <= endBlock || block.number > _getChallengeStageEndBlock(endBlock)) {
        revert InvalidProposalSlate();
    }
```
## Tools Used

## Recommended Mitigation Steps
```diff
function claimDelegateReward(
    uint24 distributionId_
) external override returns(uint256 rewardClaimed_) {
    // Revert if delegatee didn't vote in screening stage
    if(screeningVotesCast[distributionId_][msg.sender] == 0) revert DelegateRewardInvalid();

    QuarterlyDistribution memory currentDistribution = _distributions[distributionId_];

    // Check if Challenge Period is still active
-        if(block.number < _getChallengeStageEndBlock(currentDistribution.endBlock)) revert ChallengePeriodNotEnded();
+        if(block.number <= _getChallengeStageEndBlock(currentDistribution.endBlock)) revert ChallengePeriodNotEnded();

    // check rewards haven't already been claimed
    if(hasClaimedReward[distributionId_][msg.sender]) revert RewardAlreadyClaimed();

    QuadraticVoter memory voter = _quadraticVoters[distributionId_][msg.sender];

    // calculate rewards earned for voting
    rewardClaimed_ = _getDelegateReward(currentDistribution, voter);

    hasClaimedReward[distributionId_][msg.sender] = true;

    emit DelegateRewardClaimed(
        msg.sender,
        distributionId_,
        rewardClaimed_
    );

    // transfer rewards to delegatee
    IERC20(ajnaTokenAddress).safeTransfer(msg.sender, rewardClaimed_);
}

```

