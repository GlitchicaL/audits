# First Flight #10: One Shot - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `RapBattle::_battle()` uses weak randomness to determine a winner](#H-01)
    - ### [H-02. A user can call `RapBattle::goOnStageOrBattle()` and represent any Rapper NFT](#H-02)
    - ### [H-03. A user calling `RapBattle::goOnStageOrBattle()` can avoid losing their tokens by not approving.](#H-03)

- ## Low Risk Findings
    - ### [L-01. `RapBattle::_battle()` can emit incorrect event parameters](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #10

### Dates: Feb 22nd, 2024 - Feb 29th, 2024

[See more contest details here](https://www.codehawks.com/contests/clstf5qd2000rakskkj0lkm33)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 0
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. `RapBattle::_battle()` uses weak randomness to determine a winner            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/main/src/RapBattle.sol#L62-L63

### Summary

`RapBattle::_battle()` uses the hash of `block.timestamp`, `block.prevrandao`, `msg.sender`, and `totalBattleSkill` to determine a winner but this is known to be predictable. 

### Vulnerability Details

Inside of `RapBattle::_battle()` it declares a local variable of type uint256 called `random`:

```javascript
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

@>      uint256 random =
@>          uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
        emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```

The value of `random` is assigned by hashing `block.timestamp`, `block.prevrandao`, `msg.sender`, and `totalBattleSkill`. Since `block.timestamp`, `block.prevrandao`, `msg.sender`, and `totalBattleSkill` can all be known at the time of execution, this allows a user to know the value of `random` and ultimately the outcome of who wins the battle.

### Impact

- A user can know the outcome of the battle and call the function for a guaranteed win.

### Tools Used

VS Code, Slither

### Recommendations

Consider using [Chainlink VRF](https://docs.chain.link/vrf) for true randomness.
## <a id='H-02'></a>H-02. A user can call `RapBattle::goOnStageOrBattle()` and represent any Rapper NFT            



### Summary

A user can pass any `_tokenId` into `RapBattle::goOnStageOrBattle()` without actually owning that NFT to challenge a defender.

### Vulnerability Details

the `RapBattle::goOnStageOrBattle()` function is meant to assign a defender if there is no defender, else allow a challenger to challenge the defender. It requires 2 arguments, a uint256 representing a Rapper's NFT ID and a uint256 representing how many cred tokens to bet:

```javascript
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

If defender is address 0, then we set the defender variable to `msg.sender`, set the cred bet, and the rapper ID that will be defending. It then emits an event, and transfers the rapper NFT in addition to the tokens to the contract.

If the defender address is not address 0, then we have a defender, and the caller `msg.sender` of `RapBattle::goOnStageOrBattle()` will be the challenger. Note that execution in the else block goes into `RapBattle::_battle()` passing in the `_tokenId` for the first argument and does not check to see if `msg.sender` owns the `_tokenId` that is passed in. If we look into `RapBattle::_battle()` function:

```javascript
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
        emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```

We'll see that `_tokenId` is only referenced twice. First for getting the rapper's skill by calling `RapBattle::getRapperSkill()`, the 2nd for emitting the Battle event. If we look at `RapBattle::getRapperSkill()`:

```javascript
    function getRapperSkill(
        uint256 _tokenId
    ) public view returns (uint256 finalSkill) {
        IOneShot.RapperStats memory stats = oneShotNft.getRapperStats(_tokenId);
        finalSkill = BASE_SKILL;
        if (stats.weakKnees) {
            finalSkill -= VICE_DECREMENT;
        }
        if (stats.heavyArms) {
            finalSkill -= VICE_DECREMENT;
        }
        if (stats.spaghettiSweater) {
            finalSkill -= VICE_DECREMENT;
        }
        if (stats.calmAndReady) {
            finalSkill += VIRTUE_INCREMENT;
        }
    }
```

This function then calls `OneShot::getRapperStats()` passing in the `_tokenId` to get the stats of the token:

```javascript
    function getRapperStats(uint256 tokenId) public view returns (RapperStats memory) {
        return rapperStats[tokenId];
    }
```

Since throughout the remainder of the `RapBattle::_battle()` function there is no other reference to the `_tokenId` to ensure the challenger `msg.sender` owns the rapper, this allows any user to challenge a defender with any `_tokenId` that they don't own.

### Impact

- Puts the defender at a disadvantage to lose their bet as a challenger can pass in a `_tokenId` with better skills.

### POC

Here is how the scenario can play out:

1. User A calls `OneShot::mintRapper()` and gets NFT #0.
2. User A calls `RapBattle::goOnStageOrBattle()` to place a bet
3. User B calls `OneShot::mintRapper()` and gets NFT #1.
4. User C calls `RapBattle::goOnStageOrBattle()` passing in `_tokenId` of NFT #1.
5. User C wins the bet and gets rewarded the defender tokens.

Below you can see a POC of the above scenario:

```javascript
    function testUserNotApprovingNFTBeforeBattle() public twoSkilledRappers {
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        rapBattle.goOnStageOrBattle(0, 0);
        vm.stopPrank();

        address attacker = makeAddr("attacker");
        vm.prank(attacker);

        vm.warp(2 hours);
        vm.recordLogs();
        rapBattle.goOnStageOrBattle(1, 0);
        vm.stopPrank();

        Vm.Log[] memory entries = vm.getRecordedLogs();
        address winner = address(uint160(uint256(entries[0].topics[2])));
        assert(winner == attacker);
    }
```

### Tools Used

VS Code, Foundry

### Recommendations

Consider adding a check to ensure the challenger owns their NFT. Note that this would require their NFT to not be staked in the Streets contract:

```diff
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
+       require(
+           oneShotNft.ownerOf(_tokenId) == msg.sender,
+           "RapBattle: Not the owner"
+       );
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
        emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```
## <a id='H-03'></a>H-03. A user calling `RapBattle::goOnStageOrBattle()` can avoid losing their tokens by not approving.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/main/src/RapBattle.sol#L70-L77

### Summary

A user can call the `RapBattle::goOnStageOrBattle()` function to challenge a defender without needing to approve their tokens.

### Vulnerability Details

the `RapBattle::goOnStageOrBattle()` function is meant to assign a defender if there is no defender, else allow a challenger to challenge the defender:

```javascript
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

When a defender has been assigned, and another user calls this function to challenge the defender, the internal `_battle()` function is called. In that function we determine the winner. If the winner is the defender then tokens are meant to be transferred from `msg.sender` (the challenger) to the defender via the `CredToken::transferFrom()` function:

```javascript
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
        emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
@>          credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```

The CredToken contract is a ERC20 contract therefore the caller of `transferFrom()` must be approved to transfer tokens on behalf of the user. If the `msg.sender` (the challenger) of `RapBattle::goOnStageOrBattle()` doesn't approve and the defender wins, this results in the function reverting.

Whereas if `msg.sender` (the challenger) of `RapBattle::goOnStageOrBattle()` doesn't approve and they win, this results in them earning tokens regardless of if they approved or not.

It's worth noting that in the scenario that `msg.sender` (the challenger) does approve the contract to transfer their tokens, and they win, this results in `CredToken::transferFrom()` not being executed and leaving the challengers `CredToken::_allowances` to not be updated.

### Impact

- Allows a challenger to only successfully call `RapBattle::goOnStageOrBattle()` when they win, thus not risking any of their tokens.
- Prevents a defender from earning tokens from a challenge they should've won.
- Causes `CredToken::_allowances` to not be updated when a challenger approves and wins.

### POC

Below you can see how a scenario plays out where the defender is expected to win but the challenger doesn't approve their CredTokens:

```javascript
    function testRapperNotApprovingBeforeBattle() public twoSkilledRappers {
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 10);
        rapBattle.goOnStageOrBattle(0, 3);
        vm.stopPrank();

        vm.startPrank(challenger);
        oneShot.approve(address(rapBattle), 1);

        vm.warp(1 days);
        vm.expectRevert();
        rapBattle.goOnStageOrBattle(1, 3);
        vm.stopPrank();
    }
```

### Tools Used

VS Code, Foundry

### Recommendations

Call `CredToken::transferFrom()` before calling `CredToken::_battle_()`:

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        if (defender == address(0)) {
            defender = msg.sender;
            defenderBet = _credBet;
            defenderTokenId = _tokenId;

            emit OnStage(msg.sender, _tokenId, _credBet);

            oneShotNft.transferFrom(msg.sender, address(this), _tokenId);
            credToken.transferFrom(msg.sender, address(this), _credBet);
        } else {
-           // credToken.transferFrom(msg.sender, address(this), _credBet);
+           credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

Then update `CredToken::_battle_()` to transfer the `totalPrize` of tokens since both the defender and challenger will have already transferred their tokens:

```diff
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
        emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
-           credToken.transfer(_defender, defenderBet);
-           credToken.transferFrom(msg.sender, _defender, _credBet);
+           credToken.transfer(_defender, totalPrize);
        } else {
-           // Otherwise, since the challenger never sent us the money, we just give the money in the contract
-           credToken.transfer(msg.sender, _credBet);
+           // Otherwise, send the prize to the challenger
+           credToken.transfer(msg.sender, totalPrize);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```
		


# Low Risk Findings

## <a id='L-01'></a>L-01. `RapBattle::_battle()` can emit incorrect event parameters            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/main/src/RapBattle.sol#L67

### Summary

The `RapBattle::_battle()` function can emit incorrect event parameters and make it appear that the challenger won the battle when in actuality the defender has won.

### Vulnerability Details

The `RapBattle::_battle()` function emits the Battle event with 3 parameters, the challenger (`msg.sender`), the `_tokenId`, and the winner of the battle:

```javascript
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
@>      emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```

The winner of the battle is determined by the condition that if `random` is less than the `defenderRapperSkill` then the event emits with the `_defender` as the winner. Where as if `random` is greater than `defenderRapperSkill`, then the event emits with the `msg.sender` (the challenger) as the winner.

This statement contradicts the following condition after the event is emitted that determines who wins the funds. As that statement determines the winner on the condition that if `random` is less than or equal to the `defenderRapperSkill` then the `_defender` wins the tokens. 

Due to the mismatch of conditions, there is a scenario where if the `random` number generated is equal to the `defenderRapperSkill`, the event emitted will emit with `msg.sender` (the challenger) as the winner but the funds will get transferred to the `_defender`.

### Impact

- The Battle event can emit the wrong winner of the battle

### POC

Below you can see a POC that shows how this can happen:

```javascript
    function testIncorrectBattleEvent() public twoSkilledRappers {
        uint256 DEFENDER_AMOUNT_TO_BET = cred.balanceOf(user);
        uint256 CHALLENGER_AMOUNT_TO_BET = cred.balanceOf(challenger);

        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), DEFENDER_AMOUNT_TO_BET);
        rapBattle.goOnStageOrBattle(0, DEFENDER_AMOUNT_TO_BET);
        vm.stopPrank();

        vm.startPrank(challenger);
        vm.warp(54 seconds);
        oneShot.approve(address(rapBattle), 1);
        cred.approve(address(rapBattle), CHALLENGER_AMOUNT_TO_BET);

        vm.recordLogs();
        rapBattle.goOnStageOrBattle(1, CHALLENGER_AMOUNT_TO_BET);
        vm.stopPrank();

        Vm.Log[] memory entries = vm.getRecordedLogs();
        address winner = address(uint160(uint256(entries[0].topics[2])));

        assert(winner == challenger); // The winner emitted from the event is the challenger...
        assert(cred.balanceOf(challenger) == 0); // However the challenger actually lost...
        assert(
            cred.balanceOf(user) ==
                DEFENDER_AMOUNT_TO_BET + CHALLENGER_AMOUNT_TO_BET
        );
    }
```

### Tools Used

VS Code, Foundry

### Recommendations

Update the condition of the determined winner when emitting the event:

```diff
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
-       emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);
+       emit Battle(msg.sender, _tokenId, random <= defenderRapperSkill ? _defender : msg.sender);

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```


