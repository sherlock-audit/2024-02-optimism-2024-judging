Vast Currant Parrot

medium

# claimCredit() user may not be able to claim bond

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