Sharp Pecan Ape

high

# Withdrawals may fail to finalize, permanently locking user funds

## Summary

Withdrawal transactions may fail to finalize because of the logical error in the `finalizeWithdrawalTransactionExternalProof` function. When external call fails, the transaction will be permanently marked as finalized, preventing the transaction from being finalized in the future. This leads to permanent locking of user funds in Optimism L1 contracts.

## Vulnerability Detail

Function `finalizeWithdrawalTransactionExternalProof` contains [the following snippet](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L356-L383) at the end (cited abridged): it first marks the transaction as finalized, then calls an external contract, and finally reverts if the call was not successful _and_ the transaction origin is `ESTIMATION_ADDRESS`:

```solidity
        // Mark the withdrawal as finalized so it can't be replayed.
        finalizedWithdrawals[withdrawalHash] = true;
        ...
        // Trigger the call to the target contract.
        bool success = SafeCall.callWithMinGas(_tx.target, _tx.gasLimit, _tx.value, _tx.data);
        ...
        // Reverting here is useful for determining the exact gas cost to successfully execute the
        // sub call to the target contract if the minimum gas limit specified by the user would not
        // be sufficient to execute the sub call.
        if (!success && tx.origin == Constants.ESTIMATION_ADDRESS) {
            revert("OptimismPortal: withdrawal failed");
        }
```

The problem is, when the external call fails, and the transaction origin _is not_ `ESTIMATION_ADDRESS`, the user transaction won't execute, but `finalizeWithdrawalTransactionExternalProof` won't revert and the transaction will be marked as finalized, thus preventing it from executing in the future.

External calls may fail due to various reasons, including but not limited to dynamic conditions inside target contracts, dynamic environment (e.g. dependencies on oracles/exchanges), underestimation of the gas cost, etc. Notice that it is not possible in many cases to either correctly statically predict the gas cost, or to estimate whether the external call will certainly succeed, because of the dynamic environment. Finally, as anyone (not only the users themselves) may call `finalizeWithdrawalTransactionExternalProof`, this may be exploited by attackers by calling the function in an unfavorable moment, when they know the external call will fail. Users should be given the opportunity to finalize the transaction any number of times.

## Impact

Withdrawal transactions may become unfinalizable, permanently locking user funds.

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L381-L383

## Tool used

Manual Review

## Recommendation

We recommend replacing `&&` with `||`, thus correctly reverting if either the external call failed, _or_ the transaction origin is `ESTIMATION_ADDRESS`. We believe this was the original developer's intent.

```diff
--- a/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol
+++ b/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol
@@ -378,7 +378,7 @@ contract OptimismPortal2 is Initializable, ResourceMetering, ISemver {
         // Reverting here is useful for determining the exact gas cost to successfully execute the
         // sub call to the target contract if the minimum gas limit specified by the user would not
         // be sufficient to execute the sub call.
-        if (!success && tx.origin == Constants.ESTIMATION_ADDRESS) {
+        if (!success || tx.origin == Constants.ESTIMATION_ADDRESS) {
             revert("OptimismPortal: withdrawal failed");
         }
     }
```
