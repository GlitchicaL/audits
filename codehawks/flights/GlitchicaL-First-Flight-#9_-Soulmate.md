# First Flight #9: Soulmate - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `Staking::claimRewards()` function incorrectly tracks user deposits leading to incorrect distribution of rewards.](#H-01)
    - ### [H-02. Any user can claim airdrop rewards via `Airdrop::claim()` resulting in a loss of funds](#H-02)
    - ### [H-03. `Airdrop::claim()` function distributes reward tokens for divorced soulmates](#H-03)
- ## Medium Risk Findings
    - ### [M-01. A user can call `Soulmate::getDivorced()` without having a soulmate](#M-01)
    - ### [M-02. Any user can call `Soulmate::writeMessageInSharedSpace()` to write a message](#M-02)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #9

### Dates: Feb 8th, 2024 - Feb 15th, 2024

[See more contest details here](https://www.codehawks.com/contests/clsathvgg0005yhmxmoe455mm)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 2
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. `Staking::claimRewards()` function incorrectly tracks user deposits leading to incorrect distribution of rewards.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Staking.sol#L74-L76

### Summary

The `Staking::claimRewards()` function incorrectly tracks timestamp for `msg.sender` and allows anyone to claim token rewards without having to wait 1 week since a deposit.

### Vulnerability Details

The `Staking::claimRewards()` function is meant to give tokens to those who have deposited tokens into the contract for a specific time. For example, if 1 token is deposited, they can claim 1 token after 1 week, however the contract does not follow the intended behavior. The issue lies when setting the `lastClaim` local variable:

```javascript
    function claimRewards() public {
        uint256 soulmateId = soulmateContract.ownerToId(msg.sender);
        // first claim
        if (lastClaim[msg.sender] == 0) {
@>          lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(
@>            soulmateId
@>          );
        }

        // How many weeks passed since the last claim.
        // Thanks to round-down division, it will be the lower amount possible until a week has completly pass.
        uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
            lastClaim[msg.sender]) / 1 weeks);

        if (timeInWeeksSinceLastClaim < 1)
            revert Staking__StakingPeriodTooShort();

        lastClaim[msg.sender] = block.timestamp;

        // Send the same amount of LoveToken as the week waited times the number of token staked
        uint256 amountToClaim = userStakes[msg.sender] *
            timeInWeeksSinceLastClaim;
        loveToken.transferFrom(
            address(stakingVault),
            msg.sender,
            amountToClaim
        );

        emit RewardsClaimed(msg.sender, amountToClaim);
    }
```

When the contract is first called, the function declares and sets the `soulmateId` of `msg.sender` by calling the `Soulmate::ownerToId()` function. This will set the soulmate NFT ID that belongs to `msg.sender`. It's worth noting, if `msg.sender` has never claimed a soulmate via `Soulmate::mintSoulmateToken()` this will set the `soulmateId` to 0, while if the `msg.sender` is awaiting a soulmate, this will set `soulmateId` to the NFT ID that will be minted.

The function proceeds to see if `lastClaim[msg.sender]` equals 0. If we assume this is the first time a soulmate attempts to claim, it will call `Soulmate::idToCreationTimestamp()` function and set `lastClaim[msg.sender]` to the timestamp their soulmate NFT was created. For a user who has no soulmate, this will set `lastClaim[msg.sender]` to the time the first soulmate NFT was created. For a user who is still awaiting a soulmate, it will set `lastClaim[msg.sender]` to 0 as the timestamp of the next soulmate NFT ID doesn't exist yet.

If we then look at how then the `timeInWeeksSinceLastClaim` local variable is set:

```javascript
        uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
            lastClaim[msg.sender]) / 1 weeks);
```

This takes the current `block.timestamp` and minuses it from `lastClaim[msg.sender]`. This results in the time since the soulmate NFT was created, and not when tokens were deposited into the staking contract.

For a user who has never minted a soulmate, This results in the time since the first soulmate NFT was created, and not when tokens were deposited into the staking contract.

In a case where a user is awaiting a soulmate, `block.timestamp - 0` will result in just `block.timestamp`. Thus the calculation for `timeInWeeksSinceLastClaim` will just be `block.timestamp / 1 weeks` (604800). If we assume `block.timestamp` is equal to `X` amount of weeks, this user will be rewarded `X` amount of tokens regardless of when their tokens were deposited.

It's worth noting if no soulmate NFTs have been minted, there should be no way for tokens to be deposited assuming the only way to claim tokens is initially through the `Airdrop::claim()` function. 

### Impact

- A soulmate is rewarded tokens based on when their soulmate NFT was created regardless of the time they deposited. 
- A user who has never minted a soulmate will be rewarded tokens based on when the first soulmate NFT was created regardless of the time they deposited.
- A user awaiting a soulmate but has deposited tokens can be rewarded tokens based on current `block.timestamp` divided by 604800 (`1 weeks`) regardless of the time they deposited.

### POC

We can imagine the most-likely scenario and how it plays out:

1. User A calls `Soulmate::mintSoulmateToken()`.
2. User B calls `Soulmate::mintSoulmateToken()` and now NFT #0 has been minted.
3. 7 days passes by since NFT #0 was minted.
4. User A calls `Airdrop::claim()` and now has 7 lovetokens.
5. User A transfers 7 lovetokens to user C.
6. User C deposits 7 lovetokens to Staking contract.
6. User C calls `Soulmate::mintSoulmateToken()` and is awaiting a soulmate.
7. User C calls `Staking::claimRewards()`.

Below you can see a POC of the above scenario including how `block.timestamp` affects rewards:

```javascript
    function test_ClaimRewardsWithoutSoulmate() public {
        vm.warp(block.timestamp + 2 weeks);

        _mintOneTokenForBothSoulmates();

        vm.warp(block.timestamp + 1 weeks + 1 seconds);

        vm.prank(soulmate1);
        airdropContract.claim();

        address attacker = makeAddr("attacker");
        uint256 amountToAttackWith = 7 ether;

        vm.prank(soulmate1);
        loveToken.transfer(attacker, amountToAttackWith);

        vm.startPrank(attacker);
        soulmateContract.mintSoulmateToken();
        loveToken.approve(address(stakingContract), amountToAttackWith);
        stakingContract.deposit(amountToAttackWith);
        stakingContract.claimRewards();

        assertTrue(loveToken.balanceOf(attacker) == 21 ether); // 7 tokens deposited * 3 weeks = 21 tokens!
    }
```

### Tools Used

VS Code, Foundry

### Recommendations

Consider creating a `lastDeposit` mapping and setting the mapping in `Staking::deposit()` for a deposit and reference that timestamp in `Staking::claimRewards()` for the very first claim of an user. You'll also want to include a check to make sure a user has deposited. Note that this could "reset" the ability to claim if the user makes multiple deposits prior to their first claim:

```diff
+   error Staking__NoDepositsMade();

+   mapping(address user => uint256 timestamp) public lastDeposit;

    function deposit(uint256 amount) public {
        if (loveToken.balanceOf(address(stakingVault)) == 0)
            revert Staking__NoMoreRewards();
        // No require needed because of overflow protection
        userStakes[msg.sender] += amount;
+       lastDeposit[msg.sender] = block.timestamp;
        loveToken.transferFrom(msg.sender, address(this), amount);

        emit Deposited(msg.sender, amount);
    }

    function claimRewards() public {
-       uint256 soulmateId = soulmateContract.ownerToId(msg.sender);
+       if (lastDeposit[msg.sender] == 0)
+           revert Staking__NoDepositsMade();

        // first claim
        if (lastClaim[msg.sender] == 0) {
-           lastClaim[msg.sender] = soulmateContract.idToCreationTimestamp(
-               soulmateId
-           );
+           lastClaim[msg.sender] = lastDeposit[msg.sender];
        }

        // How many weeks passed since the last claim.
        // Thanks to round-down division, it will be the lower amount possible until a week has completly pass.
        uint256 timeInWeeksSinceLastClaim = ((block.timestamp -
            lastClaim[msg.sender]) / 1 weeks);

        if (timeInWeeksSinceLastClaim < 1)
            revert Staking__StakingPeriodTooShort();

        lastClaim[msg.sender] = block.timestamp;

        // Send the same amount of LoveToken as the week waited times the number of token staked
        uint256 amountToClaim = userStakes[msg.sender] *
            timeInWeeksSinceLastClaim;
        loveToken.transferFrom(
            address(stakingVault),
            msg.sender,
            amountToClaim
        );

        emit RewardsClaimed(msg.sender, amountToClaim);
    }
```
## <a id='H-02'></a>H-02. Any user can claim airdrop rewards via `Airdrop::claim()` resulting in a loss of funds            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Airdrop.sol#L58

### Summary

The `Airdrop::claim()` function distributes tokens to any caller, even to those awaiting a soulmate, or have never minted a soulmate. 

### Vulnerability Details

The `Airdrop::claim()` function is meant to give tokens to soulmates, however any user, even those who don't have a soulmate or never minted a soulmate via the function `Soulmate::mintSoulmateToken()` can claim rewards. This can happen due to setting the `numberOfDaysInCouple` local variable:

```javascript
    function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
        if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();

        // Calculating since how long soulmates are reunited
@>      uint256 numberOfDaysInCouple = (block.timestamp -
@>          soulmateContract.idToCreationTimestamp(
@>              soulmateContract.ownerToId(msg.sender)
@>          )) / daysInSecond;

        uint256 amountAlreadyClaimed = _claimedBy[msg.sender];

        if (
            amountAlreadyClaimed >=
            numberOfDaysInCouple * 10 ** loveToken.decimals()
        ) revert Airdrop__PreviousTokenAlreadyClaimed();

        uint256 tokenAmountToDistribute = (numberOfDaysInCouple *
            10 ** loveToken.decimals()) - amountAlreadyClaimed;

        // Dust collector
        if (
            tokenAmountToDistribute >=
            loveToken.balanceOf(address(airdropVault))
        ) {
            tokenAmountToDistribute = loveToken.balanceOf(
                address(airdropVault)
            );
        }
        _claimedBy[msg.sender] += tokenAmountToDistribute;

        emit TokenClaimed(msg.sender, tokenAmountToDistribute);

        loveToken.transferFrom(
            address(airdropVault),
            msg.sender,
            tokenAmountToDistribute
        );
    }
```

When `numberOfDaysInCouple` is set, it initally calls `Soulmate::ownerToId()` and passes `msg.sender` as the argument. This mapping maps the address to a uint256:

```javascript
    mapping(address owner => uint256 id) public ownerToId;
```

Initially this mapping is set inside of `Soulmate::mintSoulmateToken()` for users that have a soulmate. However for users without a soulmate, this mapping will return `0` by default. 

When the function then calls the `Soulmate::idToCreationTimestamp()` function to fetch the timestamp of when the soulmates were united, it will actually return the timestamp of the very first soulmate NFT minted OR if no one has minted, this will also return `0`. 

If we assume a soulmate NFT has been minted, the function `Airdrop::claim()` will then continue to do calculations based off of the first NFT minted and transfer tokens to the caller `msg.sender` as if they were the first soulmate NFT minted.

In the case there has been no Soulmate NFTs minted, the `Airdrop::claim()` function will set `Airdrop::numberOfDaysInCouple` to the amount of days since `block.timestamp`'s inception: `(block.timestamp - 0) / 3600 * 24`.

### Impact

Any user can be rewarded with tokens without having actually minted a soulmate. 

### POC

Imagine the following most-likely scenario:

1. User A calls `Soulmate::mintSoulmateToken()`.
2. User B calls `Soulmate::mintSoulmateToken()` and now NFT #0 has been minted.
3. 10 days passes by since NFT #0 was minted.
4. User C (Attacker) calls `Airdrop::claim()`.
5. `Airdrop::claim()` checks the number of days passed. Since user C hasn't minted, it will set the days since NFT #0 was minted.
6. `Airdrop::claim()` rewards user C with 10 tokens.

Here is a POC of the above scenario:

```javascript
    function test_claimWithoutASoulmate() public {
        _mintOneTokenForBothSoulmates();

        vm.warp(block.timestamp + 10 days + 1 seconds);

        address attacker = makeAddr("attacker");
        vm.prank(attacker);
        airdropContract.claim();

        assertTrue(loveToken.balanceOf(attacker) == 10 ether);
    }
```

### Tools Used

VS Code, Foundry

### Recommendations

Add a new error and make a call to `Soulmate::soulmateOf` to check if `msg.sender` has a soulmate, if they don't revert the transaction. This will prevent those who never minted, in addition to those who are still waiting for a soulmate. 

```diff
+   error Airdrop__NoSoulmate();

    function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
        if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();

+       if (soulmateOf(msg.sender) == address(0)) revert Airdrop__NoSoulmate();

        // Calculating since how long soulmates are reunited
        uint256 numberOfDaysInCouple = (block.timestamp -
            soulmateContract.idToCreationTimestamp(
                soulmateContract.ownerToId(msg.sender)
            )) / daysInSecond;

        uint256 amountAlreadyClaimed = _claimedBy[msg.sender];

        if (
            amountAlreadyClaimed >=
            numberOfDaysInCouple * 10 ** loveToken.decimals()
        ) revert Airdrop__PreviousTokenAlreadyClaimed();

        uint256 tokenAmountToDistribute = (numberOfDaysInCouple *
            10 ** loveToken.decimals()) - amountAlreadyClaimed;

        // Dust collector
        if (
            tokenAmountToDistribute >=
            loveToken.balanceOf(address(airdropVault))
        ) {
            tokenAmountToDistribute = loveToken.balanceOf(
                address(airdropVault)
            );
        }
        _claimedBy[msg.sender] += tokenAmountToDistribute;

        emit TokenClaimed(msg.sender, tokenAmountToDistribute);

        loveToken.transferFrom(
            address(airdropVault),
            msg.sender,
            tokenAmountToDistribute
        );
    }
```
## <a id='H-03'></a>H-03. `Airdrop::claim()` function distributes reward tokens for divorced soulmates            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Airdrop.sol#L53

### Summary

A soulmate who is divorced can still claim airdrop rewards.

### Vulnerability Details

The `Airdrop::claim()` function rewards soulmates who are not divorced to claim lovetokens. The contract attempts to check the soulmates' divorce status by calling `Soulmate::isDivorced()`:

```javascript
    function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
@>      if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();

        // Calculating since how long soulmates are reunited
        uint256 numberOfDaysInCouple = (block.timestamp -
            soulmateContract.idToCreationTimestamp(
                soulmateContract.ownerToId(msg.sender)
            )) / daysInSecond;

        uint256 amountAlreadyClaimed = _claimedBy[msg.sender];

        if (
            amountAlreadyClaimed >=
            numberOfDaysInCouple * 10 ** loveToken.decimals()
        ) revert Airdrop__PreviousTokenAlreadyClaimed();

        uint256 tokenAmountToDistribute = (numberOfDaysInCouple *
            10 ** loveToken.decimals()) - amountAlreadyClaimed;

        // Dust collector
        if (
            tokenAmountToDistribute >=
            loveToken.balanceOf(address(airdropVault))
        ) {
            tokenAmountToDistribute = loveToken.balanceOf(
                address(airdropVault)
            );
        }
        _claimedBy[msg.sender] += tokenAmountToDistribute;

        emit TokenClaimed(msg.sender, tokenAmountToDistribute);

        loveToken.transferFrom(
            address(airdropVault),
            msg.sender,
            tokenAmountToDistribute
        );
    }
```

If we look inside of `Soulmate::isDivorced()` we'll see that it checks for the divorce status for `msg.sender`:

```javascript
    function isDivorced() public view returns (bool) {
        return divorced[msg.sender];
    }
```

Since Airdrop is calling `Soulmate::isDivorced()`, `msg.sender` will represent the Airdrop contract address and not the original caller of `Airdrop::claim()`. This will result in `Soulmate::isDivorced()` to return false thus bypassing the revert.

### Impact

A divorced soulmate will bypass the revert and will still be able to claim tokens.

### POC

We can imagine the following scenario:

1. User A calls `Soulmate::mintSoulmateToken()`.
2. User B calls `Soulmate::mintSoulmateToken()` and now user A & B are soulmates.
3. User A or B calls `Soulmate::getDivorced()` and now user A & B are divorced.
4. 1 day passes by.
5. User A or B calls `Airdrop::claim()`.
6. `Airdrop::claim()` checks the divorce status of it's own contract address instead of user A & B.
7. `Airdrop::claim()` rewards user A or B with tokens.

Here you can see a POC of the above scenario:

```javascript
    function test_bypassingDivorceCheck() public {
        _mintOneTokenForBothSoulmates();

        vm.prank(soulmate1);
        soulmateContract.getDivorced();

        vm.warp(block.timestamp + 200 days + 1 seconds);

        vm.prank(soulmate1);
        airdropContract.claim();

        assertTrue(loveToken.balanceOf(soulmate1) == 200 ether);

        vm.prank(soulmate2);
        airdropContract.claim();

        assertTrue(loveToken.balanceOf(soulmate2) == 200 ether);
    }
```

### Tools Used

VS Code, Foundry

### Recommendations

Declare a parameter of type address called `_soulmate` to `Soulmate::isDivorced()` and check the divorce status of `_soulmate` instead of `msg.sender`:

```diff
-   function isDivorced() public view returns (bool) {
+   function isDivorced(address _soulmate) public view returns (bool) {
-       return divorced[_soulmate];
+       return divorced[_soulmate];
    }
```
		
# Medium Risk Findings

## <a id='M-01'></a>M-02. Any user can call `Soulmate::writeMessageInSharedSpace()` to write a message            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-soulmate/blob/main/src/Soulmate.sol#L107

### Summary

Any user that hasn't minted via `Soulmate::mintSoulmateToken()` will be able to call the `Soulmate::writeMessageInSharedSpace()` to write a message to NFT ID #0.

### Vulnerability Details

The `Soulmate::writeMessageInSharedSpace()` is intended for soulmates to write messages in their shared space. However, if a user hasn't minted a soulmate, this results in the `id` local variable to be set to 0:

```javascript
    function writeMessageInSharedSpace(string calldata message) external {
@>      uint256 id = ownerToId[msg.sender];
        sharedSpace[id] = message;
        emit MessageWrittenInSharedSpace(id, message);
    }
```

Since `id` will be set to 0, the message will be written to `sharedSpace[0]` which belongs to soulmates that own NFT ID #0.

### Impact

This results in incorrect handling of state since anyone can write to `sharedSpace[0]`.

### POC

You can see the following foundry test where a non-soulmate modifies the state

```javascript
    function test_WriteAndReadSharedSpace() public {
        address attacker = makeAddr("attacker");
        vm.prank(attacker);
        soulmateContract.writeMessageInSharedSpace("Get rekt");

        vm.prank(soulmate2);
        string memory message = soulmateContract.readMessageInSharedSpace();

        string[4] memory possibleText = [
            "Get rekt, sweetheart",
            "Get rekt, darling",
            "Get rekt, my dear",
            "Get rekt, honey"
        ];
        bool found;
        for (uint i; i < possibleText.length; i++) {
            if (compare(possibleText[i], message)) {
                found = true;
                break;
            }
        }
        console2.log(message);
        assertTrue(found);
    }
```

### Tools Used

VS Code, Foundry

### Recommendations

Add a new error and check the `soulmateOf` mapping to see if `msg.sender` has a soulmate before writing messages:

```diff
+   error Soulmate__NoSoulmate();

    function writeMessageInSharedSpace(string calldata message) external {
+       if (soulmateOf(msg.sender) == address(0))
+            revert Soulmate__NoSoulmate();   
        uint256 id = ownerToId[msg.sender];
        sharedSpace[id] = message;
        emit MessageWrittenInSharedSpace(id, message);
    }
```




