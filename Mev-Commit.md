
| ID   | Title                                                                                                                      |
| ---- | -------------------------------------------------------------------------------------------------------------------------- |
| H-01 | DOS standard-bridge by setting `_recipient` a contract that doesn't accept ETH.                                            |
| L-01 | Providers who staked then unstaked and have withdraw their ETH won't be able to register with a different BLSPubkey again. |
| L-02 | Potentially Stuck ETH From Pausing                                                                                         |
| I-01 | provider who store unopened commitment without open these commitment won't get slashed                                     |

# [H-01]  DOS standard-bridge by setting `_recipient` a contract that doesn't accept ETH. 

## Description
when a user initiate a transfer index of gateway is increased by one, and when relay finalizing a transaction it checks if this index is equal to the finalizing index on the gateway. so, if a user `initiateTransfer` while setting `_recipient` a contract that doesn't have a way to accept ether it will increase the index on gateway, and this transaction won't be finalized. when a new user came to `initiateTransfer` it will increase the index also, so when relay finlalizing transaction it willl always revert due to check of gateways indexs.

## Proof of Concept
we take an approch form `SettlementGateway` -> `L1Gateway` 
1. an attacker initate a transfer on L2 setting `_recipient` a deployed contract on mainnet that didn't implement any way to receive Eth. this will set `transferInitiatedIdx` = 1
2. when relay call `finalizeTransfer` won't be finalized due to the contract.
3. a noraml user fire the same transcation it will increase `transferInitiatedIdx` = 2 
4. when relay call `finalizetransfer` it will always fail due to this check. because `_counterpartyIdx` is 2 and `transferFinalizedIdx` is 1.
```javascript
  require(_counterpartyIdx == transferFinalizedIdx, InvalidCounterpartyIndex(_counterpartyIdx, transferFinalizedIdx));
```

## Recommendation
to prevent this dos check if `_recipient` is a smart contract and revert if it was by require its code length is zero. add this line to `initiateTransfer` function 

```diff
+    require(_recipient.code.length == 0);                
```


# [L-01] Providers who staked then unstaked and have withdraw their ETH won't be able to register with a different BLSPubkey again.

## Description
when register a provider by calling `_registerAndStake` function we are flagging the provider in a `providerRegistered` mapping with `true`, but when we unstake and withdraw later we don't update that mapping in any of these functions. so, if a provider want to register to a different BlsPubKey again after withdraw his ETH he wouldn't be able due to the require in `_registerAndStake` function. the only thing he can do is to stake to the old blspubkey by calling `stake` function.

```javascript
    require(!providerRegistered[provider], ProviderAlreadyRegistered(provider));
```

## Recommendation

update `providerRegistered` mapping in the `withdraw` function by adding this line:

```diff
        uint256 providerStake = providerStakes[msg.sender];
        providerStakes[msg.sender] = 0;
        withdrawalRequests[msg.sender] = 0;
+       providerRegistered[provider] = false;
        require(providerStake != 0, ProviderStakedAmountZero(msg.sender));
        require(preconfManager != address(0), PreconfManagerNotSet()); 
```




# [L-02] Potentially Stuck ETH From Pausing

## Description
An issue arises when the `SettlementGateway` is paused while the `L1Gatewaty` remains unpaused and usable. Users may still use the gateway to `initiateTransfer` from L1 to L2, and while the transaction succeed on L1, it will fail during finalization on L2 via `finalizeTransfer`. Therefore, the users who deposited L1 ETH will have their ETH locked in the gateway and unable to replay their transaction, or obtain a refund.

## impact
loss of funds
## Recommendation

add function that users can call to refund their ETH



# [L-03] provider who store unopened commitment without open these commitment won't get slashed 

## Description

providers can consume all bids have been issued to them sending bidders commitments and store on chain without opening them latter, then choose commitments which have higher bids to open to obtain more rewards than other providers and enforce bidders to resend their bids again and open them.

despite docs stated that unopened commitments gets their providers slashed. `oracle.sol` process commitment only for opened commitments.
https://docs.primev.xyz/concepts/rewards-and-slashing#unopened-commitment.


## Recommendation
in oracle contract, check if commitment is unopened and slash the 1 ETH fixed amount from provider.


