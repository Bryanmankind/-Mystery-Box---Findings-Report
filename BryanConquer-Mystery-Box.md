# Mystery Box - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. Unauthorized access to changeOwner function](#H-01)
- ## Medium Risk Findings
    - ### [M-01. Weak random number generation in openBox](#M-01)
    - ### [M-02. transferReward Deletes Reward Without Shifting Array](#M-02)
    - ### [M-03. Inability to Assign New Rewards in openBox Function](#M-03)
- ## Low Risk Findings
    - ### [L-01. Gas Costs for claimAllRewards()](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #25

### Dates: Sep 26th, 2024 - Oct 3rd, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-09-mystery-box)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 3
- Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. Unauthorized access to changeOwner function            



## Summary

The `changeOwner` function allows any external caller to change the ownership of the contract. This exposes the contract to malicious actors, who can call this function to take over ownership without authorization. This creates a significant security vulnerability, potentially leading to complete loss of control over the contract.

## Vulnerability Details

```Solidity
 function changeOwner(address _newOwner) public {
        owner = _newOwner;
    }
```

## Impact

Any address is able to change ownership, regardless of whether it is the current owner.

## Tools Used

Manule review 

Recommendations\
Add a `require` statement to ensure that only the current owner can call this function:
---------------------------------------------------------------------------------------

```Solidity
 function changeOwner(address _newOwner) public {
   require(msg.sender == owner, "Only the owner can change ownership");
        owner = _newOwner;
}
```

    
# Medium Risk Findings

## <a id='M-01'></a>M-01. Weak random number generation in openBox            



## Summary

The `openBox` function uses `block.timestamp` and `msg.sender` to generate a "random" number, which is vulnerable to manipulation by miners or adversarial users, as these values can be influenced or predicted.

## Vulnerability Details

## Impact

The random number is predictable due to weak entropy sources.

## Tools Used

manual review 

## Recommendations

While true randomness is difficult to achieve on-chain, you can improve randomness by incorporating more unpredictable inputs like `block.difficulty` or an external randomness oracle such as Chainlink VRF

```Solidity
uint256 randomValue = uint256(keccak256(abi.encodePacked(block.timestamp, block.difficulty, msg.sender))) % 100;

```

## <a id='M-02'></a>M-02. transferReward Deletes Reward Without Shifting Array            



## Summary

The `transferReward` function uses `delete` to remove a reward from the `rewardsOwned` array, but this leaves a "gap" in the array, causing issues when iterating through it. This can result in lost rewards or unintentional behavior when trying to access elements later.

## The array should maintain a continuous structure after transferring a reward.

Impact\
lost rewards or unintentional behavior when trying to access elements later.
----------------------------------------------------------------------------

Tools Used\
manual 
-------

Recommendations\
Instead of using `delete`, consider shifting the array elements after removing a reward:
----------------------------------------------------------------------------------------

```Solidity
function transferReward(address _to, uint256 _index) public {
    require(_index < rewardsOwned[msg.sender].length, "Invalid index");
    rewardsOwned[_to].push(rewardsOwned[msg.sender][_index]);
    
    // Shift the array elements left to fill the gap
    for (uint256 i = _index; i < rewardsOwned[msg.sender].length - 1; i++) {
        rewardsOwned[msg.sender][i] = rewardsOwned[msg.sender][i + 1];
    }
    
    rewardsOwned[msg.sender].pop();
}

```

## <a id='M-03'></a>M-03. Inability to Assign New Rewards in openBox Function            



## Summary

The `openBox` function in the smart contract fails to account for dynamically added rewards. When the owner adds a new reward using `addReward`, users are unable to receive this newly added reward because the `openBox` function is hardcoded to only distribute the original set of rewards (Coal, Bronze Coin, Silver Coin, and Gold Coin). The current implementation does not accommodate newly added rewards or dynamically update the probabilities, resulting in the new reward being inaccessible to users.

## Vulnerability Details

```Solidity
function openBox() public {
        require(boxesOwned[msg.sender] > 0, "No boxes to open");

        // Generate a random number between 0 and 99
        uint256 randomValue = uint256(keccak256(abi.encodePacked(block.timestamp, msg.sender))) % 100;

        // Determine the reward based on probability
        if (randomValue < 75) {
            // 75% chance to get Coal (0-74)
            rewardsOwned[msg.sender].push(Reward("Coal", 0 ether));
        } else if (randomValue < 95) {
            // 20% chance to get Bronze Coin (75-94)
            rewardsOwned[msg.sender].push(Reward("Bronze Coin", 0.1 ether));
        } else if (randomValue < 99) {
            // 4% chance to get Silver Coin (95-98)
            rewardsOwned[msg.sender].push(Reward("Silver Coin", 0.5 ether));
        } else {
            // 1% chance to get Gold Coin (99)
            rewardsOwned[msg.sender].push(Reward("Gold Coin", 1 ether));
        }

        boxesOwned[msg.sender] -= 1;
    }
```

\
POC 
----

```Solidity
function testBug_NewRewardNotAssigned() public {
    // Owner adds a new reward (Diamond Coin) with 2 ether value
    vm.startPrank(owner);
    mysteryBox.addReward("Diamond Coin", 2 ether);

    // Verify that the reward was successfully added to the reward pool
    MysteryBox.Reward[] memory rewards = mysteryBox.getRewardPool();
    assertEq(rewards.length, 5);  // Reward pool now contains 5 rewards
    assertEq(rewards[4].name, "Diamond Coin");
    assertEq(rewards[4].value, 2 ether);
    vm.stopPrank();

    // User1 buys a mystery box
    vm.deal(user1, 2 ether);
    vm.startPrank(user1);
    mysteryBox.buyBox{value: 0.1 ether}();

    // User opens the mystery box
    mysteryBox.openBox();

    // Check the rewards assigned to the user
    MysteryBox.Reward[] memory userRewards = mysteryBox.getRewards();

    // Assert that the user did NOT receive the "Diamond Coin" (new reward)
    // Since the probabilities are hardcoded in openBox(), the new reward will never be assigned
    for (uint256 i = 0; i < userRewards.length; i++) {
        require(
            keccak256(bytes(userRewards[i].name)) != keccak256(bytes("Diamond Coin")),
            "Bug: User received the new reward which should not be possible"
        );
    }

    console2.log("Bug confirmed: User cannot receive newly added rewards due to hardcoded probabilities.");
    vm.stopPrank();
}

```



\
\
Impact
------

user will never receive the newly added reward

## Tools Used

Manual Rivew 

## Recommendations

**Dynamic Reward Pool**: Replace the hardcoded conditions in `openBox` with a loop that dynamically checks the cumulative probabilities of all rewards in the `rewardPool`.

* **Adjustable Probabilities**: Allow the owner to specify probabilities for new rewards when they are added, ensuring that the total probabilities always sum to 100%.


# Low Risk Findings

## <a id='L-01'></a>L-01. Gas Costs for claimAllRewards()            



## Summary

The `claimAllRewards` function iterates through all rewards in `rewardsOwned[msg.sender]`, which can be expensive in terms of gas fees if a user has many rewards.

## Vulnerability Details

```Solidity
 function claimAllRewards() public {
        uint256 totalValue = 0;
        for (uint256 i = 0; i < rewardsOwned[msg.sender].length; i++) {
            totalValue += rewardsOwned[msg.sender][i].value;
        }
   }
```

## Impact

Gas costs increase linearly with the number of rewards, which could make the function impractical for users with many rewards.

## Tools Used

Recommendations\
Consider breaking up the reward claiming process into smaller steps or allowing users to claim rewards incrementally to avoid excessive gas fees.
-------------------------------------------------------------------------------------------------------------------------------------------------



