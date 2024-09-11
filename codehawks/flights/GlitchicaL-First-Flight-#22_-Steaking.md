# First Flight #22: Steaking - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `Steaking::stake()` incorrectly updates `Steaking::usersToStakes` resulting in loss of funds](#H-01)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #22

### Dates: Aug 15th, 2024 - Aug 22nd, 2024

[See more contest details here](https://codehawks.cyfrin.io/c/2024-08-steaking)

# <a id='results-summary'></a>Results Summary

### Number of findings:
- High: 1
- Medium: 0
- Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. `Steaking::stake()` incorrectly updates `Steaking::usersToStakes` resulting in loss of funds            



## Summary

The `Steaking::stake()` function incorrectly updates the `Steaking::usersToStakes` mapping, causing users to unstake or deposit incorrect funds to the vault.

## Vulnerability Details

The `Steaking::stake()` function allows users to deposit ETH, and is responsible for updating the `Steaking::usersToStakes` mapping. Currently the mapping is updated by setting the behalfed user to the `msg.value` in the function:

```Vyper
def stake(_onBehalfOf: address):
    """
    @notice Allows users to stake ETH for themselves or any other user within the staking period.
    @param _onBehalfOf The address to stake on behalf of.
    """
    assert not self._hasStakingPeriodEnded(), STEAK__STAKING_PERIOD_ENDED
    assert msg.value >= MIN_STAKE_AMOUNT, STEAK__INSUFFICIENT_STAKE_AMOUNT
    assert _onBehalfOf != ADDRESS_ZERO, STEAK__ADDRESS_ZERO

@>  self.usersToStakes[_onBehalfOf] = msg.value
    self.totalAmountStaked += msg.value

    log Staked(msg.sender, msg.value, _onBehalfOf)
```

Due to how the value is set for a user, this results in multiple future stakes to overwrite the previous stake rather than add on to what was previously staked. Imagine a scenario where Alice calls `Steaking::stake()` to deposit 1 ether on behalf of herself. The function will set `Steaking::usersToStakes[_onBehalfOf]` to `1000000000000000000`. At a later point, Alice decides to stake another 1 ether and calls `Steaking::stake()` again. At this point, `Steaking::usersToStakes[_onBehalfOf]` will then be set to `1000000000000000000` again.

Let's then say Alice decides to call `Steaking::unstake()` passing in 2 ether as the amount. Since `Steaking::usersToStakes[_onBehalfOf]` will represent 1 ether, Alice will be met with `Steaking::STEAK__INSUFFICIENT_STAKE_AMOUNT`.

Let's say Alice does not unstake, and waits until the staking period is over and deposits into the vault calling `Steaking::depositIntoVault()`. Since `Steaking::usersToStakes[_onBehalfOf]` for Alice will represent 1 ether, only 1 ether will be deposited into the vault, thus leaving the remaining 1 ether stuck in the Steaking contract.

## Impact

* Prevent users from unstaking their total ETH staked.
* Prevent users from being able to deposit the full amount of ETH staked.
* Results in ETH being stuck in the Steaking contract.

## Tools Used

Foundry, VS Code

## POC

```Solidity
    function testIncorrectStakedAmount() public {
        uint256 stakeAmount = 1 ether;

        // Let's stake 2 times, totaling 2 ether staked
        _stake(user1, stakeAmount, user1);
        _stake(user1, stakeAmount, user1);

        // Get user staked amount
        uint256 userStakedAmount = steaking.usersToStakes(user1);

        // user1 should have 2 ether staked but seemingly not the case...
        assertNotEq(userStakedAmount, 2 ether);

        // We should be able to unstake 2 ether...but can't...
        vm.expectRevert(bytes(STEAK__INSUFFICIENT_STAKE_AMOUNT));
        _unstake(user1, 2 ether, user1);

        // Let's wait and try depositing to the vault instead
        // Start the vault phase
        _endStakingPeriod();

        vm.startPrank(owner);
        steaking.setVaultAddress(address(wethSteakVault));
        vm.stopPrank();

        // Have user deposit into the vault
        vm.startPrank(user1);
        steaking.depositIntoVault();
        vm.stopPrank();

        // We expect the steaking balance to be 0, but actually ends up being 1 ether...
        uint256 steakingBalance = address(steaking).balance;
        assertEq(steakingBalance, 1 ether);

        // We expect the vault shares to be 2 ether, but actually ends up being 1 ether...
        uint256 wethSteakVaultShares = wethSteakVault.balanceOf(user1);
        assertEq(wethSteakVaultShares, 1 ether);
    }
```

## Recommendations

Consider adding to the existing value of `Steaking::usersToStakes[_onBehalfOf]` by using `+=` rather than `=` to keep track of subsequent stakes for users:

```diff
    def stake(_onBehalfOf: address):
        """
        @notice Allows users to stake ETH for themselves or any other user within the staking period.
        @param _onBehalfOf The address to stake on behalf of.
        """
        assert not self._hasStakingPeriodEnded(), STEAK__STAKING_PERIOD_ENDED
        assert msg.value >= MIN_STAKE_AMOUNT, STEAK__INSUFFICIENT_STAKE_AMOUNT
        assert _onBehalfOf != ADDRESS_ZERO, STEAK__ADDRESS_ZERO

-       self.usersToStakes[_onBehalfOf] = msg.value
+       self.usersToStakes[_onBehalfOf] += msg.value
        self.totalAmountStaked += msg.value

        log Staked(msg.sender, msg.value, _onBehalfOf)
```

    





