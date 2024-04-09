Tart Ultraviolet Kitten

medium

# M - The safety mechanism of the DelayedWETH contract can be bypassed

## Summary

A malicious user can frontrun directly the admin's call of  `hold()` function in DelayedWETH. They can reset back the approval to zero, so that the admin cannot actually confiscate their funds.

## Vulnerability Detail

DelayedWETH has several safety mechanisms in place to prevent malicious actors from cashing out illicit profits. The `hold()` function allows the owner to set arbitrary approval:
```solidity
function hold(address _guy, uint256 _wad) external {
    require(msg.sender == owner(), "DelayedWETH: not owner");
    allowance[_guy][msg.sender] = _wad;
    emit Approval(_guy, msg.sender, _wad);
}
```

The function only sets a custom approval for the admin. However it doesn't make use of that approval. A malicious `guy` who is supposed to lose access to their funds, can frontrun the call with `approve(admin, 0)`. At this point, the admin can no longer confiscate their funds. 

The only resolution for the admin is to bundle together the `hold()` and `transferFrom()` calls, but until they figure out how to do that, additional time was wasted, making it likely that the freezing time will have passed and the malicious user can call `withdraw()` to cash out their profits into ETH, which is unsanctionable.

## Impact

Bypass of a key security mechanism of the DelayedWETH contract

## Code Snippet

```solidity
function hold(address _guy, uint256 _wad) external {
    require(msg.sender == owner(), "DelayedWETH: not owner");
    allowance[_guy][msg.sender] = _wad;
    emit Approval(_guy, msg.sender, _wad);
}
```


## Tool used

Manual Review

## Recommendation

The `hold()` function should either:
- Disable the approve() functionality from the malicious user
- Actually seize the funds inside the `hold()` function.
