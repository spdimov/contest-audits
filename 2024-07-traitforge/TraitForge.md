
# Nuke transactions can be frontrunned and result in significantly lower reward for users

## Impact
Users who nuke their entity expect a reward based on the entity's nuke factor and the nuke fund's balance. However, due to the absence of a minimum reward mechanism, their transactions can be front-run by malicious actors, resulting in significantly lower rewards than anticipated. This issue arises not only from malicious behavior but also from the natural variability in transaction ordering on the blockchain, which can finalize transactions in a different order than they were sent.

## Proof of Concept
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/NukeFund/NukeFund.sol#L153

Consider this scenario:

User A and User B each have an entity with a nuke factor that entitles them to 50% of the nuke fund. The nuke fund balance is 5 ether.

User A submits a nuke transaction, expecting to receive 2.5 ether as a reward. However, User B sees this transaction and submits their own nuke transaction with a higher gas price, which gets processed first.

As a result:

- User B's transaction is processed first, and they receive 2.5 ether.
- User A's transaction is processed next, but now the fund only has 2.5 ether left. Thus, User A receives 1.25 ether.

This results in User A receiving 1.25 ether less than expected.
The losses for users can become even more significant as the reward pot increases.


## Tools Used

Manual review.

## Recommended Mitigation Steps

Consider implementing a minimum reward mechanism. In the example I have added a 2% accepted slippage.

```
function nuke(uint256 tokenId, uint256 expectedClaimAmount) public whenNotPaused nonReentrant { 
    require(
      nftContract.isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    require(
      nftContract.getApproved(tokenId) == address(this) ||
        nftContract.isApprovedForAll(msg.sender, address(this)),
      'Contract must be approved to transfer the NFT.'
    );
    require(canTokenBeNuked(tokenId), 'Token is not mature yet');

    uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
    uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
    uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

    // Directly assign the value to claimAmount based on the condition, removing the redeclaration
    uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam
      ? maxAllowedClaimAmount
      : potentialClaimAmount;
+   if (claimAmount < (expectedClaimAmount * 2) 
+ / 100) {
+     revert(); 
+   }

    fund -= claimAmount; // Deduct the claim amount from the fund

    nftContract.burn(tokenId); // Burn the token
    (bool success, ) = payable(msg.sender).call{ value: claimAmount }('');
    require(success, 'Failed to send Ether');

    emit Nuked(msg.sender, tokenId, claimAmount); // Emit the event with the actual claim amount
    emit FundBalanceUpdated(fund); // Update the fund balance
  }
```

# Users can deliberately revert transactions if the entropy they get is not good enough for them
## Impact
Due to atomic blockchain transactions, users can choose to revert their transaction if the entropy of the entity they get is not. This way malicious users can take advantage of honest players which pay the whole fee to get an entity. With current gas prices of `3 gwei` for L1 mainnet and gas used to mint new token  `~ 491048`, it will cost roughly` ~ 0.001473 ETH` which is lower than the first entity price (`0.005 ETH`) and much lower than the final price of `0.25 ETH`. This provide an option for users to try minting a new entity many times in order to get better entropy for relatively small amount of ETH. Users are not guaranteed to get the entropy they want, but they can "throw away" entities with bad entropy and just pay the gas fees.

## Proof of Concept
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L101
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L181
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L280

Consider the following scenario:
Alice wants to mint a forger entity.
We are in middle of generation count, prices for minting new entity are around `0.12 ETH`.
Instead of minting and hoping to get a forger, `Alice` can use the following contract

```
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.20;

import {TraitForgeNft} from "contracts/TraitForgeNft/TraitForgeNft.sol";

contract EntropyManipulator {
    TraitForgeNft private nftContract;

    constructor(TraitForgeNft _nftContract) payable {
        nftContract = _nftContract;
    }

    function mintForger() external {
        bytes32[] memory proof;
        nftContract.mintToken{value: 0.5 ether}(proof);
        uint256 entropy = nftContract.tokenEntropy(nftContract.totalSupply());
        bool isForger = (entropy % 3) == 0;

        if (!isForger) {
            revert();
        }
    }

    receive() external payable {}
}
```

Alice can repeatedly call `mintForger()` until she gets a successful transaction, ultimately paying `mintPrice + (gasTransactionCosts * timesTried)`. Given that `gasTransactionCosts * timesTried` is relatively small compared to the `mintPrice` and considering that an entity with "good" entropy might be worth significantly more than one with "bad" entropy, Alice can exploit this situation. While the protocol itself does not incur any losses, this strategy indirectly causes honest players to suffer losses.

Although the example uses the desire for a forger entity, this manipulation can apply to various factors, such as waiting for a better nuke factor or performance factor. Since blockchain data is public and etropies are consequently used, users can monitor which entropy slots contain more valuable entropies and exploit this strategy when those slots are about to be used.
## Tools Used

Manual review.
## Recommended Mitigation Steps

To prevent users from exploiting the predictability of entropy slots in the minting process, try to implement measures that enhance the randomness of the entropy used.
Possible solutions:
* **Verifiable Random Functions (VRFs)**: Implement VRFs like Chainlink VRF to generate truly random numbers on-chain. This makes it impossible for users to know the entropy of the entity they mint in the same transaction.
* **Delay Entropy Revelation**: Introduce a delay between the minting request and the revelation of the entropy. This can be achieved by committing to a random seed at the time of the transaction and revealing the actual entropy after a certain number of blocks have passed. This way users have to submit a mint request which commits to a future block hash or a similar unpredictable value as the source of randomness.
# mintWithBudget() in TraitForgeNft.sol wont work after first generation

## Impact
Users wont be able to use mintWithBudget() functionality after all 10 000 entities of first generation are minted.

## Proof of Concept
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L215
`_tokenIds < maxTokensPerGen` will always be `false` after all 10 000 generation 1 entities are minted.
`_tokenIds` is increased consequently with every mint of new entity and `maxTokensPerGen` is set to 10 000. After first generation, `_tokenIds` would always be > 10 000 thus making this function unusable.

```
...
while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
      _mintInternal(msg.sender, mintPrice);
      amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
...
```

## Tools Used
Manual review.
## Recommended Mitigation Steps

Use the following implementation which takes in consideration the current generation.
```
- while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen) {
+ while (budgetLeft >= mintPrice && _tokenIds < maxTokensPerGen * currentGeneration) {
      _mintInternal(msg.sender, mintPrice);
      amountMinted++;
      budgetLeft -= mintPrice;
      mintPrice = calculateMintPrice();
    }
```

# Generation cannot be incremented due to access control issue

## Impact

Generation number is designed to be incremented automatically when minting count for the current generation reach 10 000 entities. But with the current implementation of `_incrementGeneration()` this wont be possible and game would be stuck on generation 1.
## Proof of Concept

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345
https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/EntropyGenerator/EntropyGenerator.sol#L206

`_incrementGeneration()` inside `TraitForgeNft.sol` relies on `initializeAlphaIndices()` in `EntropyGenerator.sol`. `initializeAlphaIndices()` has `onlyOwner` modifier which is the contract which deployed the contract thus the NFT contract wont be able to call it.

```
function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
    generationMintCounts[currentGeneration] = 0;
    priceIncrement = priceIncrement + priceIncrementByGen;
    // Problem external call as address(this) will be different than the owner contract of entropyGenerator.
    entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
  }
```

```
function initializeAlphaIndices() public whenNotPaused onlyOwner { // Incorrect access control modifer, should allow nft contract to call it
    uint256 hashValue = uint256(
      keccak256(abi.encodePacked(blockhash(block.number - 1), block.timestamp))
    );

    uint256 slotIndexSelection = (hashValue % 258) + 512;
    uint256 numberIndexSelection = hashValue % 13;

    slotIndexSelectionPoint = slotIndexSelection;
    numberIndexSelectionPoint = numberIndexSelection;
  }
```
## Tools Used

Manual review.
## Recommended Mitigation Steps

Use the `onlyAllowedCaller` modifier instead of `onlyOwner`

# Possibility to create more than 10 000 entities per generation

## Impact

Overriding counter for minted entities per generation leads to possibility of more than 10 000 entities per generation. \
## Proof of Concept

https://github.com/code-423n4/2024-07-traitforge/blob/279b2887e3d38bc219a05d332cbcb0655b2dc644/contracts/TraitForgeNft/TraitForgeNft.sol#L345
The problem arises when transitioning between generations in the `TraitForgeNft` contract. The `generationMintCounts` mapping tracks the number of entities minted per generation. When the generation is incremented using the `_incrementGeneration()` function, the mint count for the new generation is set to zero:

```
function _incrementGeneration() private {
    require(
      generationMintCounts[currentGeneration] >= maxTokensPerGen,
      'Generation limit not yet reached'
    );
    currentGeneration++;
    generationMintCounts[currentGeneration] = 0; // problem line
    priceIncrement = priceIncrement + priceIncrementByGen;
    entropyGenerator.initializeAlphaIndices();
    emit GenerationIncremented(currentGeneration);
  }
```

The issue is caused by the line `generationMintCounts[currentGeneration] = 0;` in the `_incrementGeneration()` function. When the generation is incremented, this line sets the mint count for the new generation to zero, disregarding any previously minted entities for that generation. This results in an incorrect mint count for the new generation.

The provided test demonstrates a scenario where the issue occurs:

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console} from "forge-std/Test.sol";
import {EntityForging} from "contracts/EntityForging/EntityForging.sol";
import {IEntityForging} from "contracts/EntityForging/IEntityForging.sol";
import {TraitForgeNft} from "contracts/TraitForgeNft/TraitForgeNft.sol";
import {EntropyGenerator} from "contracts/EntropyGenerator/EntropyGenerator.sol";
import {Airdrop} from "contracts/Airdrop/Airdrop.sol";
import {DevFund} from "contracts/DevFund/DevFund.sol";
import {NukeFund} from "contracts/NukeFund/NukeFund.sol";
import {EntityTrading} from "contracts/EntityTrading/EntityTrading.sol";
import {IEntityTrading} from "contracts/EntityTrading/IEntityTrading.sol";

contract EntityForgingTest is Test {
    EntityForging entityForging;
    TraitForgeNft traitForgeNft;
    EntropyGenerator entropyGenerator;
    Airdrop airdrop;
    DevFund devFund;
    NukeFund nukeFund;
    EntityTrading entityTrading;

    address ownerAddr = makeAddr("ownerAddr");
    address player1 = makeAddr("player1");
    address player2 = makeAddr("player2");

    function setUp() public {
        vm.startPrank(ownerAddr);

        // Deploy TraitForgeNft contract
        traitForgeNft = new TraitForgeNft();

        // Deploy Airdrop contract
        airdrop = new Airdrop();
        traitForgeNft.setAirdropContract(address(airdrop));
        airdrop.transferOwnership(address(traitForgeNft));

        // Deploy entropyGenerator contract
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entropyGenerator.writeEntropyBatch1();
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));

        // Deploy EntityForging contract
        entityForging = new EntityForging(address(traitForgeNft));
        traitForgeNft.setEntityForgingContract(address(entityForging));

        devFund = new DevFund();
        nukeFund = new NukeFund(address(traitForgeNft), address(airdrop), payable(address(devFund)), payable(ownerAddr));
        traitForgeNft.setNukeFundContract(payable(address(nukeFund)));

        entityTrading = new EntityTrading(address(traitForgeNft));
        entityTrading.setNukeFundAddress(payable(address(nukeFund)));

        vm.stopPrank();

        vm.deal(player1, 10000 ether);
        vm.deal(player2, 10000 ether);

        // Used to avoid whitelist proof
        vm.warp(86402);
        vm.roll(86402);
    }

    function testMint() public {
        bytes32[] memory proof;
        // ---- Initial Minting ----
        vm.prank(player1);
        traitForgeNft.mintWithBudget{value: 1 ether}(proof);
        vm.prank(player2);
        traitForgeNft.mintWithBudget{value: 1 ether}(proof);

        console.log("Generation 1 mint count after initial mint: %s", traitForgeNft.generationMintCounts(1));
        console.log("Generation 2 mint count after initial mint: %s", traitForgeNft.generationMintCounts(2));

        // ---- First Forge ----
        vm.startPrank(player1);
        entityForging.listForForging(getForgerFromPlayer1(), 0.1 ether);
        vm.stopPrank();

        vm.startPrank(player2);
        entityForging.forgeWithListed{value: 0.1 ether}(getForgerFromPlayer1(), getMergerFromPlayer2());

        // ---- Second Forge ----
        vm.startPrank(player1);
        entityForging.listForForging(getForgerFromPlayer1(), 0.1 ether);
        vm.stopPrank();

        vm.startPrank(player2);
        entityForging.forgeWithListed{value: 0.1 ether}(getForgerFromPlayer1(), getMergerFromPlayer2());
        vm.stopPrank();

        console.log("Generation 1 mint count after forging: %s", traitForgeNft.generationMintCounts(1));
        console.log("Generation 2 mint count after forging: %s", traitForgeNft.generationMintCounts(2));

        // ---- Mint 10 000 more entities to switch to next generation ----
        uint256 initalMintCount = traitForgeNft.generationMintCounts(1);
        vm.startPrank(player1);
        for (uint256 i = 0; i <= 10000 - initalMintCount; i++) {
            traitForgeNft.mintToken{value: 10 ether}(proof);
        }

        console.log("Generation 1 mint count after second mint from player1: %s", traitForgeNft.generationMintCounts(1));
        console.log("Generation 2 mint count after second mint from player1: %s", traitForgeNft.generationMintCounts(2));
    }

    function getForgerFromPlayer1() private returns (uint256) {
        // hardcoded lastTokenId based on prices and provided value when minted
        for (uint256 i = 1; i <= 147; i++) {
            if (traitForgeNft.tokenEntropy(i) % 3 == 0 && uint8((traitForgeNft.tokenEntropy(i / 10) % 10)) > 0) {
                return i;
            }
        }
        return 0;
    }

    function getMergerFromPlayer2() private returns (uint256) {
        // hardcoded lastTokenId based on prices and provided value when minted
        for (uint256 i = 148; i <= 248; i++) {
            if (traitForgeNft.tokenEntropy(i) % 3 != 0 && uint8((traitForgeNft.tokenEntropy(i / 10) % 10)) > 0) {
                return i;
            }
        }
        return 0;
    }
}

```

```
Logs:
  Generation 1 mint count after initial mint: 248
  Generation 2 mint count after initial mint: 0
  Generation 1 mint count after forging: 248
  Generation 2 mint count after forging: 2
  Generation 1 mint count after second mint from player1: 10000
  Generation 2 mint count after second mint from player1: 1
```

The mint count for Generation 1 reaches its limit (10,000), triggering an increment to Generation 2. However, the mint count for Generation 2 unexpectedly shows as 1 despite having 2 entities already minted during forging.

To run test use: `forge test -vv --mc EntityForgingTest --gas-limit 10000000000`
## Tools Used
Manual review, Foundry.
## Recommended Mitigation Steps

Remove `generationMintCounts[currentGeneration] = 0;` line in `_incrementGeneration()` function.

# Incorrect calculation logic/parameters for claim amount when entity is nuked

## Impact
Based on the documentation the following should be true:
`The maximum total NukeFactor is 50%`
`Entities move from their initial Nuke Factor (set randomly) towards the maximum with a speed dictated by Performance Factor (set randomly).`
`fastest maturing in around 30 days, and slowest 600 days`

But in reality nuke factor is much lower than expected in the current implementation, leading to much lower rewards for users which decide to nuke.
## Proof of Concept

In order to calculate the amount to be claimed when nuking an entity, we need to consider initial nuke factor, performance factor and age of an entity. Nuke factor and performance factor are derived from the entropy of the entity. The math equation for the claim amount can be represented like this:

`age = (daysOld * performanceFactor * MAX_DENOMINATOR * ageMultiplier) / 365`
`nukeFactor = (age * defaultNukeFactorIncrease) / MAX_DENOMINATOR + initialNukeFactor`
`potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR`

Having these equations and knowing that:
`ageMultiplier = 1 (sponsor confirmed)`
`MAX_DENOMINATOR = 100000`
`defaultNukeFactorIncrease = 250`
`performanceFactor is between 0 and 9`
`max entropy is 99999`

Even with max entropy and 600 days maturity, claim amount would be much lower than expected 50%

```
 function nuke(uint256 tokenId) public whenNotPaused nonReentrant {
    require(
      nftContract.isApprovedOrOwner(msg.sender, tokenId),
      'ERC721: caller is not token owner or approved'
    );
    require(
      nftContract.getApproved(tokenId) == address(this) ||
        nftContract.isApprovedForAll(msg.sender, address(this)),
      'Contract must be approved to transfer the NFT.'
    );
    require(canTokenBeNuked(tokenId), 'Token is not mature yet');

    uint256 finalNukeFactor = calculateNukeFactor(tokenId); // finalNukeFactor has 5 digits
    uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
    uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

    // Directly assign the value to claimAmount based on the condition, removing the redeclaration
    uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam
      ? maxAllowedClaimAmount
      : potentialClaimAmount;

    fund -= claimAmount; // Deduct the claim amount from the fund

    nftContract.burn(tokenId); // Burn the token
    (bool success, ) = payable(msg.sender).call{ value: claimAmount }('');
    require(success, 'Failed to send Ether');

    emit Nuked(msg.sender, tokenId, claimAmount); // Emit the event with the actual claim amount
    emit FundBalanceUpdated(fund); // Update the fund balance
  }
```

```
function calculateNukeFactor(uint256 tokenId) public view returns (uint256) {
    require(
      nftContract.ownerOf(tokenId) != address(0),
      'ERC721: operator query for nonexistent token'
    );

    uint256 entropy = nftContract.getTokenEntropy(tokenId);
    uint256 adjustedAge = calculateAge(tokenId);

    uint256 initialNukeFactor = entropy / 40; // calcualte initalNukeFactor based on entropy, 5 digits

    uint256 finalNukeFactor = ((adjustedAge * defaultNukeFactorIncrease) /
      MAX_DENOMINATOR) + initialNukeFactor;

    return finalNukeFactor;
  }
```

```
function calculateAge(uint256 tokenId) public view returns (uint256) {
    require(nftContract.ownerOf(tokenId) != address(0), 'Token does not exist');

    uint256 daysOld = (block.timestamp -
      nftContract.getTokenCreationTimestamp(tokenId)) /
      60 /
      60 /
      24;
    uint256 perfomanceFactor = nftContract.getTokenEntropy(tokenId) % 10;

    uint256 age = (daysOld *
      perfomanceFactor *
      MAX_DENOMINATOR *
      ageMultiplier) / 365; // add 5 digits for decimals
    return age;
  }
```

Test case using the provided implementation, changed to suite the scenario with 600 days maturity period and 99999 entropy.

```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console} from "forge-std/Test.sol";
import {EntityForging} from "contracts/EntityForging/EntityForging.sol";
import {IEntityForging} from "contracts/EntityForging/IEntityForging.sol";
import {TraitForgeNft} from "contracts/TraitForgeNft/TraitForgeNft.sol";
import {EntropyGenerator} from "contracts/EntropyGenerator/EntropyGenerator.sol";
import {Airdrop} from "contracts/Airdrop/Airdrop.sol";
import {DevFund} from "contracts/DevFund/DevFund.sol";
import {NukeFund} from "contracts/NukeFund/NukeFund.sol";
import {EntityTrading} from "contracts/EntityTrading/EntityTrading.sol";
import {IEntityTrading} from "contracts/EntityTrading/IEntityTrading.sol";

contract NukeFundTest is Test {
    EntityForging entityForging;
    TraitForgeNft traitForgeNft;
    EntropyGenerator entropyGenerator;
    Airdrop airdrop;
    DevFund devFund;
    NukeFund nukeFund;
    EntityTrading entityTrading;

    address ownerAddr = makeAddr("ownerAddr");
    address player1 = makeAddr("player1");
    address player2 = makeAddr("player2");

    uint256 public constant MAX_DENOMINATOR = 100000;
    uint256 public nukeFactorMaxParam = MAX_DENOMINATOR / 2;
    uint256 public defaultNukeFactorIncrease = 250;
    uint256 public maxAllowedClaimDivisor = 2;
    uint256 public ageMultiplier = 1;

    function setUp() public {
        vm.startPrank(ownerAddr);

        // Deploy TraitForgeNft contract
        traitForgeNft = new TraitForgeNft();

        // Deploy Airdrop contract
        airdrop = new Airdrop();
        traitForgeNft.setAirdropContract(address(airdrop));
        airdrop.transferOwnership(address(traitForgeNft));

        // Deploy entropyGenerator contract
        entropyGenerator = new EntropyGenerator(address(traitForgeNft));
        entropyGenerator.writeEntropyBatch1();
        traitForgeNft.setEntropyGenerator(address(entropyGenerator));

        // Deploy EntityForging contract
        entityForging = new EntityForging(address(traitForgeNft));
        traitForgeNft.setEntityForgingContract(address(entityForging));

        devFund = new DevFund();
        nukeFund = new NukeFund(address(traitForgeNft), address(airdrop), payable(address(devFund)), payable(ownerAddr));
        traitForgeNft.setNukeFundContract(payable(address(nukeFund)));

        entityTrading = new EntityTrading(address(traitForgeNft));
        entityTrading.setNukeFundAddress(payable(address(nukeFund)));

        vm.stopPrank();

        vm.deal(player1, 10000 ether);
        vm.deal(player2, 10000 ether);

        // Used to avoid whitelist proof
        vm.warp(86402);
        vm.roll(86402);
    }

    function calculateAge() public returns (uint256) {
        uint256 daysOld = getDaysOld();
        uint256 perfomanceFactor = getTokenEntropy() % 10;

        uint256 age = (daysOld * perfomanceFactor * MAX_DENOMINATOR * ageMultiplier) / 365; // add 5 digits for decimals
        return age;
    }

    // Calculate the nuke factor of a token, which affects the claimable amount from the fund
    function calculateNukeFactor() public returns (uint256) {
        uint256 entropy = getTokenEntropy();
        uint256 adjustedAge = calculateAge();

        uint256 initialNukeFactor = entropy / 40; // calcualte initalNukeFactor based on entropy, 5 digits

        uint256 finalNukeFactor = ((adjustedAge * defaultNukeFactorIncrease) / MAX_DENOMINATOR) + initialNukeFactor;

        return finalNukeFactor;
    }

    function nuke(uint256 fund) public returns (uint256) {
        uint256 finalNukeFactor = calculateNukeFactor(); // finalNukeFactor has 5 digits
        uint256 potentialClaimAmount = (fund * finalNukeFactor) / MAX_DENOMINATOR; // Calculate the potential claim amount based on the finalNukeFactor
        uint256 maxAllowedClaimAmount = fund / maxAllowedClaimDivisor; // Define a maximum allowed claim amount as 50% of the current fund size

        // Directly assign the value to claimAmount based on the condition, removing the redeclaration
        uint256 claimAmount = finalNukeFactor > nukeFactorMaxParam ? maxAllowedClaimAmount : potentialClaimAmount;

        return claimAmount;
    }

    function getDaysOld() private returns (uint256) {
        return 600;
    }

    function getTokenEntropy() private returns (uint256) {
        return 99999;
    }

    function testClaimAmount() public {
        vm.assertEq(nuke(10 ether), 0.6197 ether);
    }
}
```

## Tools Used

Manual review, Foundry.

## Recommended Mitigation Steps

Consider adjusting initial values for `defaultNukeFactorIncrease` or `ageMultiplier` to match the expected behaviour in the docs.

