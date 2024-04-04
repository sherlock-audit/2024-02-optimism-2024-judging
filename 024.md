Flat Fleece Snail

medium

# Bonds are overcharged

## Summary

Bonds are overcharged

## Vulnerability Detail
To determine the bond to be paid for executing a `move()` function, a formula is used to determine the bond cost and this can be seen in:

[FaultDisputeGame.sol#L585-L626](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L585-L626)
```solidity
    /// @notice Returns the required bond for a given move kind.
    /// @param _position The position of the bonded interaction.
    /// @return requiredBond_ The required ETH bond for the given move, in wei.
    function getRequiredBond(Position _position) public view returns (uint256 requiredBond_) {
        uint256 depth = uint256(_position.depth());
        if (depth > MAX_GAME_DEPTH) revert GameDepthExceeded();

        // Values taken from Big Bonds v1.5 (TM) spec.
        uint256 assumedBaseFee = 200 gwei;
        uint256 baseGasCharged = 400_000;
        uint256 highGasCharged = 200_000_000;

        // Goal here is to compute the fixed multiplier that will be applied to the base gas
        // charged to get the required gas amount for the given depth. We apply this multiplier
        // some `n` times where `n` is the depth of the position. We are looking for some number
        // that, when multiplied by itself `MAX_GAME_DEPTH` times and then multiplied by the base
        // gas charged, will give us the maximum gas that we want to charge.
        // We want to solve for (highGasCharged/baseGasCharged) ** (1/MAX_GAME_DEPTH).
        // We know that a ** (b/c) is equal to e ** (ln(a) * (b/c)).
        // We can compute e ** (ln(a) * (b/c)) quite easily with FixedPointMathLib.

        // Set up a, b, and c.
        uint256 a = highGasCharged / baseGasCharged;
        uint256 b = FixedPointMathLib.WAD;
        uint256 c = MAX_GAME_DEPTH * FixedPointMathLib.WAD;

        // Compute ln(a).
        // slither-disable-next-line divide-before-multiply
        uint256 lnA = uint256(FixedPointMathLib.lnWad(int256(a * FixedPointMathLib.WAD)));

        // Computes (b / c) with full precision using WAD = 1e18.
        uint256 bOverC = FixedPointMathLib.divWad(b, c);

        // Compute e ** (ln(a) * (b/c))
        // sMulWad can be used here since WAD = 1e18 maintains the same precision.
        uint256 numerator = FixedPointMathLib.mulWad(lnA, bOverC);
        int256 base = FixedPointMathLib.expWad(int256(numerator));

        // Compute the required gas amount.
        int256 rawGas = FixedPointMathLib.powWad(base, int256(depth * FixedPointMathLib.WAD));
        uint256 requiredGas = FixedPointMathLib.mulWad(baseGasCharged, uint256(rawGas));

        // Compute the required bond.
        requiredBond_ = assumedBaseFee * requiredGas;
    }
```

And it is used via the following manner:

[FaultDisputeGame.sol#L226-L260](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L226-L260)
```solidity
    function move(uint256 _challengeIndex, Claim _claim, bool _isAttack) public payable virtual {
        ...
        // Compute the position that the claim commits to. Because the parent's position is already
        // known, we can compute the next position by moving left or right depending on whether
        // or not the move is an attack or defense.
        Position parentPos = parent.position;
        Position nextPosition = parentPos.move(_isAttack);
        uint256 nextPositionDepth = nextPosition.depth();
        ...
        // INVARIANT: The `msg.value` must exactly equal the required bond.
        if (getRequiredBond(nextPosition) != msg.value) revert IncorrectBondAmount();
        ...
```

A high level overview of the above is that as follows:

Firstly, we consider each `move` action to cost some "gas", and the price of this gas is 200 gwei
- We want a minimum gas of 400_000 at the first depth
- We want a maximum gas of 200_000_000  at the max game depth
- On every move, we multiply the gas used of the previous move by a fixed constant.

The following is the derivation by Optimism, the formula to calculate the gas of $D$ the depth of the next position being moved to and $M$ being the max game depth.

$$400000*500^\frac{D}{M}$$

However, this seems to be unintended, considering that we want a minimum gas of 400_000 when we move from the root position to the next position where $D=1$, see [docs](https://specs.optimism.io/experimental/fault-proof/stage-one/bond-incentives.html#:~:text=FPM%20bonds%20are%20priced%20in%20the%20amount%20of%20gas%20that%20they%20are%20intended%20to%20cover.%20Bonds%20start%20at%20the%20very%20first%20depth%20of%20the%20game%20at%20a%20baseline%20of%20400_000%20gas.).  

> FPM bonds are priced in the amount of gas that they are intended to cover. Bonds start at the very first depth of the game at a baseline of 400_000 gas.

Suppose $M=8$, subbing into the formula we get a gas used of:

$$400000*500^\frac{1}{8}=869823$$

Which is not intended because recall we want a minimum gas of 400_000 cost when we move from the root position to the next position below the root position where $D=1$.

On subsequent levels, the gas is overcharged as well.

A PoC test case:
```solidity
    function test_bondOvercharging() public {
        Position ROOT_POSITION = Position.wrap(1);
        Position DEPTH_1_POSITION = ROOT_POSITION.move(true);
        assertEq(gameProxy.maxGameDepth(), 8);

        // By right, the bond cost of moving from root position to depth 1 position should be 400_000 * 200 gwei.
        assertEq(gameProxy.getRequiredBond(DEPTH_1_POSITION), 869_823 * 200 gwei);
    }
```

## Impact

Bonds are overcharged.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L585-L626

## Tool used

Quick Maths

## Recommendation

To resolve this, we want to deduct 1 from both the depth and the max depth in the calculation:

$$400000*500^\frac{D-1}{M-1}$$

