Hot Stone Peacock

medium

# Dispute games can be infinitely played when `MAX_GAME_DEPTH = 127`

## Summary

When `MAX_GAME_DEPTH = 125`, dispute games can be played infinitely (ie. `move()` can keep being called on parent claims, regardless of their depth).

## Vulnerability Detail

`Position` is a user-defined types that wraps a `uint128`:

[DisputeTypes.sol#L69-L74](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/libraries/DisputeTypes.sol#L69-L74)

```solidity
type Position is uint128;
```

This means that it can hold up to `2 ** 128 - 1` unique positions. As such, `MAX_GAME_DEPTH` can be assumed to have an upper bound (maximum value) of 127, since `Position` can only hold nodes in the game tree up to a depth of 127.

When calling `FaultDisputeGame.move()` to make a move from a parent position, the function checks that the depth of the next position does not exceed `MAX_GAME_DEPTH`:

[FaultDisputeGame.sol#L247-L251](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L247-L251)

```solidity
        // INVARIANT: A move can never surpass the `MAX_GAME_DEPTH`. The only option to counter a
        //            claim at this depth is to perform a single instruction step on-chain via
        //            the `step` function to prove that the state transition produces an unexpected
        //            post-state.
        if (nextPositionDepth > MAX_GAME_DEPTH) revert GameDepthExceeded();
```

However, doing this instead of checking that `parentPos >= MAX_GAME_DEPTH` does not work when `MAX_GAME_DEPTH == 127`. This is because `LibPosition.move()` does not account for the scenario where a position overflows `uint128`:

[LibPosition.sol#L174-L178](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/lib/LibPosition.sol#L174-L178)

```solidity
    function move(Position _position, bool _isAttack) internal pure returns (Position move_) {
        assembly {
            move_ := shl(1, or(iszero(_isAttack), _position))
        }
    }
```

As seen from above, if `_position` is `2 ** 127` or greater, the calculation of `nextPosition` will overflow in `LibPosition.move()`, causing it to wrap around and become a smaller number. For example:
- If `parentPos = 2 ** 127`, calling `attack()` would return `nextPosition = 0`.
- If `parentPos = 2 ** 127`, calling `defend()` would return `nextPosition = 2`.

Due to the overflow, depending on the value of `parentPos` at depth 127, making a move could return any arbitrary position in the game tree with a depth smaller than 127.

This becomes a problem as the `nextPositionDepth > MAX_GAME_DEPTH` check will not catch this. For example, when `move()` is called on a node where `parentPos = 2 ** 127`, `nextPosition = 0` and `nextPositionDepth = 1`, thereby passing the check. 

As such, the `nextPositonDepth > MAX_GAME_DEPTH` condition will never be true and the check will never be reached.

## Impact

When `MAX_GAME_DEPTH = 127`, the `nextPositonDepth > MAX_GAME_DEPTH` check will never be reached due to an overflow of `Position`. 

As such, `move()` can be called an infinite number of times and the game can be played infinitely. This will force honest challengers and proposers to keep on playing the game until their game clock runs out, otherwise, there will be an uncontested child claim in the subgame.

Note that this impact is not exhaustive - there may be more impacts depending on what `nextPosition` is calculated as when `Position` overflows.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/libraries/DisputeTypes.sol#L69-L74

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/lib/LibPosition.sol#L174-L178

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L247-L251

## Tool used

Manual Review

## Recommendation

Consider adding a check in the constructor to limit `MAX_GAME_DEPTH` to maximum of `125`.
