Real Olive Locust

medium

# Move can be DOSed / frontrun

## Summary

Move can be DOSed / frontrun

## Vulnerability Detail

Part One:

when make a move, first we move:

```solidity
Position nextPosition = parentPos.move(_isAttack);
uint256 nextPositionDepth = nextPosition.depth();
```

then we validate the msg.value fee:

```solidity
// INVARIANT: The `msg.value` must exactly equal the required bond.
if (getRequiredBond(nextPosition) != msg.value) revert IncorrectBondAmount();
```

the problem is that in each position, the getRequiredBond could be differect, and this too strict can leads to DOS

suppose user A wants to make a move for claim data at index 0,

user B frontrun user A's move and make a move first, the user A's move will revert because of the too strict msg.value check

Part B:

also note the check

```solidity
// INVARIANT: There cannot be multiple identical claims with identical moves on the same challengeIndex. Multiple
//            claims at the same position may dispute the same challengeIndex. However, they must have different
//            values.
ClaimHash claimHash = _claim.hashClaimPos(nextPosition, _challengeIndex);
if (claims[claimHash]) revert ClaimAlreadyExists();
claims[claimHash] = true;
```

at specific position and challange index, if user make a move, the same position + challange index slot is taken

another user make the same position + challange index transaction will revert

then this create a chance for frontrun, 

1. a user A can frontrun other people's move at challange index with invalid _claim data if they wants to block user from making the move

2. a user A can decode other valid move from User B (decode the claim data) and send higher gas via private RPC 

to make sure their move with valid claim data copied from user B get included in the block first, 
then make the original move transaction revert.

because only the The left-most correct counter is preferred in bond payouts, the user B's valid claim data still gets no payout, and the payout goes to frontruner user A

in the second case

## Impact

Frontrun / DOS

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L260

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L292

## Tool used

Manual Review

## Recommendation

N/A