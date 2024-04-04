Hot Stone Peacock

medium

# `uint64` is too small to hold `indexAtDepth` for nodes below a depth of 64

## Summary

As `uint64` is used to represent the index of nodes, calling `step()` with nodes below a depth of 64 may return an incorrect result.

## Vulnerability Detail

`LibPosition.indexAtDepth()` returns `indexAtDepth_` as a `uint64`:

```solidity
    function indexAtDepth(Position _position) internal pure returns (uint64 indexAtDepth_) {
        // Return bits p_{msb-1}...p_{0}. This effectively pulls the 2^{depth} out of the gindex,
        // leaving only the `indexAtDepth`.
        uint256 msb = depth(_position);
        assembly {
            indexAtDepth_ := sub(_position, shl(msb, 1))
        }
    }
```

However, `uint64` is too small to contain the index of nodes at a depth of 64 and below.

## Impact

`indexAtDepth()` is [used in `step()`](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L177) to find the depth of nodes. As such, calling `step()` on nodes at a depth of 64 and below may result in an incorrect result.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L177

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L825

## Tool used

Manual Review

## Recommendation

In `indexAtDepth()`, return `indexAtDepth_` as a `uint128` instead.