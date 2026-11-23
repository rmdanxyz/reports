
| ID   | Title                                                                               |
| ---- | ----------------------------------------------------------------------------------- |
| H-01 | users might mint themself 2X tokens what they deserve                               |
| M-01 | Non-Whitelisted addresses cannot contribute as a result of wrong control Flow.      |
| M-02 | Wrong Calculations of `amountForLP` and `amountForTreasury` and Temporary Stuck ETH |
| I-01 | Extend fund expiry will always fail                                                 |


# [H-01] users might mint themself 2X tokens what they deserve 


## Finding Description

when refunding address doesn't get deleted from `contributors` array. in the scenario where Fundrasing isn't reached and deadline is over and owner or Admin decided to extend Fundraising Deadline, during this time users can refund their ETH and when time extend they came back to contribute again this will push clone of their Address again to the `contributors` Array, when Fundrasing is Finalized they will be minted twice the deserve.

here's the refund() function, we just setting contributions to be zero and Don't delete 
```javascript
    function refund() external nonReentrant {

        require(!goalReached, "Fundraising goal was reached");

        require( block.timestamp > fundraisingDeadline, "Deadline not reached yet");

        require(contributions[msg.sender] > 0, "No contributions to refund");

        uint256 contributedAmount = contributions[msg.sender];

        contributions[msg.sender] = 0;

        payable(msg.sender).transfer(contributedAmount);

        emit Refund(msg.sender, contributedAmount);
    }
```

And when Fundraising is Finalized, looping through array of contributors to mint users twice, when they have contributed once. 

```javascript
   for (uint256 i = 0; i < contributors.length; i++) {
            address contributor = contributors[i];
            uint256 contribution = contributions[contributor];
            uint256 tokensToMint = (contribution * SUPPLY_TO_FUNDRAISERS) / totalRaised;

            emit MintDetails(contributor, tokensToMint);

            token.mint(contributor, tokensToMint);
        }
```

## impact
users gets 2X token at least

## Recommendation
store the index of address and delete it from `contributors` array when users refund.



# [M-01] Non-Whitelisted addresses cannot contribute as a result of wrong control Flow. 



## Description
We have two Kind of Addresses who can contribute `Whitelisted` with its kinds (Gold, silver, Platinum) and Non-whitelisted. Non-whitelisted should be able to contribute when `maxWhitelistAmount=<0`, Since enum are just `uint8` in the background, so the default value is `None`. `contribute()` will always revert and Non-whitelisted won't be able to contribute. 

in `contribute()` we always revert is users is not whitelisted 
```javascript
        WhitelistTier userTier = userTiers[msg.sender];
        require(userTier != WhitelistTier.None, "Not whitelisted");
```


## Impact 
where non-whitelisted should be able to contribute they won't be able to and this a large set of contributors.

## Recommendation
check if the msg.sender is whitelisted or not instead of revert when it's not whitelisted then do the require checks.


# [M-02 ]Wrong Calculations of `amountForLP` and `amountForTreasury` and Temporary Stuck ETH 

## Finding Description

There is  A Fallback function to make contributions simply by sending ETH to the contract, while there's a check for determine if they can't contribute or not it doesn't revert when fundraising is finalized, so the contract receives ETH anyways, if This ETH is sent when Goal is reached and fundraising ain't finalized yet, ETH will be Added to `totalCollected` which will cause a wrong calculation of `amountForLP` and `amountForTreseary` and user who sent this ETH will not get any minted tokens since he didn't contribute while he think he does. And if this ETH is sent After fundraising is Finalized, users will be not able to get their ETH back until admin gets it by calling `emergencyEscape()` and distribute for users again.

```javascript
   receive() external payable {
        if (!goalReached && block.timestamp < fundraisingDeadline) {
            contribute();
        }
    }
```

## Impact 
-  Wrong calculations of ETH sent to LP and Treasury.
- Temporary Stuck ETH of users.


## Recommendation

-  revert if Goal is reached in `receive` Function otherwise call `contribute()`
- check if fundraising is finalized or not
 
 ```diff

-   receive() external payable {
-        if (!goalReached && block.timestamp < fundraisingDeadline) {
-            contribute();
-        }
-    }

+   receive() external payable {
+        if (!goalReached && block.timestamp < fundraisingDeadline && !fundraisingFinalized) {
+            contribute();
+        }else{
+               revert();
+        }
+   }
```

# [i-0] Extend fund expiry will always fail


## Description


`fundExpiry` is passed to the `liquidityLocker` while deployment, since it is might be important to change the `fundExpiry` and there an setter function for. it was implemented wrong because of an external call. two times when we can change `fundExpiry` the first time before deployment of `liquidityLocker` and this will fail because address is 0x0000000000000000000000000000000000000000, the second time after deployment of `liquidityLocker` this also will always fail because there's check where `msg.sender` = `_daotreasury` and it will be called by `Daao`
`_daoTreasur=_daoManager = owner() of Daao` also but it ain't the same as `Daao`

```javascript
    function extendFundExpiry(uint256 newFundExpiry) external onlyOwner {
        require(newFundExpiry > fundExpiry, "Must choose later fund expiry");
        fundExpiry = newFundExpiry;
>>      ILocker(liquidityLocker).extendFundExpiry(newFundExpiry); // this call a zero address 
}
```

here's the contract of `LpLocker`, `extendFundExpiry` at line 189
https://explorer.mode.network/address/0xaEDEDdDC448AEE5237f6b3f11Ec370aB5793A0d3?tab=contract 
got this from the `DeploDaao.sol`
 
the snippent from the link above
```javascript
   function extendFundExpiry(uint256 newFundExpiry) public {
        require(msg.sender == _daoTreasury, "Must be called by dao treasury contract");
        require(newFundExpiry > fundExpiry, "Must be later fundexpiry");
        fundExpiry = newFundExpiry;
    }
```

## Impact 
the call to change `fundExpiry` will revert and owners can not change FundExpiry



## Recommendation
- no need for this external call since deployment haven't done yet and `LpLocker` already has function to extend fund Expiry 

```diff
- ILocker(liquidityLocker).extendFundExpiry(newFundExpiry);
```



