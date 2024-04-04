Noisy Pewter Woodpecker

medium

# Data overwrite bug in ClonesWithImmutableArgs could be abused to deploy invalid proxy

## Summary

The `ClonesWithImmutableArgs` library can be manipulated when data with an arbitrary length is passed to the `clone` function. This manipulation can lead to proxies with undefined behavior, which could risk the safety of the dispute game system.

## Vulnerability Detail

The basic idea of `ClonesWithImmutableArgs` is to deploy a proxy contract with a set of immutable arguments included in the code. When the proxy is called, it delegate calls to the implementation, and includes the immutable arguments in the call data. These values can be accessed by indexing into the calldata at the correct offsets.

In order to create this code, the `clone()` function stores the `extraLength` in multiple places in the code. This is the length of the arguments that the proxy is supposed to store.

In order to perform these additions, the hardcoded proxy code has gaps (`0000`) in the code. This is then `or`d with `runSize` (which is equal to `extraLength - 53`) and `extraLength` at the right offset to add the length into these gaps.

```solidity
mstore(
    ptr,
    or(
        hex"610000_3d_81_600a_3d_39_f3_36_3d_3d_37_3d_3d_3d_3d_610000_80_6035_36_39_36_01_3d_73",
        or(
            shl(0xe8, runSize),
            shl(0x58, extraLength)
        )
    )
)
```
However, there is no check that these values will in fact fit in the 2 bytes that are allocated to them. The result is that, if a longer length is passed, the data will overwrite the code to the left of the `extraLength` value with its extra digits.

This is important because by `or`ing additional bits with the preceding opcode, `0x61` (PUSH2) can be changed into any higher op code. For example, if it is changed into PUSH4, it will consume the following bytes (`3d81`) as the data to push, rather than performing the operations.

This gives an attacker a lot of power to determine the behavior of the proxy, skipping intended operations (like above) or jumping into data that is not intended to be executed (like the other PUSH values) in order to create undefined behavior.

## Proof of Concept

As a simple proof of concept, we can see above that if we turn `PUSH2` into `PUSH18`, it will skip all the way to the later `00` values. These values represent the `STOP` op code, and will leave to a contract being deployed with no bytecode.

```solidity
function testBreakClones() public {
    // If the initial PUSH2 becomes PUSH18, it eats next 18 bytes & leaves us with 00 for STOP (returning empty bytecode).
    // To turn PUSH2 into PUSH18, we need to add a value to the 16 slot (1 << 4).
    // Assuming we want 0s for the 2 bytes (16), then we need length == 1 << (16 + 4) == 1048576
    // The value entered into the slot is `length + 63 - 10`, so `targetLength = 1048576 - 53 = 1048523`
    // `cast 2h 1048523` = 0xfffcb
    bytes memory b = new bytes(0xfffcb);
    address proxy = ClonesWithImmutableArgs.clone(address(123), abi.encodePacked(uint(1), bytes32(0), b));

    // A proxy address is returned, but it has no code.
    assert(address(proxy) != address(0));
    assert(address(proxy).code.length == 0);
}
```
This is a simple example, but it shows the potential for abuse.

[Note that this isn't a risk in the current game due to the check that `extraData.length < 32`. However, since our focus is on the surrounding system and ensuring it is robust against incorrect games, this vulnerablity is important to address.]

## Impact

The `DisputeGameFactory` can deploy proxies with undefined behavior, breaking the assumptions of the other interacting contracts and potentially leading to risky behavior.

## Code Snippet

https://github.com/Saw-mon-and-Natalie/clones-with-immutable-args/blob/105efee1b9127ed7f6fedf139e1fc796ce8791f2/src/ClonesWithImmutableArgs.sol#L94-L103

## Tool used

Manual Review

## Recommendation

Use Solady's `LibClone`, which has addressed this issue thanks to the incredible find in their [Cantina audit](https://github.com/Vectorized/solady/blob/main/audits/cantina-solady-report.pdf) and is safe from this attack.