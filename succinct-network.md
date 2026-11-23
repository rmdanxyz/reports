
| ID   | Title                                                                                                   |
| ---- | ------------------------------------------------------------------------------------------------------- |
| L-01 | Stakers overslashed due to preventing them finish unstake as long as there a slash claim for the prover |


# [L-01]  Stakers overslashed due to preventing them finish unstake as long as there a slash claim for the prover


## Description


Stakers who have requested unstake gets overslashing if a slash request was made before they call `finishStake`  or a slash request was made in the last 7 days that force unstaker to wait more leaves a gap for a another slash request to be made despite they have passed the unstake period,  and that because reverting if there is a slash request for the prover they are stake to no matter when it was made.

```javascript
    function finishUnstake(address _staker) external override returns (uint256 PROVE) {
        // Get the prover that the staker is staked with.
        address prover = stakerToProver[_staker];
        if (prover == address(0)) revert NotStaked();

        // Get the unstake claims for this staker.
        UnstakeClaim[] storage claims = unstakeClaims[_staker];
        if (claims.length == 0) revert NoUnstakeRequests();

        // Check that this prover is not in the process of being slashed.
@>>     if (slashClaims[prover].length > 0) revert ProverHasSlashRequest();
        
        // Process the available unstake claims.
        PROVE += _finishUnstake(_staker, prover, claims);

        // If the staker has no remaining balance with this prover, remove the staker's delegate.
        // This allows them to choose a different prover if they stake again.
        if (balanceOf(_staker) == 0) {
            // Remove the staker's prover delegation.
            stakerToProver[_staker] = address(0);

            emit ProverUnbound(_staker, prover);
        }
    }
```


## Impact Explanation
High - users gets over-slashed and wait more than unstake period to get their funds back

## Likelihood Explanation
 Low - medium as slashing might be common action and might be there a good time-frame before users  finish their unstake that a slash request made, not that only but any slash happen in the last 7 days since user request unstake force user to wait more before finish stake.

## Proof of Concept
copy the test in a file named `SuccinctBug.t.sol` in tests and run this command:
```command
forge test test/SuccinctBug.t.sol -vvvv
```  

```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {SuccinctStakingTest} from "./SuccinctStaking.t.sol";
import {SuccinctStaking} from "../src/SuccinctStaking.sol";
import {IProverRegistry} from "../src/interfaces/IProverRegistry.sol";
import {ISuccinctStaking} from "../src/interfaces/ISuccinctStaking.sol";
import {IERC20} from "../lib/openzeppelin-contracts/contracts/interfaces/IERC20.sol";
import {ERC20Permit} from
    "../lib/openzeppelin-contracts/contracts/token/ERC20/extensions/ERC20Permit.sol";

import {IERC4626} from "../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";

contract SuccinctStakingStakeTests is SuccinctStakingTest {

    function test_overslash() public{

        uint256 stakeAmount = STAKER_PROVE_AMOUNT;

        // Initial state checks
        assertEq(IERC20(PROVE).balanceOf(STAKER_1), stakeAmount);
        assertEq(IERC20(I_PROVE).balanceOf(ALICE_PROVER), 0);
        assertEq(IERC20(ALICE_PROVER).balanceOf(STAKING), 0);
        assertEq(IERC20(PROVE).balanceOf(I_PROVE), 0);
        assertEq(SuccinctStaking(STAKING).proverStaked(ALICE_PROVER), 0);

        vm.prank(STAKER_1);
        IERC20(PROVE).approve(STAKING, stakeAmount);

        vm.expectEmit(true, true, true, true);
        emit ISuccinctStaking.ProverBound(STAKER_1, ALICE_PROVER);
        vm.expectEmit(true, true, true, true);
        emit ISuccinctStaking.Stake(STAKER_1, ALICE_PROVER, stakeAmount, stakeAmount, stakeAmount);

        // Stake to Alice prover.
        vm.prank(STAKER_1);
        uint256 stPROVE = SuccinctStaking(STAKING).stake(ALICE_PROVER, stakeAmount);
        assertEq(stPROVE, stakeAmount);

        // Check balances
        assertEq(SuccinctStaking(STAKING).stakedTo(STAKER_1), ALICE_PROVER);
        assertEq(SuccinctStaking(STAKING).balanceOf(STAKER_1), stakeAmount);
        assertEq(SuccinctStaking(STAKING).staked(STAKER_1), stakeAmount);
        assertEq(SuccinctStaking(STAKING).unstakePending(STAKER_1), 0);
        assertEq(SuccinctStaking(STAKING).previewUnstake(ALICE_PROVER, stakeAmount), stakeAmount);
        assertEq(SuccinctStaking(STAKING).proverStaked(ALICE_PROVER), stakeAmount);
        assertEq(IERC20(PROVE).balanceOf(STAKING), DISPENSE_AMOUNT);
        assertEq(IERC20(I_PROVE).balanceOf(ALICE_PROVER), stakeAmount);
        assertEq(IERC20(STAKING).balanceOf(STAKER_1), stakeAmount);
        assertEq(IERC20(ALICE_PROVER).balanceOf(STAKING), stakeAmount);
        assertEq(IERC20(PROVE).balanceOf(I_PROVE), stakeAmount);


        //staker one request unstake 
        _requestUnstake(STAKER_1, stakeAmount);

        skip(17 days);

        uint256 slashAmount = SuccinctStaking(STAKING).staked(STAKER_1);
        uint256 slashIndex = _requestSlash(ALICE_PROVER, slashAmount /3 );

        // Complete slash
        skip(SLASH_PERIOD);
        _finishSlash(ALICE_PROVER, slashIndex);

        // another slash request made before staker call unstake period
         _requestSlash(ALICE_PROVER, slashAmount / 3);

        // users try to unstake as he passed the unstake period
         vm.expectRevert();
        _finishUnstake(STAKER_1);

        // Complete slash
        skip(SLASH_PERIOD);
        _finishSlash(ALICE_PROVER, slashIndex);

        _finishUnstake(STAKER_1);

        assertEq(IERC20(PROVE).balanceOf(STAKER_1), stakeAmount - ((stakeAmount/3) + (stakeAmount/3)));


    }

}

```

## Recommendation
- When finishing unstake loop through slash claims and revert only if there is a slash request made in the unstake period of the users. 
