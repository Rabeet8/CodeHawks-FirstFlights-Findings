<!DOCTYPE html>
<html>
<body>

<div class="full-page">
    <div>
    <h1>My Cut Audit Report</h1>
    <h3>Prepared by: Syed Rabeet</h3>
    <h5>Sept 12th,2024</h5>
    </div>

</div>

</body>
</html>

# Disclaimer:

Syed Rabeet makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the solidity implementation of the contracts.


# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

## Scope

```
./src/
-- ContestManager.sol
-- Pot.sol
```

# Protocol Summary 

MyCut is a contest rewards distribution protocol which allows the set up and management of multiple rewards distributions, allowing authorized claimants 90 days to claim before the manager takes a cut of the remaining pool and the remainder is distributed equally to those who claimed in time!

## Roles

* Owner/Admin (Trusted) - Is able to create new Pots, close old Pots when the claim period has elapsed and fund Pots
* User/Player - Can claim their cut of a Pot


## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 1                      |
| Medium   | 0                      |
| Low      | 0                      |
| Info     | 0                      |
| Total    | 1                      |

### Findings

### [H-01] Incorrect Distribution of Unclaimed Rewards in `closePot::Pot` Contract

**Summary:** The  `closePot:Pot` function incorrectly distributes unclaimed rewards, potentially shortchanging users who have claimed them. Instead of allocating all remaining funds (after the manager's cut) to claimants, it distributes only a portion, leaving funds trapped in the contract.

**Vulnerability Details:**

The issue lies in the logic of this function

```solidity
 function closePot() external onlyOwner {
        if (block.timestamp - i_deployedAt < 90 days) {
            revert Pot__StillOpenForClaim();
        }
        if (remainingRewards > 0) {
            uint256 managerCut = remainingRewards / managerCutPercent;
            i_token.transfer(msg.sender, managerCut);

            uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
            for (uint256 i = 0; i < claimants.length; i++) {
                _transferReward(claimants[i], claimantCut);
            }
        }
    }
```

The test case `testUnclaimedRewardDistribution` involves a total reward pool of 1000 tokens. Initially, Player 1 claims 500 tokens, leaving 500 tokens remaining.

10% of the remaining rewards (50 tokens) are allocated to the contest manager. This leaves 450 tokens as unclaimed rewards.

Half of the unclaimed rewards (225 tokens) are returned to Player 1, while the contest retains the other half (225 tokens).

Therefore, Player 1's final balance is 725 tokens (500 initially claimed + 225 unclaimed), not 950 tokens.


```solidity
function testUnclaimedRewardDistribution() public mintAndApproveTokens {
        vm.startPrank(user);
        rewards = [500, 500];
        totalRewards = 1000;
        contest = ContestManager(conMan).createContest(
            players,
            rewards,
            IERC20(ERC20Mock(weth)),
            totalRewards
        );
        ContestManager(conMan).fundContest(0);
        vm.stopPrank();

        vm.startPrank(player1);
        Pot(contest).claimCut();
        vm.stopPrank();


        uint256 claimantBalanceBefore = ERC20Mock(weth).balanceOf(player1);

        vm.warp(91 days);

        vm.startPrank(user);
        ContestManager(conMan).closeContest(contest);
        vm.stopPrank();

        uint256 claimantBalanceAfter = ERC20Mock(weth).balanceOf(player1);
        console.log(ERC20Mock(weth).balanceOf(contest), "contest");
        console.log(
            ERC20Mock(weth).balanceOf(address(conMan)),
            "ContestManager Balance"
        );

        console.log(address(contest));
        assert(claimantBalanceAfter > claimantBalanceBefore);
```

```solidity
ERC20Mock::balanceOf(player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84]) [staticcall]
    │   └─ ← [Return] 725
ERC20Mock::balanceOf(Pot: [0x43e82d2718cA9eEF545A591dfbfD2035CD3eF9c0]) [staticcall]
    │   └─ ← [Return] 225
 ERC20Mock::balanceOf(ContestManager: [0x7BD1119CEC127eeCDBa5DCA7d1Bd59986f6d7353]) [staticcall]
    │   └─ ← [Return] 50
```




**Impact:**

* Users who have claimed their rewards will suffer financial loss as they receive less than their fair share of unclaimed funds.

* Potential permanent loss of funds that remain locked in the contract, reducing the overall efficiency of the reward system.
* Undermines the incentive structure of the contest, potentially discouraging prompt claiming of rewards.

**Tools Used:**

Manual Review

**Recommendations:**



```diff

function closePot() external onlyOwner {
        if (block.timestamp - i_deployedAt < 90 days) {
            revert Pot__StillOpenForClaim();
        }
        if (remainingRewards > 0) {
---            uint256 managerCut = remainingRewards / managerCutPercent;
+++            uint256 managerCut = remainingRewards * managerCutPercent / 100;

            i_token.transfer(msg.sender, managerCut);

---            uint256 claimantCut = (remainingRewards - managerCut) / i_players.length;
+++            uint256 claimantsRewards = remainingRewards - managerCut;

+++          if (claimants.length > 0) {
+++                uint256 rewardPerClaimant = claimantsRewards / claimants.length;
                for (uint256 i = 0; i < claimants.length; i++) {
                    _transferReward(claimants[i], rewardPerClaimant);
                }
+++            } else {
+++                i_token.transfer(msg.sender, claimantsRewards);
++            }

+++        }
+++            remainingRewards = 0;

+++    }
```

Now if we run the `testUnclaimedRewardDistribution`  we will get the appropriate result:

player1 balance is 500 + 450 = 950

The contest Manager balance is 50


```javascript
 ├─ [641] ERC20Mock::balanceOf(player1: [0x7026B763CBE7d4E72049EA67E89326432a50ef84]) [staticcall]
    │   └─ ← [Return] 950
 [641] ERC20Mock::balanceOf(ContestManager: [0x7BD1119CEC127eeCDBa5DCA7d1Bd59986f6d7353]) [staticcall]
    │   └─ ← [Return] 50
```
