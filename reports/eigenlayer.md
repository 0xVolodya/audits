<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="https://pbs.twimg.com/profile_images/1635879999264940033/_pozth32_400x400.jpg" width="250" height="250" /></td>
        <td>
            <h1>EigenLayer Audit Report</h1>
            <h2>Enabling restaking of staked Ether</h2>
            <p>Prepared by: 0xVolodya, Independent Security Researcher</p>
            <p>Date: Apr 28 to May 5, 2023</p>
        </td>
    </tr>
</table>

# About EigenLayer
EigenLayer (formerly 'EigenLayr') is a set of smart contracts deployed on Ethereum that enable restaking of assets to secure new services.
At present, this repository contains both the contracts for EigenLayer and a set of general "middleware" contracts, designed to be reuseable across different applications built on top of EigenLayer.

# Summary & Scope

The [Layr-Labs/eigenlayer-contracts](https://github.com/Layr-Labs/eigenlayer-contracts) repository was audited at commit [7a23e259050fe88a179ab0345cc8cfc9b5e57221](https://github.com/Layr-Labs/eigenlayer-contracts/commit/7a23e259050fe88a179ab0345cc8cfc9b5e57221).

# Summary of Findings
Not yet available

| &nbsp;&nbsp;&nbsp;ID&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Title                                                                                  | Severity | Fixed |
|----------------------------------|----------------------------------------------------------------------------------------|----------| ----- |
| [H-01]                           | "verifyAndProcessWithdrawal" can be abused to steal from validator | High     |  ✓ |
| [H-02]                           | It is impossible to slash some queued withdrawals                                      | High     |  ✓ |
| [L-01]                           | computePhase0Eth1DataRoot returns an incorrect Merkle tree                             | Low     |  ✓ |
| [L-02]                           | processInclusionProofKeccak does not work as expected                                  | Low     |  ✓ |
| [L-03]                           | merkleizeSha256 doesn’t work as expected                                               | Low     |  ✓ |
| [L-04]                           | claimableUserDelayedWithdrawals bug                                                    | Low     |  ✓ |
| [L-05]                           | The condition for full withdrawals different fromthe documentation                     | Low     |  ✓ |
| [L-06]                           | Missing validation to a threshold value on full withdrawal                             | Low     |  ✓ |
| [L-07]                           | User can stake twice on beacon chain from same eipod                                   | Low     |  ✓ |

# Detailed Findings

## [H-01]  "verifyAndProcessWithdrawal" can be abused to steal from every validator at least once

## Impact
Detailed description of the impact of this finding.
"verifyAndProcessWithdrawal" can be abused to steal from every validator at least once.
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Whevener user call `verifyAndProcessWithdrawal` there is a verification that proofs are valid

```solidity
    function verifyAndProcessWithdrawal(
        BeaconChainProofs.WithdrawalProofs calldata withdrawalProofs,
        bytes calldata validatorFieldsProof,
        bytes32[] calldata validatorFields,
        bytes32[] calldata withdrawalFields,
        uint256 beaconChainETHStrategyIndex,
        uint64 oracleBlockNumber
    )
        external
        onlyWhenNotPaused(PAUSED_EIGENPODS_VERIFY_WITHDRAWAL)
        onlyNotFrozen
        /**
         * Check that the provided block number being proven against is after the `mostRecentWithdrawalBlockNumber`.
         * Without this check, there is an edge case where a user proves a past withdrawal for a validator whose funds they already withdrew,
         * as a way to "withdraw the same funds twice" without providing adequate proof.
         * Note that this check is not made using the oracleBlockNumber as in the `verifyWithdrawalCredentials` proof; instead this proof
         * proof is made for the block number of the withdrawal, which may be within 8192 slots of the oracleBlockNumber.
         * This difference in modifier usage is OK, since it is still not possible to `verifyAndProcessWithdrawal` against a slot that occurred
         * *prior* to the proof provided in the `verifyWithdrawalCredentials` function.
         */
        proofIsForValidBlockNumber(Endian.fromLittleEndianUint64(withdrawalProofs.blockNumberRoot))
    {
    ...
        BeaconChainProofs.verifyWithdrawalProofs(beaconStateRoot, withdrawalProofs, withdrawalFields);
    ...
```
[](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/pods/EigenPod.sol#LL340C62-L340C62)



Inside function `verifyWithdrawalProofs` there are not validation that `slotProof` is at least 32 length and `slotProof % 32 ==0` like all the other proofs in this function.
```solididty
    function verifyWithdrawalProofs(
        bytes32 beaconStateRoot,
        WithdrawalProofs calldata proofs,
        bytes32[] calldata withdrawalFields
    ) internal view {
        require(withdrawalFields.length == 2**WITHDRAWAL_FIELD_TREE_HEIGHT, "BeaconChainProofs.verifyWithdrawalProofs: withdrawalFields has incorrect length");

        require(proofs.blockHeaderRootIndex < 2**BLOCK_ROOTS_TREE_HEIGHT, "BeaconChainProofs.verifyWithdrawalProofs: blockRootIndex is too large");
        require(proofs.withdrawalIndex < 2**WITHDRAWALS_TREE_HEIGHT, "BeaconChainProofs.verifyWithdrawalProofs: withdrawalIndex is too large");

        // verify the block header proof length
        require(proofs.blockHeaderProof.length == 32 * (BEACON_STATE_FIELD_TREE_HEIGHT + BLOCK_ROOTS_TREE_HEIGHT),
            "BeaconChainProofs.verifyWithdrawalProofs: blockHeaderProof has incorrect length");
        require(proofs.withdrawalProof.length == 32 * (EXECUTION_PAYLOAD_HEADER_FIELD_TREE_HEIGHT + WITHDRAWALS_TREE_HEIGHT + 1),
            "BeaconChainProofs.verifyWithdrawalProofs: withdrawalProof has incorrect length");
        require(proofs.executionPayloadProof.length == 32 * (BEACON_BLOCK_HEADER_FIELD_TREE_HEIGHT + BEACON_BLOCK_BODY_FIELD_TREE_HEIGHT),
            "BeaconChainProofs.verifyWithdrawalProofs: executionPayloadProof has incorrect length");

        /**
         * Computes the block_header_index relative to the beaconStateRoot.  It concatenates the indexes of all the
         * intermediate root indexes from the bottom of the sub trees (the block header container) to the top of the tree
         */
        uint256 blockHeaderIndex = BLOCK_ROOTS_INDEX << (BLOCK_ROOTS_TREE_HEIGHT)  | uint256(proofs.blockHeaderRootIndex);
        // Verify the blockHeaderRoot against the beaconStateRoot
        require(Merkle.verifyInclusionSha256(proofs.blockHeaderProof, beaconStateRoot, proofs.blockHeaderRoot, blockHeaderIndex),
            "BeaconChainProofs.verifyWithdrawalProofs: Invalid block header merkle proof");

        //Next we verify the slot against the blockHeaderRoot
        require(Merkle.verifyInclusionSha256(proofs.slotProof, proofs.blockHeaderRoot, proofs.slotRoot, SLOT_INDEX), "BeaconChainProofs.verifyWithdrawalProofs: Invalid slot merkle proof");

        // Next we verify the executionPayloadRoot against the blockHeaderRoot
        uint256 executionPayloadIndex = BODY_ROOT_INDEX << (BEACON_BLOCK_BODY_FIELD_TREE_HEIGHT)| EXECUTION_PAYLOAD_INDEX ;
        require(Merkle.verifyInclusionSha256(proofs.executionPayloadProof, proofs.blockHeaderRoot, proofs.executionPayloadRoot, executionPayloadIndex),
            "BeaconChainProofs.verifyWithdrawalProofs: Invalid executionPayload merkle proof");

        // Next we verify the blockNumberRoot against the executionPayload root
        require(Merkle.verifyInclusionSha256(proofs.blockNumberProof, proofs.executionPayloadRoot, proofs.blockNumberRoot, BLOCK_NUMBER_INDEX),
            "BeaconChainProofs.verifyWithdrawalProofs: Invalid blockNumber merkle proof");

        /**
         * Next we verify the withdrawal fields against the blockHeaderRoot:
         * First we compute the withdrawal_index relative to the blockHeaderRoot by concatenating the indexes of all the
         * intermediate root indexes from the bottom of the sub trees (the withdrawal container) to the top, the blockHeaderRoot.
         * Then we calculate merkleize the withdrawalFields container to calculate the the withdrawalRoot.
         * Finally we verify the withdrawalRoot against the executionPayloadRoot.
         */
        uint256 withdrawalIndex = WITHDRAWALS_INDEX << (WITHDRAWALS_TREE_HEIGHT + 1) | uint256(proofs.withdrawalIndex);
        bytes32 withdrawalRoot = Merkle.merkleizeSha256(withdrawalFields);
        require(Merkle.verifyInclusionSha256(proofs.withdrawalProof, proofs.executionPayloadRoot, withdrawalRoot, withdrawalIndex),
            "BeaconChainProofs.verifyWithdrawalProofs: Invalid withdrawal merkle proof");
    }
```
[contracts/libraries/BeaconChainProofs.sol#L245](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/libraries/BeaconChainProofs.sol#L245)

Therefore its possible to set `slotProof` to any string below 32 length and there will be no Sha256 validation inside `Merkle.sol` file due to `proof.length` <32 and loop start with 32
```solidity
    function processInclusionProofSha256(bytes memory proof, bytes32 leaf, uint256 index) internal view returns (bytes32) {
        bytes32[1] memory computedHash = [leaf];
        for (uint256 i = 32; i <= proof.length; i+=32) {
            if(index % 2 == 0) {
                // if ith bit of index is 0, then computedHash is a left sibling
                assembly {
                    mstore(0x00, mload(computedHash))
                    mstore(0x20, mload(add(proof, i)))
                    if iszero(staticcall(sub(gas(), 2000), 2, 0x00, 0x40, computedHash, 0x20)) {revert(0, 0)}
                    index := div(index, 2)
                }
            } else {
                // if ith bit of index is 1, then computedHash is a right sibling
                assembly {
                    mstore(0x00, mload(add(proof, i)))
                    mstore(0x20, mload(computedHash))
                    if iszero(staticcall(sub(gas(), 2000), 2, 0x00, 0x40, computedHash, 0x20)) {revert(0, 0)}
                    index := div(index, 2)
                }           
            }
        }
        return computedHash[0];
    }

```
[src/contracts/libraries/Merkle.sol#L99](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/libraries/Merkle.sol#L99)

What we need to do in exploit is set `slotRoot` to `blockHeaderRoot` to get a bonus eth. There maybe other ways how to get a reward from more than 1 slot per each validator but I`ve not figured it out.
## Tools Used
POC:
```solidity
    function testPartialWithdrawalFlow() public returns(IEigenPod){
        //this call is to ensure that validator 61068 has proven their withdrawalcreds
        setJSON("./src/test/test-data/withdrawalCredentialAndBalanceProof_61068.json");
        _testDeployAndVerifyNewEigenPod(podOwner, signature, depositDataRoot);
        IEigenPod newPod = eigenPodManager.getPod(podOwner);

        //generate partialWithdrawalProofs.json with:
        // ./solidityProofGen "WithdrawalFieldsProof" 61068 656 "data/slot_58000/oracle_capella_beacon_state_58100.ssz" "data/slot_58000/capella_block_header_58000.json" "data/slot_58000/capella_block_58000.json" "partialWithdrawalProof.json"
        setJSON("./src/test/test-data/partialWithdrawalProof.json");
        BeaconChainProofs.WithdrawalProofs memory withdrawalProofs = _getWithdrawalProof();
        bytes memory validatorFieldsProof = abi.encodePacked(getValidatorProof());

        withdrawalFields = getWithdrawalFields();   
        validatorFields = getValidatorFields();
        bytes32 newBeaconStateRoot = getBeaconStateRoot();
        BeaconChainOracleMock(address(beaconChainOracle)).setBeaconChainStateRoot(newBeaconStateRoot);

        uint64 withdrawalAmountGwei = Endian.fromLittleEndianUint64(withdrawalFields[BeaconChainProofs.WITHDRAWAL_VALIDATOR_AMOUNT_INDEX]);
        uint64 slot = Endian.fromLittleEndianUint64(withdrawalProofs.slotRoot);
        cheats.deal(address(newPod), stakeAmount);   

        uint256 delayedWithdrawalRouterContractBalanceBefore = address(delayedWithdrawalRouter).balance;
        newPod.verifyAndProcessWithdrawal(withdrawalProofs, validatorFieldsProof, validatorFields, withdrawalFields, 0, 0);
//      ------------start POC----------
        withdrawalProofs.slotRoot = withdrawalProofs.blockHeaderRoot; // slotRoot should be the same as blockHeaderRoot
        withdrawalProofs.slotProof = ""; // any length below 32 so loop will be bypassed
        newPod.verifyAndProcessWithdrawal(withdrawalProofs, validatorFieldsProof, validatorFields, withdrawalFields, 0, 0);
//      ------------end POC----------

        uint40 validatorIndex = uint40(getValidatorIndex());
        require(newPod.provenPartialWithdrawal(validatorIndex, slot), "provenPartialWithdrawal should be true");
        withdrawalAmountGwei = uint64(withdrawalAmountGwei*GWEI_TO_WEI);
        require(address(delayedWithdrawalRouter).balance - delayedWithdrawalRouterContractBalanceBefore == withdrawalAmountGwei * 2,
            "pod delayed withdrawal balance hasn't been updated correctly"); // double withdraw

        cheats.roll(block.number + PARTIAL_WITHDRAWAL_FRAUD_PROOF_PERIOD_BLOCKS + 1);
        uint podOwnerBalanceBefore = address(podOwner).balance;
        delayedWithdrawalRouter.claimDelayedWithdrawals(podOwner, 1);
        require(address(podOwner).balance - podOwnerBalanceBefore == withdrawalAmountGwei, "Pod owner balance hasn't been updated correctly");
        return newPod;
    }
```
## Recommended Mitigation Steps
I think its important to add these require inside merkle for security
```diff
    function processInclusionProofSha256(bytes memory proof, bytes32 leaf, uint256 index) internal view returns (bytes32) {
        bytes32[1] memory computedHash = [leaf];
+        require(proof.length % 32 == 0 && proof.length > 0, "Invalid proof length");

        for (uint256 i = 32; i <= proof.length; i+=32) {
            if(index % 2 == 0) {
                // if ith bit of index is 0, then computedHash is a left sibling
                assembly {
                    mstore(0x00, mload(computedHash))
                    mstore(0x20, mload(add(proof, i)))
                    if iszero(staticcall(sub(gas(), 2000), 2, 0x00, 0x40, computedHash, 0x20)) {revert(0, 0)}
                    index := div(index, 2)
                }
            } else {
                // if ith bit of index is 1, then computedHash is a right sibling
                assembly {
                    mstore(0x00, mload(add(proof, i)))
                    mstore(0x20, mload(computedHash))
                    if iszero(staticcall(sub(gas(), 2000), 2, 0x00, 0x40, computedHash, 0x20)) {revert(0, 0)}
                    index := div(index, 2)
                }           
            }
        }
        return computedHash[0];
    }

```

## [H-02]  It is impossible to slash queued withdrawals that contain a malicious strategy due to a misplacement of the ++i increment
## Impact
Detailed description of the impact of this finding.
slashQueuedWithdrawal cannot skip malicious strategies
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Whenever admin would like to skip malicious strategy in the `strategies` array which always reverts on calls to its 'withdraw' function. They will still be triggered. Whevener check `indicesToSkipIndex < indicesToSkip.length && indicesToSkip[indicesToSkipIndex] == i` is in place, array doenst go to the next strategy on list but stays on the same index.
```solidity
function slashQueuedWithdrawal(address recipient, QueuedWithdrawal calldata queuedWithdrawal, IERC20[] calldata tokens, uint256[] calldata indicesToSkip)
    external
    onlyOwner
    onlyFrozen(queuedWithdrawal.delegatedAddress)
    nonReentrant
{
    require(tokens.length == queuedWithdrawal.strategies.length, "StrategyManager.slashQueuedWithdrawal: input length mismatch");

    // find the withdrawalRoot
    bytes32 withdrawalRoot = calculateWithdrawalRoot(queuedWithdrawal);

    // verify that the queued withdrawal is pending
    require(
        withdrawalRootPending[withdrawalRoot],
        "StrategyManager.slashQueuedWithdrawal: withdrawal is not pending"
    );

    // reset the storage slot in mapping of queued withdrawals
    withdrawalRootPending[withdrawalRoot] = false;

    // keeps track of the index in the `indicesToSkip` array
    uint256 indicesToSkipIndex = 0;

    uint256 strategiesLength = queuedWithdrawal.strategies.length;
    for (uint256 i = 0; i < strategiesLength;) {
        // check if the index i matches one of the indices specified in the `indicesToSkip` array
        if (indicesToSkipIndex < indicesToSkip.length && indicesToSkip[indicesToSkipIndex] == i) {
            unchecked {
                ++indicesToSkipIndex;
            }
        } else {
            if (queuedWithdrawal.strategies[i] == beaconChainETHStrategy){
                 //withdraw the beaconChainETH to the recipient
                _withdrawBeaconChainETH(queuedWithdrawal.depositor, recipient, queuedWithdrawal.shares[i]);
            } else {
                // tell the strategy to send the appropriate amount of funds to the recipient
                queuedWithdrawal.strategies[i].withdraw(recipient, tokens[i], queuedWithdrawal.shares[i]);
            }
            unchecked {
                ++i;
            }
        }
    }
}

```
[/src/contracts/core/StrategyManager.sol#L537](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/core/StrategyManager.sol#L537)
## Tools Used
POC
```solidity
function testSlashQueuedWithdrawalNotBeaconChainETH2() external {
    address recipient = address(333);
    uint256 depositAmount = 1e18;
    uint256 withdrawalAmount = depositAmount;
    bool undelegateIfPossible = false;

    (IStrategyManager.QueuedWithdrawal memory queuedWithdrawal, /*IERC20[] memory tokensArray*/, bytes32 withdrawalRoot) =
        testQueueWithdrawal_ToSelf_NotBeaconChainETH(depositAmount, withdrawalAmount, undelegateIfPossible);

    uint256 balanceBefore = dummyToken.balanceOf(address(recipient));

    // slash the delegatedOperator
    slasherMock.freezeOperator(queuedWithdrawal.delegatedAddress);

    cheats.startPrank(strategyManager.owner());
    uint256[] memory emptyUintArray2 = new uint256[](1);
    emptyUintArray2[0] = 0;
    strategyManager.slashQueuedWithdrawal(recipient, queuedWithdrawal, _arrayWithJustDummyToken(), emptyUintArray2);
    cheats.stopPrank();

    uint256 balanceAfter = dummyToken.balanceOf(address(recipient));

    require(balanceAfter == balanceBefore, "balance should be equal to before because we skip it inside emptyUintArray2");// should pass but it doesn't
    require(!strategyManager.withdrawalRootPending(withdrawalRoot), "withdrawalRootPendingAfter is true!");
}

```
## Recommended Mitigation Steps
Move `++i` outside of if block
```diff
function slashQueuedWithdrawal(address recipient, QueuedWithdrawal calldata queuedWithdrawal, IERC20[] calldata tokens, uint256[] calldata indicesToSkip)
    external
    onlyOwner
    onlyFrozen(queuedWithdrawal.delegatedAddress)
    nonReentrant
{
    require(tokens.length == queuedWithdrawal.strategies.length, "StrategyManager.slashQueuedWithdrawal: input length mismatch");

    // find the withdrawalRoot
    bytes32 withdrawalRoot = calculateWithdrawalRoot(queuedWithdrawal);

    // verify that the queued withdrawal is pending
    require(
        withdrawalRootPending[withdrawalRoot],
        "StrategyManager.slashQueuedWithdrawal: withdrawal is not pending"
    );

    // reset the storage slot in mapping of queued withdrawals
    withdrawalRootPending[withdrawalRoot] = false;

    // keeps track of the index in the `indicesToSkip` array
    uint256 indicesToSkipIndex = 0;

    uint256 strategiesLength = queuedWithdrawal.strategies.length;
    for (uint256 i = 0; i < strategiesLength;) {
        // check if the index i matches one of the indices specified in the `indicesToSkip` array
        if (indicesToSkipIndex < indicesToSkip.length && indicesToSkip[indicesToSkipIndex] == i) {
            unchecked {
                ++indicesToSkipIndex;
            }
        } else {
            if (queuedWithdrawal.strategies[i] == beaconChainETHStrategy){
                 //withdraw the beaconChainETH to the recipient
                _withdrawBeaconChainETH(queuedWithdrawal.depositor, recipient, queuedWithdrawal.shares[i]);
            } else {
                // tell the strategy to send the appropriate amount of funds to the recipient
                queuedWithdrawal.strategies[i].withdraw(recipient, tokens[i], queuedWithdrawal.shares[i]);
            }
        }
+            unchecked {
+                ++i;
+            }
    }
}

```

## [L-01]  computePhase0Eth1DataRoot always returns an incorrect Merkle tree

## Impact
Detailed description of the impact of this finding.
The Merkle tree creation inside the computePhase0Eth1DataRoot function is incorrect
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Not all fields of eth1DataFields being used in an array due to usage of `i < ETH1_DATA_FIELD_TREE_HEIGHT` instead of `i<NUM_ETH1_DATA_FIELDS`
Check other similar function.
```solidity
    function computePhase0Eth1DataRoot(bytes32[NUM_ETH1_DATA_FIELDS] calldata eth1DataFields) internal pure returns(bytes32) { 
        bytes32[] memory paddedEth1DataFields = new bytes32[](2**ETH1_DATA_FIELD_TREE_HEIGHT);

        for (uint256 i = 0; i < ETH1_DATA_FIELD_TREE_HEIGHT; ++i) {
            paddedEth1DataFields[i] = eth1DataFields[i];
        }

        return Merkle.merkleizeSha256(paddedEth1DataFields);
    }

```
[src/contracts/libraries/BeaconChainProofs.sol#L160](https://github.com/Layr-Labs/eigenlayer-contracts/blob/eccdfd43bb882d66a68cad8875dde2979e204546/src/contracts/libraries/BeaconChainProofs.sol#L160)
## Tools Used
Manual
## Recommended Mitigation Steps

```diff
    function computePhase0Eth1DataRoot(bytes32[NUM_ETH1_DATA_FIELDS] calldata eth1DataFields) internal pure returns(bytes32) { 
        bytes32[] memory paddedEth1DataFields = new bytes32[](2**ETH1_DATA_FIELD_TREE_HEIGHT);

_        for (uint256 i = 0; i < ETH1_DATA_FIELD_TREE_HEIGHT; ++i) {
+        for (uint256 i = 0; i < NUM_ETH1_DATA_FIELDS; ++i) {
           paddedEth1DataFields[i] = eth1DataFields[i];
        }

        return Merkle.merkleizeSha256(paddedEth1DataFields);
    }

```


## Assessed type
Math

## [L-02]  processInclusionProofKeccak does not work as expected
## Impact
Detailed description of the impact of this finding.
processInclusionProofKeccak does not work as expected
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
function `verifyInclusionKeccak` is not used anywhere but its in the scope of this contest. There is no validation that proof is a tree and a valid tree like it described in the comments. E.x. if proof is less than 32 length that function will just return a leaf without reverting. In my opinion function doesn't work as expected and can be exploited. I've submitted the same issue with `processInclusionProofSha256` function that lead to loss a funds for validator due the same issue.
```solidity
    function processInclusionProofKeccak(bytes memory proof, bytes32 leaf, uint256 index) internal pure returns (bytes32) {
        bytes32 computedHash = leaf;
        for (uint256 i = 32; i <= proof.length; i+=32) {
            if(index % 2 == 0) {
                // if ith bit of index is 0, then computedHash is a left sibling
                assembly {
                    mstore(0x00, computedHash)
                    mstore(0x20, mload(add(proof, i)))
                    computedHash := keccak256(0x00, 0x40)
                    index := div(index, 2)
                }
            } else {
                // if ith bit of index is 1, then computedHash is a right sibling
                assembly {
                    mstore(0x00, mload(add(proof, i)))
                    mstore(0x20, computedHash)
                    computedHash := keccak256(0x00, 0x40)
                    index := div(index, 2)
                }           
            }
        }
        return computedHash;
    }
```
[src/contracts/libraries/Merkle.sol#L49](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/libraries/Merkle.sol#L48)
## Tools Used

## Recommended Mitigation Steps
I think its important to add security to that function like this
```diff
    function processInclusionProofKeccak(bytes memory proof, bytes32 leaf, uint256 index) internal pure returns (bytes32) {
+        require(proof.length % 32 == 0 && proof.length > 0, "Invalid proof length");

        bytes32 computedHash = leaf;
        for (uint256 i = 32; i <= proof.length; i+=32) {
            if(index % 2 == 0) {
                // if ith bit of index is 0, then computedHash is a left sibling
                assembly {
                    mstore(0x00, computedHash)
                    mstore(0x20, mload(add(proof, i)))
                    computedHash := keccak256(0x00, 0x40)
                    index := div(index, 2)
                }
            } else {
                // if ith bit of index is 1, then computedHash is a right sibling
                assembly {
                    mstore(0x00, mload(add(proof, i)))
                    mstore(0x20, computedHash)
                    computedHash := keccak256(0x00, 0x40)
                    index := div(index, 2)
                }           
            }
        }
        return computedHash;
    }
```

## [L-03] merkleizeSha256 doesn’t work as expected

## Impact
Detailed description of the impact of this finding.
merkleizeSha256  doesn't work as expected
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
Whenever merkleizeSha256 is being used in the code there is always a check that array length is power of 2. E.x.
```solidity
bytes32[] memory paddedHeaderFields = new bytes32[](2**BEACON_BLOCK_HEADER_FIELD_TREE_HEIGHT);
```
[contracts/libraries/BeaconChainProofs.sol#L131](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/libraries/BeaconChainProofs.sol#L131)

But inside the function `merkleizeSha256` there is no check that incoming array is power of 2
```solidity
/**
  @notice this function returns the merkle root of a tree created from a set of leaves using sha256 as its hash function
  @param leaves the leaves of the merkle tree

  @notice requires the leaves.length is a power of 2
  */
 function merkleizeSha256(
     bytes32[] memory leaves
 ) internal pure returns (bytes32) {
     //there are half as many nodes in the layer above the leaves
     uint256 numNodesInLayer = leaves.length / 2;
     //create a layer to store the internal nodes
     bytes32[] memory layer = new bytes32[](numNodesInLayer);
     //fill the layer with the pairwise hashes of the leaves
     for (uint i = 0; i < numNodesInLayer; i++) {
         layer[i] = sha256(abi.encodePacked(leaves[2*i], leaves[2*i+1]));
     }
     //the next layer above has half as many nodes
     numNodesInLayer /= 2;
     //while we haven't computed the root
     while (numNodesInLayer != 0) {
         //overwrite the first numNodesInLayer nodes in layer with the pairwise hashes of their children
         for (uint i = 0; i < numNodesInLayer; i++) {
             layer[i] = sha256(abi.encodePacked(layer[2*i], layer[2*i+1]));
         }
         //the next layer above has half as many nodes
         numNodesInLayer /= 2;
     }
     //the first node in the layer is the root
     return layer[0];
 }
```

There is a @notice that doesn't hold
>  @notice requires the leaves.length is a power of 2

But whenever there is a require in natspec inside the project it always holds. E.x.
```
 /**
  * @notice Delegates from `staker` to `operator`.
  * @dev requires that:
  * 1) if `staker` is an EOA, then `signature` is valid ECSDA signature from `staker`, indicating their intention for this action
  * 2) if `staker` is a contract, then `signature` must will be checked according to EIP-1271
  */
```
[src/contracts/core/DelegationManager.sol#L89](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/core/DelegationManager.sol#L89)
```solidity
  * WARNING: In order to mitigate against inflation/donation attacks in the context of ERC_4626, this contract requires the
  *          minimum amount of shares be either 0 or 1e9. A consequence of this is that in the worst case a user will not
  *          be able to withdraw for 1e9-1 or less shares.
  *
```
[/src/contracts/strategies/StrategyBase.sol#L72](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/strategies/StrategyBase.sol#L72)
## Tools Used
You can insert this into remix to check
```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";

contract Owner {

 mapping(address => bool) internal frozenStatus;
 constructor() {
 }

 function dod() external returns (bytes32){
     bytes32[] memory leaves = new bytes32[](7);
     for (uint256 i = 0; i < 7; ++i) {
         leaves[i] = bytes32(i);
     }
     return merkleizeSha256(leaves);
 }

 function merkleizeSha256(
     bytes32[] memory leaves
 ) internal pure returns (bytes32) {
     //there are half as many nodes in the layer above the leaves
     uint256 numNodesInLayer = leaves.length / 2;
     //create a layer to store the internal nodes
     bytes32[] memory layer = new bytes32[](numNodesInLayer);
     //fill the layer with the pairwise hashes of the leaves
     for (uint i = 0; i < numNodesInLayer; i++) {
         layer[i] = sha256(abi.encodePacked(leaves[2*i], leaves[2*i+1]));
     }
     //the next layer above has half as many nodes
     numNodesInLayer /= 2;
     //while we haven't computed the root
     while (numNodesInLayer != 0) {
         //overwrite the first numNodesInLayer nodes in layer with the pairwise hashes of their children
         for (uint i = 0; i < numNodesInLayer; i++) {
             layer[i] = sha256(abi.encodePacked(layer[2*i], layer[2*i+1]));
         }
         //the next layer above has half as many nodes
         numNodesInLayer /= 2;
     }
     //the first node in the layer is the root
     return layer[0];
 }
}
```
## Recommended Mitigation Steps
Either remove @notice or add this code for more security because sometimes you can just forget to check arrey size before calling that function
```diff
 function merkleizeSha256(
     bytes32[] memory leaves
 ) internal pure returns (bytes32) {
+        uint256 len = leaves.length;
+        while (len > 1 && len % 2 == 0) {
+            len /= 2;
+        }
+        require(len==1, "requires the leaves.length is a power of 2");
     //there are half as many nodes in the layer above the leaves
     uint256 numNodesInLayer = leaves.length / 2;
     //create a layer to store the internal nodes
     bytes32[] memory layer = new bytes32[](numNodesInLayer);
     //fill the layer with the pairwise hashes of the leaves
     for (uint i = 0; i < numNodesInLayer; i++) {
         layer[i] = sha256(abi.encodePacked(leaves[2*i], leaves[2*i+1]));
     }
     //the next layer above has half as many nodes
     numNodesInLayer /= 2;
     //while we haven't computed the root
     while (numNodesInLayer != 0) {
         //overwrite the first numNodesInLayer nodes in layer with the pairwise hashes of their children
         for (uint i = 0; i < numNodesInLayer; i++) {
             layer[i] = sha256(abi.encodePacked(layer[2*i], layer[2*i+1]));
         }
         //the next layer above has half as many nodes
         numNodesInLayer /= 2;
     }
     //the first node in the layer is the root
     return layer[0];
 }

```
Remix
```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";

contract Owner {

 mapping(address => bool) internal frozenStatus;
 constructor() {
 }

 function dod(uint len) external returns (bytes32){
     bytes32[] memory leaves = new bytes32[](len);
     for (uint256 i = 0; i < len; ++i) {
         leaves[i] = bytes32(i);
     }
     return merkleizeSha256(leaves);
 }
 function merkleizeSha256(
     bytes32[] memory leaves
 ) internal pure returns (bytes32) {
     uint256 len = leaves.length;
     while (len > 1 && len % 2 == 0) {
         len /= 2;
     }
     require(len==1, "requires the leaves.length is a power of 2");
     //there are half as many nodes in the layer above the leaves
     uint256 numNodesInLayer = leaves.length / 2;
     //create a layer to store the internal nodes
     bytes32[] memory layer = new bytes32[](numNodesInLayer);
     //fill the layer with the pairwise hashes of the leaves
     for (uint i = 0; i < numNodesInLayer; i++) {
         layer[i] = sha256(abi.encodePacked(leaves[2*i], leaves[2*i+1]));
     }
     //the next layer above has half as many nodes
     numNodesInLayer /= 2;
     //while we haven't computed the root
     while (numNodesInLayer != 0) {
         //overwrite the first numNodesInLayer nodes in layer with the pairwise hashes of their children
         for (uint i = 0; i < numNodesInLayer; i++) {
             layer[i] = sha256(abi.encodePacked(layer[2*i], layer[2*i+1]));
         }
         //the next layer above has half as many nodes
         numNodesInLayer /= 2;
     }
     //the first node in the layer is the root
     return layer[0];
 }


}
```
## [L-04]  claimableUserDelayedWithdrawals sometimes returns unclaimable DelayedWithdrawals, so users will see incorrect data
## Impact
Detailed description of the impact of this finding.
claimableUserDelayedWithdrawals sometimes returns unclaimable DelayedWithdrawals, so users will see incorrect data

## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
The "canClaimDelayedWithdrawal" function will return false for a withdrawal for which the block duration has not passed. The same restriction will be checked whenever an actual withdrawal is triggered, but the "claimableUserDelayedWithdrawals" function does not take into account block duration validation.

```solidity
function claimableUserDelayedWithdrawals(address user) external view returns (DelayedWithdrawal[] memory) {
    uint256 delayedWithdrawalsCompleted = _userWithdrawals[user].delayedWithdrawalsCompleted;
    uint256 delayedWithdrawalsLength = _userWithdrawals[user].delayedWithdrawals.length;
    uint256 claimableDelayedWithdrawalsLength = delayedWithdrawalsLength - delayedWithdrawalsCompleted;
    DelayedWithdrawal[] memory claimableDelayedWithdrawals = new DelayedWithdrawal[](claimableDelayedWithdrawalsLength);
    for (uint256 i = 0; i < claimableDelayedWithdrawalsLength; i++) {
        claimableDelayedWithdrawals[i] = _userWithdrawals[user].delayedWithdrawals[delayedWithdrawalsCompleted + i];
    }
    return claimableDelayedWithdrawals;
}
...
function canClaimDelayedWithdrawal(address user, uint256 index) external view returns (bool) {
    return ((index >= _userWithdrawals[user].delayedWithdrawalsCompleted) && (block.number >= _userWithdrawals[user].delayedWithdrawals[index].blockCreated + withdrawalDelayBlocks));
}
```
[src/contracts/pods/DelayedWithdrawalRouter.sol#L110](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/pods/DelayedWithdrawalRouter.sol#L110)
## Tools Used
Manual
## Recommended Mitigation Steps

```solidity
function claimableUserDelayedWithdrawals(address user) external view returns (DelayedWithdrawal[] memory) {
    uint256 delayedWithdrawalsCompleted = _userWithdrawals[user].delayedWithdrawalsCompleted;
    uint256 delayedWithdrawalsLength = _userWithdrawals[user].delayedWithdrawals.length;
    uint256 claimableDelayedWithdrawalsLength = delayedWithdrawalsLength - delayedWithdrawalsCompleted;
    DelayedWithdrawal[] memory claimableDelayedWithdrawals;
    for (uint256 i = 0; i < claimableDelayedWithdrawalsLength; i++) {
        if (block.number < _userWithdrawals[user].delayedWithdrawals[delayedWithdrawalsCompleted + i].blockCreated + withdrawalDelayBlocks) {
            break;
        }
        claimableDelayedWithdrawals.push(_userWithdrawals[user].delayedWithdrawals[delayedWithdrawalsCompleted + i]);
    }
    return claimableDelayedWithdrawals;
}

```

## [L-05] The condition for full withdrawals in the code is different from that in the documentation

## Impact
Detailed description of the impact of this finding.
The condition for full withdrawals in the code is different from that in the documentation.
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
The condition in [docs](https://github.com/code-423n4/2023-04-eigenlayer/blob/138cf7edb887f641ae48e33e963ab1be4ff474c1/docs/EigenPods.md) for full withdrawal is `validator.withdrawableEpoch < executionPayload.slot/SLOTS_PER_EPOCH` while in the code its `validator.withdrawableEpoch <= executionPayload.slot/SLOTS_PER_EPOCH`

```solidity
function verifyAndProcessWithdrawal(
    BeaconChainProofs.WithdrawalProofs calldata withdrawalProofs,
    bytes calldata validatorFieldsProof,
    bytes32[] calldata validatorFields,
    bytes32[] calldata withdrawalFields,
    uint256 beaconChainETHStrategyIndex,
    uint64 oracleBlockNumber
)
...
    // reference: uint64 withdrawableEpoch = Endian.fromLittleEndianUint64(validatorFields[BeaconChainProofs.VALIDATOR_WITHDRAWABLE_EPOCH_INDEX]);
    if (Endian.fromLittleEndianUint64(validatorFields[BeaconChainProofs.VALIDATOR_WITHDRAWABLE_EPOCH_INDEX]) <= slot/BeaconChainProofs.SLOTS_PER_EPOCH) {
        _processFullWithdrawal(withdrawalAmountGwei, validatorIndex, beaconChainETHStrategyIndex, podOwner, validatorStatus[validatorIndex]);
    } else {
        _processPartialWithdrawal(slot, withdrawalAmountGwei, validatorIndex, podOwner);
    }
}

```
[src/contracts/pods/EigenPod.sol#L354](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/pods/EigenPod.sol#L354)
## Tools Used

## Recommended Mitigation Steps
Synchronize them with each other.

## [L-06] Missing validation to a threshold value on full withdrawal

## Impact
Detailed description of the impact of this finding.
Missing validation to a threshold value on full withdrawal.
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.
According to the [docs](https://github.com/code-423n4/2023-04-eigenlayer/blob/138cf7edb887f641ae48e33e963ab1be4ff474c1/docs/EigenPods.md) there suppose to be a validation against a const on full withdrawal, but its missing which lead to system not work as expected.
>In this second case, in order to withdraw their balance from the EigenPod, stakers must provide a valid proof of their full withdrawal (differentiated from partial withdrawals through a simple comparison of the amount to a threshold value named MIN_FULL_WITHDRAWAL_AMOUNT_GWEI) against a beacon state root.
```solidity
function _processFullWithdrawal(
    uint64 withdrawalAmountGwei,
    uint40 validatorIndex,
    uint256 beaconChainETHStrategyIndex,
    address recipient,
    VALIDATOR_STATUS status
) internal {
    uint256 amountToSend;

    // if the validator has not previously been proven to be "overcommitted"
    if (status == VALIDATOR_STATUS.ACTIVE) {
        // if the withdrawal amount is greater than the REQUIRED_BALANCE_GWEI (i.e. the amount restaked on EigenLayer, per ETH validator)
        if (withdrawalAmountGwei >= REQUIRED_BALANCE_GWEI) {
            // then the excess is immediately withdrawable
            amountToSend = uint256(withdrawalAmountGwei - REQUIRED_BALANCE_GWEI) * uint256(GWEI_TO_WEI);
            // and the extra execution layer ETH in the contract is REQUIRED_BALANCE_GWEI, which must be withdrawn through EigenLayer's normal withdrawal process
            restakedExecutionLayerGwei += REQUIRED_BALANCE_GWEI;
        } else {
            // otherwise, just use the full withdrawal amount to continue to "back" the podOwner's remaining shares in EigenLayer (i.e. none is instantly withdrawable)
            restakedExecutionLayerGwei += withdrawalAmountGwei;
            // remove and undelegate 'extra' (i.e. "overcommitted") shares in EigenLayer
            eigenPodManager.recordOvercommittedBeaconChainETH(podOwner, beaconChainETHStrategyIndex, uint256(REQUIRED_BALANCE_GWEI - withdrawalAmountGwei) * GWEI_TO_WEI);
        }
    // if the validator *has* previously been proven to be "overcommitted"
    } else if (status == VALIDATOR_STATUS.OVERCOMMITTED) {
        // if the withdrawal amount is greater than the REQUIRED_BALANCE_GWEI (i.e. the amount restaked on EigenLayer, per ETH validator)
        if (withdrawalAmountGwei >= REQUIRED_BALANCE_GWEI) {
            // then the excess is immediately withdrawable
            amountToSend = uint256(withdrawalAmountGwei - REQUIRED_BALANCE_GWEI) * uint256(GWEI_TO_WEI);
            // and the extra execution layer ETH in the contract is REQUIRED_BALANCE_GWEI, which must be withdrawn through EigenLayer's normal withdrawal process
            restakedExecutionLayerGwei += REQUIRED_BALANCE_GWEI;
            /**
             * since in `verifyOvercommittedStake` the podOwner's beaconChainETH shares are decremented by `REQUIRED_BALANCE_WEI`, we must reverse the process here,
             * in order to allow the podOwner to complete their withdrawal through EigenLayer's normal withdrawal process
             */
            eigenPodManager.restakeBeaconChainETH(podOwner, REQUIRED_BALANCE_WEI);
        } else {
            // otherwise, just use the full withdrawal amount to continue to "back" the podOwner's remaining shares in EigenLayer (i.e. none is instantly withdrawable)
            restakedExecutionLayerGwei += withdrawalAmountGwei;
            /**
             * since in `verifyOvercommittedStake` the podOwner's beaconChainETH shares are decremented by `REQUIRED_BALANCE_WEI`, we must reverse the process here,
             * in order to allow the podOwner to complete their withdrawal through EigenLayer's normal withdrawal process
             */
            eigenPodManager.restakeBeaconChainETH(podOwner, uint256(withdrawalAmountGwei) * GWEI_TO_WEI);
        }
    // If the validator status is withdrawn, they have already processed their ETH withdrawal
    }  else {
        revert("EigenPod.verifyBeaconChainFullWithdrawal: VALIDATOR_STATUS is WITHDRAWN or invalid VALIDATOR_STATUS");
    }

    // set the ETH validator status to withdrawn
    validatorStatus[validatorIndex] = VALIDATOR_STATUS.WITHDRAWN;

    emit FullWithdrawalRedeemed(validatorIndex, recipient, withdrawalAmountGwei);

    // send ETH to the `recipient`, if applicable
    if (amountToSend != 0) {
        _sendETH(recipient, amountToSend);
    }
}
```
[src/contracts/pods/EigenPod.sol#L364](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/pods/EigenPod.sol#L364)
## Tools Used

## Recommended Mitigation Steps
```diff
function _processFullWithdrawal(
    uint64 withdrawalAmountGwei,
    uint40 validatorIndex,
    uint256 beaconChainETHStrategyIndex,
    address recipient,
    VALIDATOR_STATUS status
) internal {
+            require(withdrawalAmountGwei >= MIN_FULL_WITHDRAWAL_AMOUNT_GWEI,
+            "stakers must provide a valid proof of their full withdrawal");

    uint256 amountToSend;

    // if the validator has not previously been proven to be "overcommitted"
    if (status == VALIDATOR_STATUS.ACTIVE) {
        // if the withdrawal amount is greater than the REQUIRED_BALANCE_GWEI (i.e. the amount restaked on EigenLayer, per ETH validator)
        if (withdrawalAmountGwei >= REQUIRED_BALANCE_GWEI) {
            // then the excess is immediately withdrawable
            amountToSend = uint256(withdrawalAmountGwei - REQUIRED_BALANCE_GWEI) * uint256(GWEI_TO_WEI);
            // and the extra execution layer ETH in the contract is REQUIRED_BALANCE_GWEI, which must be withdrawn through EigenLayer's normal withdrawal process
            restakedExecutionLayerGwei += REQUIRED_BALANCE_GWEI;
        } else {
            // otherwise, just use the full withdrawal amount to continue to "back" the podOwner's remaining shares in EigenLayer (i.e. none is instantly withdrawable)
            restakedExecutionLayerGwei += withdrawalAmountGwei;
            // remove and undelegate 'extra' (i.e. "overcommitted") shares in EigenLayer
            eigenPodManager.recordOvercommittedBeaconChainETH(podOwner, beaconChainETHStrategyIndex, uint256(REQUIRED_BALANCE_GWEI - withdrawalAmountGwei) * GWEI_TO_WEI);
        }
    // if the validator *has* previously been proven to be "overcommitted"
    } else if (status == VALIDATOR_STATUS.OVERCOMMITTED) {
        // if the withdrawal amount is greater than the REQUIRED_BALANCE_GWEI (i.e. the amount restaked on EigenLayer, per ETH validator)
        if (withdrawalAmountGwei >= REQUIRED_BALANCE_GWEI) {
            // then the excess is immediately withdrawable
            amountToSend = uint256(withdrawalAmountGwei - REQUIRED_BALANCE_GWEI) * uint256(GWEI_TO_WEI);
            // and the extra execution layer ETH in the contract is REQUIRED_BALANCE_GWEI, which must be withdrawn through EigenLayer's normal withdrawal process
            restakedExecutionLayerGwei += REQUIRED_BALANCE_GWEI;
            /**
             * since in `verifyOvercommittedStake` the podOwner's beaconChainETH shares are decremented by `REQUIRED_BALANCE_WEI`, we must reverse the process here,
             * in order to allow the podOwner to complete their withdrawal through EigenLayer's normal withdrawal process
             */
            eigenPodManager.restakeBeaconChainETH(podOwner, REQUIRED_BALANCE_WEI);
        } else {
            // otherwise, just use the full withdrawal amount to continue to "back" the podOwner's remaining shares in EigenLayer (i.e. none is instantly withdrawable)
            restakedExecutionLayerGwei += withdrawalAmountGwei;
            /**
             * since in `verifyOvercommittedStake` the podOwner's beaconChainETH shares are decremented by `REQUIRED_BALANCE_WEI`, we must reverse the process here,
             * in order to allow the podOwner to complete their withdrawal through EigenLayer's normal withdrawal process
             */
            eigenPodManager.restakeBeaconChainETH(podOwner, uint256(withdrawalAmountGwei) * GWEI_TO_WEI);
        }
    // If the validator status is withdrawn, they have already processed their ETH withdrawal
    }  else {
        revert("EigenPod.verifyBeaconChainFullWithdrawal: VALIDATOR_STATUS is WITHDRAWN or invalid VALIDATOR_STATUS");
    }

    // set the ETH validator status to withdrawn
    validatorStatus[validatorIndex] = VALIDATOR_STATUS.WITHDRAWN;

    emit FullWithdrawalRedeemed(validatorIndex, recipient, withdrawalAmountGwei);

    // send ETH to the `recipient`, if applicable
    if (amountToSend != 0) {
        _sendETH(recipient, amountToSend);
    }
}

```

## [L-07] User can stake twice on beacon chain from same eipod, thus losing funds due to same withdrawal credentials

## Impact
Detailed description of the impact of this finding.
User can stake twice on beacon chain from eipod with the same params thus losing funds due to having same input params
## Proof of Concept
Provide direct links to all referenced code in GitHub. Add screenshots, logs, or any other relevant proof that illustrates the concept.

There are no restriction to how many times user can stake on beacon with EigenPodManager on EigenPod thus all of them will have the same `_podWithdrawalCredentials()` and I think first deposit will be lost
```sodlidity
    function stake(bytes calldata pubkey, bytes calldata signature, bytes32 depositDataRoot) external payable onlyEigenPodManager {
        // stake on ethpos
        require(msg.value == 32 ether, "EigenPod.stake: must initially stake for any validator with 32 ether");
        ethPOS.deposit{value : 32 ether}(pubkey, _podWithdrawalCredentials(), signature, depositDataRoot);
        emit EigenPodStaked(pubkey);
    }
```
[src/contracts/pods/EigenPod.sol#L159](https://github.com/code-423n4/2023-04-eigenlayer/blob/398cc428541b91948f717482ec973583c9e76232/src/contracts/pods/EigenPod.sol#L159)
There are some ways how users can make a mistake by calling it twice or they would like to create another one.
I've looked into rocketpool contracts they are not allowing users to stake twice with the same pubkeys, so I think its important to implement the same security issue.
```solidity
    function preStake(bytes calldata _validatorPubkey, bytes calldata _validatorSignature, bytes32 _depositDataRoot) internal {
...
        require(rocketMinipoolManager.getMinipoolByPubkey(_validatorPubkey) == address(0x0), "Validator pubkey is in use");
        // Set minipool pubkey
        rocketMinipoolManager.setMinipoolPubkey(_validatorPubkey);
        // Get withdrawal credentials
        bytes memory withdrawalCredentials = rocketMinipoolManager.getMinipoolWithdrawalCredentials(address(this));
        // Send staking deposit to casper
        casperDeposit.deposit{value : prelaunchAmount}(_validatorPubkey, withdrawalCredentials, _validatorSignature, _depositDataRoot);
        // Emit event
        emit MinipoolPrestaked(_validatorPubkey, _validatorSignature, _depositDataRoot, prelaunchAmount, withdrawalCredentials, block.timestamp);
    }
```
[contracts/contract/minipool/RocketMinipoolDelegate.sol#L235](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/minipool/RocketMinipoolDelegate.sol#L235)

## Tools Used
POC
```diff
    function testWithdrawFromPod() public {
        cheats.startPrank(podOwner);
        eigenPodManager.stake{value: stakeAmount}(pubkey, signature, depositDataRoot);
+        eigenPodManager.stake{value: stakeAmount}(pubkey, signature, depositDataRoot);
        cheats.stopPrank();

        IEigenPod pod = eigenPodManager.getPod(podOwner);
        uint256 balance = address(pod).balance;
        cheats.deal(address(pod), stakeAmount);

        cheats.startPrank(podOwner);
        cheats.expectEmit(true, false, false, false);
        emit DelayedWithdrawalCreated(podOwner, podOwner, balance, delayedWithdrawalRouter.userWithdrawalsLength(podOwner));
        pod.withdrawBeforeRestaking();
        cheats.stopPrank();
        require(address(pod).balance == 0, "Pod balance should be 0");
    }

```
## Recommended Mitigation Steps
You can look at rocketpool contracts and borrow their logic
```diff
    function stake(bytes calldata pubkey, bytes calldata signature, bytes32 depositDataRoot) external payable onlyEigenPodManager {
+        require(EigenPodManager.getEigenPodByPubkey(_validatorPubkey) == address(0x0), "Validator pubkey is in use");
+        EigenPodManager.setEigenPodByPubkey(_validatorPubkey);

        require(msg.value == 32 ether, "EigenPod.stake: must initially stake for any validator with 32 ether");
        ethPOS.deposit{value : 32 ether}(pubkey, _podWithdrawalCredentials(), signature, depositDataRoot);
        emit EigenPodStaked(pubkey);
    }

```
