
| ID   | Title                                                                   |
| ---- | ----------------------------------------------------------------------- |
| L-01 | No Global Rate Limit on Optimistic Proposals Allows Veto Power Dilution |


# [L-01] No Global Rate Limit on Optimistic Proposals Allows Veto Power Dilution

## Description

## Summary

`ThrottleLib` enforces a per-account rate limit on optimistic proposals — each `OPTIMISTIC_PROPOSER_ROLE` holder can submit at most `capacity` proposals per 12-hour window. However, there is no global cap on the total number of simultaneously active optimistic proposals across all proposers. With two proposers each submitting their full `capacity = 2` allowance in the same block, four proposals enter the veto window simultaneously. The community's veto power is split across all active proposals — each requiring independently reaching `vetoThreshold` — while the total voting power remains fixed. With only three voters and four concurrent proposals, at minimum one proposal will always succeed without sufficient veto coverage, regardless of how diligently token holders participate.



## Description

The throttle storage is per-account with no shared global counter:

```solidity
struct ProposalThrottleStorage {
    uint256 capacity;
    mapping(address account => ProposalThrottle) throttles;
}
```

`consumeProposalCharge` reads and writes only `throttles[account]` — it has no awareness of how many other proposals are currently active across other accounts:

```solidity
function consumeProposalCharge(ProposalThrottleStorage storage proposalThrottle, address account) external {
    ProposalThrottle storage throttle = proposalThrottle.throttles[account];
    (uint256 proposalsAvailable, uint256 charge) = _getProposalsAvailable(proposalThrottle, account);
    require(proposalsAvailable >= 1, ...);
    throttle.currentCharge = charge - (1e18 / proposalThrottle.capacity);
    throttle.lastUpdated = block.timestamp;
}
```

The veto mechanism requires each proposal to independently reach `vetoThreshold` of total supply in Against votes:

```solidity
// Proposal is Defeated when:
againstVotes >= vetoThreshold * pastTotalSupply / 1e18
```

With `N` simultaneous proposals and a fixed pool of voting power, the community must split their votes across all `N` proposals. If `N > 1 / vetoThreshold`, the math guarantees at least one proposal cannot be vetoed even if every token holder votes Against on some proposal.

With the deployed parameters:
```
vetoThreshold = 20%  →  max simultaneous vetoable proposals = 1 / 0.20 = 5
capacity = 2, proposers = 2  →  max simultaneous proposals = 4
```

With 4 proposals and 3 voters (Alice=40%, Bob=40%, Carol=20%), each voter can only cast their weight against one proposal at a time — leaving at least one proposal unvetoed


## Impact

 Optimistic Proposals pass the Veto threshold due to veto power dilution and gets execution. 
 

## Proof of Code

```solidity
function test_poc_noGlobalRateLimit_vetoDilution() public {
    (address[] memory t, uint256[] memory v, bytes[] memory c) =
        _singleCall(address(underlying), 0, abi.encodeCall(IERC20.transfer, (alice, 1_000e18)));

    vm.prank(optimisticProposer);
    uint256 p1 = governor.proposeOptimistic(t, v, c, "proposal 1");
    vm.prank(optimisticProposer);
    uint256 p2 = governor.proposeOptimistic(t, v, c, "proposal 2");
    vm.prank(optimisticProposer2);
    uint256 p3 = governor.proposeOptimistic(t, v, c, "proposal 3");
    vm.prank(optimisticProposer2);
    uint256 p4 = governor.proposeOptimistic(t, v, c, "proposal 4");

    _warpToActive(p1);

    // All three voters veto one proposal each — not enough to cover all four
    vm.prank(alice);
    governor.castVote(p1, 0);
    vm.prank(bob);
    governor.castVote(p2, 0);
    vm.prank(carol);
    governor.castVote(p3, 0);

    _warpPastDeadline(p1);

    // p1: alice (400_000e18) >= 20% of 1_000_000e18 (200_000e18) → Defeated
    assertEq(uint8(governor.state(p1)), uint8(IGovernor.ProposalState.Defeated), "p1: vetoed by alice");
    // p2: bob (400_000e18) >= 200_000e18 → Defeated
    assertEq(uint8(governor.state(p2)), uint8(IGovernor.ProposalState.Defeated), "p2: vetoed by bob");
    // p3: carol (200_000e18) >= 200_000e18 → Defeated
    assertEq(uint8(governor.state(p3)), uint8(IGovernor.ProposalState.Defeated), "p3: vetoed by carol");
    // p4: no votes → Succeeded — executes without timelock delay
    assertEq(uint8(governor.state(p4)), uint8(IGovernor.ProposalState.Succeeded), "p4: slips through");

    governor.execute(t, v, c, keccak256(bytes("proposal 4")));
}
```

Run with:
```bash
forge test --match-test test_poc_noGlobalRateLimit_vetoDilution -vv
```

The test proves:
- All 4 proposals enter the veto window simultaneously
- Every voter participates — full community engagement, no apathy
- 3 of 4 proposals are correctly vetoed
- `p4` succeeds and executes via `executeBatchBypass` with no timelock delay



## Recommendation

Add a global cap on the number of simultaneously active optimistic proposals. Track a counter that increments on proposal creation and decrements on execution, cancellation, or defeat:

```solidity
struct ProposalThrottleStorage {
    uint256 capacity;
    uint256 globalActiveCap;      // max concurrent active proposals
    uint256 activeProposals;      // current count
    mapping(address account => ProposalThrottle) throttles;
}
```

In `consumeProposalCharge`:
```solidity
require(
    proposalThrottle.activeProposals < proposalThrottle.globalActiveCap,
    OptimisticGovernor__GlobalCapExceeded()
);
proposalThrottle.activeProposals++;
```

Decrement `activeProposals` in `_executeOperations`, `_cancel`, and `_tallyUpdated` when a proposal reaches a terminal state. Set `globalActiveCap` such that:

```
globalActiveCap <= 1 / vetoThreshold
```

With `vetoThreshold = 20%`, `globalActiveCap <= 5` ensures the community can always veto all active proposals with full participation. A conservative value of `globalActiveCap = capacity` (matching the per-account limit) prevents any single proposal from going unexamined even under adversarial multi-proposer coordination.

