reported at 2026/05/07

## [M-1] Third parties can dilute optimistic veto thresholds by inflating vault share supply before the veto snapshot

### Summary

Optimistic proposals use a delayed veto snapshot. During the delay, `StakingVault` deposits remain permissionless. A third party that does not have `OPTIMISTIC_PROPOSER_ROLE` can deposit underlying tokens before the veto snapshot, mint new vault shares, and increase the `pastTotalSupply` used as the veto denominator. This raises the absolute number of `Against` votes required to veto an already-created optimistic proposal.

The attack does not require the third party to submit a proposal, vote, or delegate optimistic voting power. It only requires temporary capital across the proposal's veto snapshot.

### Severity

**Severity:** Medium

| Impact | Likelihood | Result |
| --- | --- | --- |
| Medium | Medium | Medium |

**Impact:** Medium. A proposal that should have been vetoed and transitioned into a full standard confirmation vote can instead remain unvetoed, reach `Succeeded`, and execute through the optimistic bypass path. The direct impact depends on the allowlisted target and selector used by the optimistic proposal.

**Likelihood:** Medium. The attack is permissionless once an optimistic proposal exists, but it requires enough temporary underlying liquidity to mint the needed vault shares before the veto snapshot. The cost is lower when `unstakingDelay` is zero or short, and higher when the vault enforces a meaningful withdrawal delay.

### Source Code Links

- Optimistic proposal snapshots are delayed until `block.timestamp + vetoDelay`: [ProposalLib.sol#L183-L185](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/lib/ProposalLib.sol#L183-L185)
- Veto threshold uses `token().getPastTotalSupply(snapshot)` as the denominator: [ReserveOptimisticGovernor.sol#L229-L250](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L229-L250)
- Optimistic veto votes use optimistic delegated votes as the numerator: [ReserveOptimisticGovernor.sol#L392-L403](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/ReserveOptimisticGovernor.sol#L392-L403)
- ERC4626 deposits mint vault shares: [StakingVault.sol#L246-L254](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L246-L254)
- Zero `unstakingDelay` allows immediate withdrawals: [StakingVault.sol#L269-L270](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L269-L270)
- Optimistic proposals execute through the timelock bypass path once they succeed: [TimelockControllerOptimistic.sol#L73-L92](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/governance/TimelockControllerOptimistic.sol#L73-L92)
- The README describes the veto threshold as based on `pastTotalSupply`: [README.md#L160-L172](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/README.md#L160-L172)

### Vulnerability Details

For optimistic proposals, `_saveProposal()` stores `voteStart` as `block.timestamp + voteDelay`. This means the supply used for the veto calculation is not fixed when the proposal is created. Any vault share minting before `voteStart` is included in `token().getPastTotalSupply(snapshot)`.

`ReserveOptimisticGovernor.state()` then computes:

```solidity
uint256 pastSupply = token().getPastTotalSupply(snapshot);
uint256 vetoThresholdTok = (_vetoThreshold * pastSupply) / 1e18;
```

However, when an account casts an optimistic veto vote, the voting weight is read from optimistic delegation checkpoints:

```solidity
uint256 totalWeight =
  _isOptimistic(proposalId) ? _getOptimisticVotes(account, snapshot) : _getVotes(account, snapshot, params);
```

Therefore, the denominator is total vault share supply, while the numerator is only the delegated optimistic voting weight that actually votes `Against`. This creates a dilution vector: minting extra shares before the snapshot increases the threshold even if those shares never vote.

Let:

```text
S = vault share totalSupply before the proposal
B = attacker shares minted before the veto snapshot
r = vetoThreshold ratio
H = honest Against votes available at the snapshot
```

The veto succeeds only if:

```text
H >= r * (S + B)
```

If honest voters originally had enough votes to meet `r * S`, a third party can deposit before the snapshot so that `r * (S + B)` exceeds `H`. The optimistic proposal then remains active until the veto period ends and can succeed without entering the standard confirmation path.

This is not a same-transaction flash loan attack because the snapshot must become a past timestamp. The relevant risk is cross-snapshot temporary capital.

### Attack Scenario

1. A legitimate optimistic proposer creates a fast proposal using an allowlisted target and selector.
2. The proposal is pending until `voteStart = block.timestamp + vetoDelay`.
3. A third party who benefits from the proposal deposits a large amount of underlying into `StakingVault`.
4. The deposit mints new vault shares before the veto snapshot.
5. At `voteStart`, `pastTotalSupply` includes the third party's newly minted shares.
6. Honest voters cast `Against`, but their votes no longer reach the inflated veto threshold.
7. The proposal does not transition into a standard confirmation vote.
8. After the veto period expires, the proposal becomes `Succeeded` and can execute through the optimistic bypass.
9. If `unstakingDelay` is zero, the third party can withdraw after the snapshot with minimal capital lockup.

### Proof of Concept

The PoC was verified in `test/POC.t.sol`.

Run:

```bash
forge test --match-path test/POC.t.sol --match-test test_optimisticVetoThresholdCanBeDilutedByThirdPartyDeposit -vvv
```

Result:

```text
Ran 1 test for test/POC.t.sol:OptimisticVetoDilutionPOC
[PASS] test_optimisticVetoThresholdCanBeDilutedByThirdPartyDeposit()
Suite result: ok. 1 passed; 0 failed; 0 skipped
```

The test first proves the baseline behavior: without third-party supply inflation, Alice's `400_000e18` `Against` vote exceeds the 20% veto threshold on the original `1_000_000e18` supply and transitions the optimistic proposal to `Defeated`.

The test then creates a second optimistic proposal. A third party with no `OPTIMISTIC_PROPOSER_ROLE` deposits `1_100_000e18` underlying before the veto snapshot. The snapshot supply becomes `2_100_000e18`, so the 20% veto threshold becomes `420_000e18`. Alice's same `400_000e18` `Against` vote no longer reaches the threshold, the proposal stays `Active`, and then becomes `Succeeded` after the veto period.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import { IGovernor } from "@openzeppelin/contracts/governance/IGovernor.sol";
import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import { OPTIMISTIC_PROPOSER_ROLE } from "@utils/Constants.sol";

import { ReserveOptimisticGovernorTestBase } from "./ReserveOptimisticGovernor.t.sol";

contract OptimisticVetoDilutionPOC is ReserveOptimisticGovernorTestBase {
  function _useExistingStakingVaultDeployment() internal pure override returns (bool) {
    return false;
  }

  function test_optimisticVetoThresholdCanBeDilutedByThirdPartyDeposit() public {
    uint256 transferAmount = 1_000e18;

    (address[] memory controlTargets, uint256[] memory controlValues, bytes[] memory controlCalldatas) =
      _singleCall(address(underlying), 0, abi.encodeCall(IERC20.transfer, (alice, transferAmount)));

    underlying.mint(address(timelock), transferAmount);

    vm.prank(optimisticProposer);
    uint256 controlProposalId =
      governor.proposeOptimistic(controlTargets, controlValues, controlCalldatas, "control optimistic veto");

    _warpToActive(controlProposalId);

    vm.prank(alice);
    governor.castVote(controlProposalId, 0);

    assertEq(uint256(governor.state(controlProposalId)), uint256(IGovernor.ProposalState.Defeated));

    address attacker = makeAddr("vetoDiluter");
    uint256 attackerDeposit = 1_100_000e18;

    assertFalse(timelock.hasRole(OPTIMISTIC_PROPOSER_ROLE, attacker));

    (address[] memory targets, uint256[] memory values, bytes[] memory calldatas) =
      _singleCall(address(underlying), 0, abi.encodeCall(IERC20.transfer, (alice, transferAmount)));

    underlying.mint(address(timelock), transferAmount);

    vm.prank(optimisticProposer);
    uint256 dilutedProposalId =
      governor.proposeOptimistic(targets, values, calldatas, "diluted optimistic veto");

    underlying.mint(attacker, attackerDeposit);
    vm.startPrank(attacker);
    underlying.approve(address(stakingVault), attackerDeposit);
    stakingVault.deposit(attackerDeposit, attacker);
    vm.stopPrank();

    _warpToActive(dilutedProposalId);

    assertEq(stakingVault.getPastTotalSupply(governor.proposalSnapshot(dilutedProposalId)), 2_100_000e18);

    vm.prank(alice);
    governor.castVote(dilutedProposalId, 0);

    (uint256 againstVotes,,) = governor.proposalVotes(dilutedProposalId);
    assertEq(againstVotes, ALICE_STAKE);
    assertEq(uint256(governor.state(dilutedProposalId)), uint256(IGovernor.ProposalState.Active));

    _warpPastDeadline(dilutedProposalId);

    assertEq(uint256(governor.state(dilutedProposalId)), uint256(IGovernor.ProposalState.Succeeded));
  }
}
```

### Recommendation

Do not let post-proposal share minting increase the veto denominator for an already-created optimistic proposal. Possible fixes include:

- Compute each optimistic proposal's veto denominator from a supply snapshot fixed at proposal creation.
- Use a stake-age or lookback requirement so shares minted shortly before the veto snapshot are excluded from the veto denominator.
- Use a rolling minimum or time-weighted supply over a pre-proposal window instead of raw total supply at `voteStart`.
- Require a meaningful unstaking delay or cooldown when optimistic governance relies on staked supply as its security budget.
- Monitor large supply changes between optimistic proposal creation and the veto snapshot and cancel suspicious optimistic proposals through the guardian path.

## [L-1] Pending rewards can be captured by late deposits before stale balance snapshots are synchronized

### Summary

This issue is based on the current post-fix reward accounting. The original implementation used the live `IERC20(asset()).balanceOf(address(this))` in `_currentAccountedNativeRewards()`, which caused freshly transferred native rewards to be distributed as if they had been present during the whole elapsed period. The current implementation replaced that live-balance calculation with `nativeBalanceLastKnown` snapshot accounting.

That change fixes the old over-distribution direction, but it introduces a new stale-snapshot entry window. `StakingVault` supports two reward paths: direct transfers of the underlying asset to the vault, which are streamed into `totalAssets()`, and registered ERC20 reward tokens, which are streamed through `rewardIndex`. Both paths now use cached balance snapshots instead of the live token balance for the current accrual calculation.

When fresh rewards are transferred directly to the vault, the first interaction after the top-up only synchronizes the cached balance. It does not distribute the new top-up in that same accrual. A searcher can back-run the top-up with a `deposit()`, mint shares before the reward is priced in, and then receive part of the pending reward stream on the next accrual cycle even though they were not staked when the reward arrived.

### Severity

**Severity:** Medium

| Impact | Likelihood | Result |
| --- | --- | --- |
| Medium | Medium | Medium |

**Impact:** Medium. Existing stakers lose part of native-asset rewards and ERC20 reward-token distributions to late depositors. The loss is proportional to the attacker's share of supply after entering and can repeat for every public reward top-up.

**Likelihood:** Medium. The attack is permissionless when rewards are funded by direct token transfers and the top-up is observable or predictable. The cost is the temporary capital needed to deposit before the first post-top-up accrual and remain through the reward streaming window.

### Preconditions

This finding assumes direct reward top-ups are intended to benefit the stake set that exists when the rewards arrive or are synchronized, not deposits made after the top-up is already observable. It applies to ERC20 `asset()` transfers sent directly to the vault and to registered ERC20 reward-token transfers. It does not apply when a user calls `deposit()`, because `deposit()` is ordinary staking: it transfers the underlying asset, mints shares, and increases `totalDeposited`.

If the intended design is that any rewards transferred to the vault are open to all future depositors until the next accrual cycle, then the behavior should be documented as an intentional reward-streaming rule and the issue should be treated as a lower-severity specification/fairness concern rather than a clear value-leak vulnerability.

### Source Code Links

- `totalAssets()` prices shares from `totalDeposited` plus cached native rewards, not the live underlying balance: [StakingVault.sol#L240-L249](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L240-L249)
- Deposits accrue after ERC4626 has already calculated the shares to mint: [StakingVault.sol#L252-L260](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L252-L260)
- Native reward accrual updates `nativeBalanceLastKnown` only after using the previous snapshot: [StakingVault.sol#L401-L430](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L401-L430)
- ERC20 reward-token accrual snapshots the fresh balance before computing handout from the old balance: [StakingVault.sol#L433-L452](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/contracts/staking/StakingVault.sol#L433-L452)
- The repository test confirms native asset donations are only snapshotted on the first poke and start distributing on the next iteration: [StakingVault.t.sol#L942-L962](https://github.com/reserve-protocol/reserve-governor/blob/dc27a68463cec356ff18bbdd3d8edfe9b2534372/test/StakingVault.t.sol#L942-L962)

### Vulnerability Details

This is a second-order issue introduced by the balance snapshot fix for the prior native reward bug. In the old code, `_currentAccountedNativeRewards()` effectively used:

```solidity
uint256 rewardsBalance = IERC20(asset()).balanceOf(address(this)) - totalDeposited;
```

That meant a direct transfer immediately entered the reward calculation and could be over-counted for time that had already elapsed. The current code avoids that by using `nativeBalanceLastKnown`, but the new snapshot is only synchronized after accrual logic runs.

Native asset rewards are expected to be funded by directly transferring the vault's underlying token to the vault address. This is different from calling `deposit()`: a direct transfer increases the vault's token balance without minting shares, while `deposit()` is ordinary staking that mints shares and increases `totalDeposited`.

The vault's share price is based on:

```solidity
function totalAssets() public view override returns (uint256) {
    return totalDeposited + _currentAccountedNativeRewards();
}

function _currentAccountedNativeRewards() internal view returns (uint256) {
    uint256 elapsed = block.timestamp - nativeRewardsLastPaid;
    uint256 rewardsBalance = nativeBalanceLastKnown - totalDeposited;
    return _calculateHandout(rewardsBalance, elapsed);
}
```

This ignores any underlying tokens that were just transferred to the vault but have not yet been synchronized into `nativeBalanceLastKnown`. ERC4626 `deposit()` calculates shares using `previewDeposit()` before `_deposit()` runs. `_deposit()` has the `accrueRewards` modifier, so the fresh donation is snapshotted during the deposit, but the attacker has already received the stale-price share amount.

The same stale snapshot pattern exists for ERC20 reward tokens:

```solidity
uint256 balanceLastKnown = rewardInfo.balanceLastKnown;
rewardInfo.balanceLastKnown = IERC20(_rewardToken).balanceOf(address(this)) + rewardInfo.totalClaimed;

uint256 elapsed = block.timestamp - rewardInfo.payoutLastPaid;
uint256 unaccountedBalance = balanceLastKnown - rewardInfo.balanceAccounted;
uint256 tokensToHandout = _calculateHandout(unaccountedBalance, elapsed);
```

The fresh top-up is written into `rewardInfo.balanceLastKnown`, but `tokensToHandout` is computed from the old `balanceLastKnown`. Therefore the first post-top-up accrual records the new balance but excludes it from the current handout. Late shares minted before this synchronization participate in the later `rewardIndex` growth.

### Attack Scenario

1. Alice is the only staker with `1,000` vault shares and `totalAssets() == 1,000`.
2. A reward funder directly transfers `1,000` underlying tokens to the vault as native rewards.
3. The vault's live token balance is now `2,000`, but `totalAssets()` still returns `1,000` because `nativeBalanceLastKnown` has not been updated.
4. A searcher back-runs the top-up and calls `deposit(1,000)`.
5. ERC4626 calculates the deposit using the stale `totalAssets()`, so the searcher receives about `1,000` shares instead of fewer shares at the reward-adjusted price.
6. The deposit's accrual path synchronizes the fresh direct transfer into `nativeBalanceLastKnown`, but the new reward is not streamed until a later accrual cycle.
7. After one reward half-life, about `500` of the `1,000` pending reward tokens are streamed into accounting.
8. Alice and the searcher each hold about half of the vault shares, so the searcher captures about `250` of the first streamed reward amount despite not being staked when the reward arrived.

The ERC20 reward-token variant follows the same sequence. The attacker deposits after a reward-token top-up but before the first post-top-up accrual snapshots the fresh reward balance. On the next accrual cycle, the attacker receives a pro-rata share of the `rewardIndex` increase.

### Existing Evidence

The existing test `test__accrual_nativeAssetRewardsDonationIsDelayedByOneIteration()` demonstrates the necessary accounting delay:

```solidity
_mintAndDepositFor(ACTOR_ALICE, 1000e18);

_payoutRewards(1);
token.mint(address(vault), 1000e18);

vault.poke();
uint256 totalAssetsAfterFirstPoke = vault.totalAssets();
assertEq(totalAssetsAfterFirstPoke, initialTotalAssets);

_payoutRewards(1);
vault.poke();
uint256 totalAssetsAfterSecondPoke = vault.totalAssets();
assertApproxEqRel(totalAssetsAfterSecondPoke, 1500e18, 0.001e18);
```

The first `poke()` after the direct transfer only snapshots the donation. Distribution starts on the next iteration. This test does not by itself assert the attacker's final profit; it verifies the stale-snapshot window that makes the attack possible. Inserting an attacker `deposit()` between the direct transfer and the first effective distribution gives the attacker shares before the reward is priced in, allowing them to capture part of the delayed stream.

### Recommendation

Remove the stale-balance entry window by synchronizing reward accounting when rewards are received, not only on the next unrelated vault interaction.

For this vault's current reward paths, the relevant assets are ERC20 tokens. A Solidity payable `receive()` function does not run when an ERC20 `transfer()` is sent to the vault, so `receive()` alone cannot make direct ERC20 top-ups update accounting. The primary fix should be an explicit ERC20 reward receiving/sync entrypoint, such as `notifyRewardAmount()` / `depositReward()` / `syncRewards()`, that reward funders call atomically with the token transfer or that pulls the tokens via `transferFrom()`.

If the protocol later supports native ETH-style rewards, a payable `receive()` function can delegate to the same synchronization logic for ETH transfers. That is an ETH-specific complement, not a replacement for the ERC20 reward entrypoint.

The important property is that direct top-ups must update the relevant snapshot before a late depositor can mint shares at a stale price. A robust fix should:

- Accrue existing rewards before accepting a new top-up.
- Update `nativeBalanceLastKnown` or `rewardInfo.balanceLastKnown` in the same transaction that receives the reward.
- Ensure deposits and mints cannot receive shares at a price that excludes live top-ups already held by the vault.
- Track new reward buckets with an eligibility timestamp or supply snapshot so shares minted after a top-up cannot receive that top-up's future stream.