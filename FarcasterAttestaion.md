
| ID   | Title                                                                                                   |
| ---- | ------------------------------------------------------------------------------------------------------- |
| L-01 | users can challange a removal or addition of a Faracster wallet verifiaction anytime even after one day |


# [L-01] users can challange a removal or addition of a Faracster wallet verifiaction anytime even after one day


## Description



incorrect logic implementation of the challenge period (one day) isn't forced when users challenges a removal or addition of a Farcaster wallet Verification leads to inconsistency of the system.


```javascript
  function challengeAdd(
        uint256 fid,
        address verifyAddress,
        bytes32 publicKey,
        bytes memory signature
    ) public {
        bytes32 h = hash(
            MessageType.MESSAGE_TYPE_VERIFICATION_ADD_ETH_ADDRESS,
            fid,
            verifyAddress,
            publicKey,
            signature
        );

        if (verificationTimestamp[h] > 0) {
            try
                onchainVerifier.verifyAdd( fid, verifyAddress, publicKey, signature) returns (bool verified) 
                {
                if (verified) {
                    revert ChallengeFailed();
                }
            } catch {}

            verificationTimestamp[h] = 0;

            {
                (bool success, ) = payable(msg.sender).call{
                    value: challengeReward()
                }("");
                require(success);
            }

            emit Challenged(
                MessageType.MESSAGE_TYPE_VERIFICATION_ADD_ETH_ADDRESS,
                fid,
                verifyAddress,
                publicKey,
                h,
                signature
            );
        }
    }
```

## Proof of Concept

Add this to `FarcasterWalletVerifier.ts` and run `npx hardhat test test/FarcasterWalletVerifier.ts`


```javascript
    it("challenging Addition of Farcaster wallet verifiaction after one day", async function () {
      const [ wallet1 ] = await hre.viem.getWalletClients()
      const { walletOptimisticVerifier, publicKeyVerifier } = await loadFixture(deployFixture);
      const { fid, alice, message, messageBytes } =
        await signVerificationAddAddress();

      await publicKeyVerifier.write.addKey([ fid, toHexString(message.signer) ])

      const encodedData = encodeAbiParameters(
        parseAbiParameters("bytes32 r, bytes32 s, bytes message"),
        [
          toHexString(message.signature.subarray(0, 32)),
          toHexString(randomBytes(32)),
          toHexString(messageBytes),
        ]
      );

      await walletOptimisticVerifier.write.submitVerification([
        MessageType.VERIFICATION_ADD_ETH_ADDRESS,
        fid,
        alice.address,
        toHexString(message.signer),
        encodedData,
      ]);

      await time.increase(86400*2); // 2 days

       await walletOptimisticVerifier.write.challengeAdd([
        fid,
        alice.address,
        toHexString(message.signer),
        encodedData,
      ]);

   });

```

## Recommendation
- add this require at `challengeAdd()` and `challengeRemove()`
```diff
+  require( block.timestamp <= verificationTimestamp[h] + challengingPeriod, "challenge period is over")
```
