Fancy Magenta Toad

medium

# depositTransaction() does not check _value passed in is the same as msg.value

## Summary

`depositTransaction()` does not check `_value` passed in is the same as `msg.value`. User can set any number in `_value` but sending different amount of ETH while calling `depositTransaction()`. This enable user creates a `TransactionDeposited` event with `_value` while in reality sending lesser ETH to OptimismPortal2.sol.

## Vulnerability Detail

`_value` in function `depositTransaction()` is used to indicate ETH value to be send to recipient on L2. However this `_value` is not checked against `msg.value` to ensure OptimismPortal2.sol is receiving the same amount before emitting a `TransactionDeposited` event.

Any user can successfully call `depositTransaction()` with any number of `_value` but sending 0 ETH to the contract and still emit a `TransactionDeposited` event. Depending on how relaying node interpret `opaqueData` in `TransactionDeposited` event, this could potentially release more ETH on L2 to the recipient than it should.

```Solidity
    /// @notice Accepts deposits of ETH and data, and emits a TransactionDeposited event for use in
    ///         deriving deposit transactions. Note that if a deposit is made by a contract, its
    ///         address will be aliased when retrieved using `tx.origin` or `msg.sender`. Consider
    ///         using the CrossDomainMessenger contracts for a simpler developer experience.
    /// @param _to         Target address on L2.
    /// @param _value      ETH value to send to the recipient.
    /// @param _gasLimit   Amount of L2 gas to purchase by burning gas on L1.
    /// @param _isCreation Whether or not the transaction is a contract creation.
    /// @param _data       Data to trigger the recipient with.
    function depositTransaction(
        address _to,
        uint256 _value,
        uint64 _gasLimit,
        bool _isCreation,
        bytes memory _data
    )
        public
        payable
        metered(_gasLimit)
```

## Impact

Medium

## Code Snippet

https://github.com/sherlock-audit/2024-02-optimism-2024/blob/f216b0d3ad08c1a0ead557ea74691aaefd5fd489/optimism/packages/contracts-bedrock/src/L1/OptimismPortal2.sol#L397

## Tool used

Manual Review

## Recommendation

Although `opaqueData` does included both `msg.value` and `_value` in `TransactionDeposited` event to at least enables cross-checking during relaying, it is better to make sure `_value` is same as `msg.value` or remove `_value` and use only `msg.value` if this deposit is only intended for ETH deposit.

Recommended amendments as below:
```diff
    /// @notice Accepts deposits of ETH and data, and emits a TransactionDeposited event for use in
    ///         deriving deposit transactions. Note that if a deposit is made by a contract, its
    ///         address will be aliased when retrieved using `tx.origin` or `msg.sender`. Consider
    ///         using the CrossDomainMessenger contracts for a simpler developer experience.
+   /// @dev               Whatever amount msg.value specify is the ETH value to send to the recipient.
    /// @param _to         Target address on L2.
-   /// @param _value      ETH value to send to the recipient.
    /// @param _gasLimit   Amount of L2 gas to purchase by burning gas on L1.
    /// @param _isCreation Whether or not the transaction is a contract creation.
    /// @param _data       Data to trigger the recipient with.
    function depositTransaction(
        address _to,
-       uint256 _value,
        uint64 _gasLimit,
        bool _isCreation,
        bytes memory _data
    )
        public
        payable
        metered(_gasLimit)
    {
        // Just to be safe, make sure that people specify address(0) as the target when doing
        // contract creations.
        if (_isCreation) {
            require(_to == address(0), "OptimismPortal: must send to address(0) when creating a contract");
        }

        // Prevent depositing transactions that have too small of a gas limit. Users should pay
        // more for more resource usage.
        require(_gasLimit >= minimumGasLimit(uint64(_data.length)), "OptimismPortal: gas limit too small");

        // Prevent the creation of deposit transactions that have too much calldata. This gives an
        // upper limit on the size of unsafe blocks over the p2p network. 120kb is chosen to ensure
        // that the transaction can fit into the p2p network policy of 128kb even though deposit
        // transactions are not gossipped over the p2p network.
        require(_data.length <= 120_000, "OptimismPortal: data too large");

        // Transform the from-address to its alias if the caller is a contract.
        address from = msg.sender;
        if (msg.sender != tx.origin) {
            from = AddressAliasHelper.applyL1ToL2Alias(msg.sender);
        }

        // Compute the opaque data that will be emitted as part of the TransactionDeposited event.
        // We use opaque data so that we can update the TransactionDeposited event in the future
        // without breaking the current interface.
+       bytes memory opaqueData = abi.encodePacked(msg.value, _gasLimit, _isCreation, _data);
-       bytes memory opaqueData = abi.encodePacked(msg.value, _value, _gasLimit, _isCreation, _data);

        // Emit a TransactionDeposited event so that the rollup node can derive a deposit
        // transaction for this deposit.
        emit TransactionDeposited(from, _to, DEPOSIT_VERSION, opaqueData);
    }
```
Function that is depending on `depositTransaction()` like `receive()` should amend its arguments accordingly too.