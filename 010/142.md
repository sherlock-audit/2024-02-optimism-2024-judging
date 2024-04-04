Refined Juniper Copperhead

high

# Unauthorized parties can claim the credit intended for the designated recipient

## Summary
See below...
## Vulnerability Detail
The vulnerability lies within the `claimCredit` function of the contract, where there is an absence of an authorization check to ensure that only the designated recipient (as specified by the `_recipient` parameter) can call the function. This is evident in the code snippet provided:
```solidity
 /// @notice Claim the credit belonging to the recipient address.
    /// @param _recipient The owner and recipient of the credit.
    function claimCredit(address _recipient) external { // @audit-info access control
        // Remove the credit from the recipient prior to performing the external call.
        uint256 recipientCredit = credit[_recipient];
        credit[_recipient] = 0;

        // Revert if the recipient has no credit to claim.
        if (recipientCredit == 0) {
            revert NoCreditToClaim();
        }

        // Try to withdraw the WETH amount so it can be used here.
        WETH.withdraw(_recipient, recipientCredit);// @audit-info CEI

        // Transfer the credit to the recipient.
        (bool success,) = _recipient.call{ value: recipientCredit }(hex"");
        if (!success) revert BondTransferFailed();
    }
```
This function is intended to allow the recipient specified during contract initialization to claim a certain amount of credit. However, without an explicit authorization check, the function can be called by any address, irrespective of whether it matches the `_recipient` address or not. This oversight compromises the intended functionality and security of the contract.
## Impact
The impact of this vulnerability is that unauthorized parties can claim the credit intended for the designated recipient. This could lead to unauthorized transfers of credit and potential misuse of funds.

## Code Snippet
[FaultDisputeGame.sol#L630-L646](https://github.com/sherlock-audit/2024-02-optimism-2024/blob/main/optimism/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol#L630-L646)
## Tool used

Manual Review

## Recommendation
Ensure that only the designated recipient can claim the credit, it is imperative to include an authorization check in the `claimCredit` function. Adding the following require statement before the credit transfer will mitigate this vulnerability:
```solidity
require(msg.sender == _recipient, "Caller not the recipient");
```