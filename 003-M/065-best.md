Delightful Gingerbread Carp

medium

# User will be blocked from finalizing and proving a withdrawal in specific conditions

## Summary

In specific conditions, a user is not able to finalize a withdrawal with a `DEFENDER_WINS` game.

Moreover, the user is not able to prove this same withdrawal with a new game.

## Vulnerability Detail

A user must use the `proveWithdrawalTransaction` to prove a withdrawal through a game. Calling this function will attach the provided game and current timestamp to the hash of the withdrawal. Note that this function doesn't check the time at which the game has been created.

Then, the user use `finalizeWithdrawalTransactionExternalProof` to finalize a transaction with a dispute game for which status is `DEFENDER_WINS`. The withdrawal is checked in `checkWithdrawal`, and a check is done to ensure that `provenWithdrawal.timestamp > createdAt`.

When the user creates the dispute game and calls `proveWithdrawalTransaction` in a single block, the user will then not be able to finalize the withdrawal. Moreover, he will not be able to prove the withdrawal with a new game because this is not a condition for reproving.

## Impact

Breaking a liveness invariant:
> Users must be able to execute messages that have actually been sent and included in the state of the L2ToL1MessagePasser contract on the L2 blockchain. Users must not be permanently prevented from executing a valid message (temporary blocks are valid findings but less severe).

A user may not be able to execute a valid message, because he is not able to both finalize a transaction or reprove this transaction.

## Code Snippet

[`proveWithdrawalTransaction` function](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L242-L325) doesn't implement any check to ensure that `FaultDisputeGame.createdAt() < block.timestamp`.

So, a user will be able to call the function when `FaultDisputeGame.createdAt() == block.timestamp`. Then, the dispute game is stored in storage with the current timestamp (see following code).

```solidity
    function proveWithdrawalTransaction(
        Types.WithdrawalTransaction memory _tx,
        uint256 _disputeGameIndex,
        Types.OutputRootProof calldata _outputRootProof,
        bytes[] calldata _withdrawalProof
    )
        external
        whenNotPaused
    {

        // @POC REDACTED


        // Designate the withdrawalHash as proven by storing the `disputeGameProxy` & `timestamp` in the
        // `provenWithdrawals` mapping. A `withdrawalHash` can only be proven once unless the dispute game it proved
        // against resolves against the favor of the root claim.
        provenWithdrawals[withdrawalHash][msg.sender] =
            ProvenWithdrawal({ disputeGameProxy: gameProxy, timestamp: uint64(block.timestamp) }); // @POC: The `block.timestamp` is stored.

        // Emit a `WithdrawalProven` event.
        emit WithdrawalProven(withdrawalHash, _tx.sender, _tx.target);

        // Add the proof submitter to the list of proof submitters for this withdrawal hash.
        proofSubmitters[withdrawalHash].push(msg.sender);
    }
```

After some delay, the game resolves to `DEFENDER_WINS`. It can be used in `finalizeWithdrawalTransactionExternalProof` for finalization. This [function then calls `checkWithdrawal`](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L354) to verify the withdrawal and its associated game.


[`checkWithdrawal` ensures that the game **was created before** being used with `proveWithdrawalTransaction`](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L478-L481):

```solidity
    function checkWithdrawal(bytes32 _withdrawalHash, address _proofSubmitter) public view {
        // @POC: REDACTED

        // As a sanity check, we make sure that the proven withdrawal's timestamp is greater than
        // starting timestamp inside the Dispute Game. Not strictly necessary but extra layer of
        // safety against weird bugs in the proving step.
        require(
            provenWithdrawal.timestamp > createdAt,
            "OptimismPortal: withdrawal timestamp less than dispute game creation timestamp"
        );

        //@POC: REDACTED
    }
```

As shown previously, the user is able to call `proveWithdrawalTransaction` when `FaultDisputeGame.createdAt() == block.timestamp`. But then, he will not be able to finalize this withdrawal even if the game indicates valid claim due to this condition in `checkWithdrawal`.

The user would then like to **reprove** its claim. He creates a new game and waits until `FaultDisputeGame.createdAt() < block.timestamp`. Finally, he calls `proveWithdrawalTransaction` with this new game.

But this call to replace the previous game will fail because [the previous game doesn't match the expected conditions for reproving](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L287-L291):

```solidity
    function proveWithdrawalTransaction(
        Types.WithdrawalTransaction memory _tx,
        uint256 _disputeGameIndex,
        Types.OutputRootProof calldata _outputRootProof,
        bytes[] calldata _withdrawalProof
    )
        external
        whenNotPaused
    {
        //@POC: REDACTED

        // We generally want to prevent users from proving the same withdrawal multiple times
        // because each successive proof will update the timestamp. A malicious user can take
        // advantage of this to prevent other users from finalizing their withdrawal. However,
        // in the case that an honest user proves their withdrawal against a dispute game that
        // resolves against the root claim, or the dispute game is blacklisted, we allow
        // re-proving the withdrawal against a new proposal.
        IDisputeGame oldGame = provenWithdrawal.disputeGameProxy;
        require(
            provenWithdrawal.timestamp == 0 || oldGame.status() == GameStatus.CHALLENGER_WINS
                || disputeGameBlacklist[oldGame] || oldGame.gameType().raw() != respectedGameType.raw(),
            "OptimismPortal: withdrawal hash has already been proven, and the old dispute game is not invalid"
        ); // @POC: Previous game is not blacklisted or different or CHALLENGER_WINS

        //@POC: REDACTED
    }
```

As these conditions are not true about the previous game and that the timestamp was initialized, the user is not able to prove the withdrawal with a new game.

## Proof of Concept

Import the following unit test in `test/L1/OptimismPortal2.sol`:
```solidity
    function test_userNotAbleToFinalizeOrReprove() external {
        console2.log("start of POC");

        // @POC: Create a legit dispute game
        IDisputeGame legitGame = disputeGameFactory.create(
            optimismPortal2.respectedGameType(), Claim.wrap(_outputRoot), abi.encode(_proposedBlockNumber + 1)
        );

        // @POC: Use it in same block
        vm.prank(address(0xb0b));
        optimismPortal2.proveWithdrawalTransaction({
            _tx: _defaultTx,
            _disputeGameIndex: _proposedGameIndex + 1,
            _outputRootProof: _outputRootProof,
            _withdrawalProof: _withdrawalProof
        });

        // @POC: set game status to CHALLENGER_WINS.
        vm.mockCall(address(legitGame), abi.encodeCall(game.status, ()), abi.encode(GameStatus.DEFENDER_WINS));

        // @POC: Make proof finalizable
        vm.warp(block.timestamp + optimismPortal2.proofMaturityDelaySeconds() + 1 seconds);

        // @POC: User can't finalize
        vm.prank(address(0xb0b));
        vm.expectRevert("OptimismPortal: withdrawal timestamp less than dispute game creation timestamp");
        optimismPortal2.finalizeWithdrawalTransaction(_defaultTx);

        // @POC: User creates a new legit dispute game, to reprove the transaction
        IDisputeGame legitGame2 = disputeGameFactory.create(
            optimismPortal2.respectedGameType(), Claim.wrap(_outputRoot), abi.encode(_proposedBlockNumber + 2)
        );

        // @POC: set game status to CHALLENGER_WINS.
        vm.mockCall(address(legitGame2), abi.encodeCall(game.status, ()), abi.encode(GameStatus.DEFENDER_WINS));
        vm.warp(block.timestamp + optimismPortal2.proofMaturityDelaySeconds() + 1 seconds);

        // @POC: User can't use a new game for same transaction
        vm.prank(address(0xb0b));
        vm.expectRevert("OptimismPortal: withdrawal hash has already been proven, and the old dispute game is not invalid");
        optimismPortal2.proveWithdrawalTransaction({
            _tx: _defaultTx,
            _disputeGameIndex: _proposedGameIndex + 2,
            _outputRootProof: _outputRootProof,
            _withdrawalProof: _withdrawalProof
        });
    }
```


## Tool used

Manual Review

## Recommendation

This issue can be fixed through multiple ways. Here are two easy ways of fixing the issue:
- **Fix 1**: check that `FaultDisputeGame.createdAt() < block.timestamp` in `proveWithdrawalTransaction`.
- **Fix 2**: modify `provenWithdrawal.timestamp > createdAt` into `provenWithdrawal.timestamp => createdAt` in `checkWithdrawal`

The following patch implements **Fix 1**:

```diff
diff --git a/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol b/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol
index 8e61261..2f3b70c 100644
--- a/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol
+++ b/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol
@@ -254,9 +254,11 @@ contract OptimismPortal2 is Initializable, ResourceMetering, ISemver {
         require(_tx.target != address(this), "OptimismPortal: you cannot send messages to the portal contract");
 
         // Fetch the dispute game proxy from the `DisputeGameFactory` contract.
-        (GameType gameType,, IDisputeGame gameProxy) = disputeGameFactory.gameAtIndex(_disputeGameIndex);
+        (GameType gameType, Timestamp creationTime, IDisputeGame gameProxy) = disputeGameFactory.gameAtIndex(_disputeGameIndex);
         Claim outputRoot = gameProxy.rootClaim();
 
+        require(creationTime.raw() < block.timestamp, "OptimismPortal: invalid creation time");
+
         // The game type of the dispute game must be the respected game type.
         require(gameType.raw() == respectedGameType.raw(), "OptimismPortal: invalid game type");
 
```


The following patch implements **Fix 2**:

```diff
diff --git a/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol b/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol
index 8e61261..3c724be 100644
--- a/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol
+++ b/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol
@@ -476,7 +476,7 @@ contract OptimismPortal2 is Initializable, ResourceMetering, ISemver {
         // starting timestamp inside the Dispute Game. Not strictly necessary but extra layer of
         // safety against weird bugs in the proving step.
         require(
-            provenWithdrawal.timestamp > createdAt,
+            provenWithdrawal.timestamp => createdAt,
             "OptimismPortal: withdrawal timestamp less than dispute game creation timestamp"
         );
 
```

*Note: The provided patch can be applied with `git apply`.*