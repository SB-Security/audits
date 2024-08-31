## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-inconsistency-between-actual-owners-and-these-in-lastlive) |Inconsistency between actual owners and these in `lastLive` | Medium |

## [M-01] Inconsistency between actual owners and these in `lastLive`

**Description:**

`LivenessModele.removeOwners()` is intended to be used via `SAFE.execTransaction()` and if so `LivenessGuard` will be triggered and will remove the removed owner from the `LivenessGuard.lastLive` mapping, but if `LivenessModele.removeOwners()` is called directly it will not triggers `LivenessGuard` and so the `lastLive` mapping will not be up to date with the current owners, also when a next transaction is executed via `SAFE.execTransaction()` `lastLive` will only be updated if the transaction removes or adds someone, therefore that removed owner/owners that was directly removed by anyone's `LivenessModele.removeOwners()` call, will be present in the `lastLive` mapping, which is inconsistent.

```solidity
function removeOwners(address[] memory _previousOwners, address[] memory _ownersToRemove) external {
    require(_previousOwners.length == _ownersToRemove.length, "LivenessModule: arrays must be the same length");

    // Initialize the ownersCount count to the current number of owners, so that we can track the number of
    // owners in the Safe after each removal. The Safe will revert if an owner cannot be removed, so it is safe
    // keep a local count of the number of owners this way.
    uint256 ownersCount = SAFE.getOwners().length;
    for (uint256 i = 0; i < _previousOwners.length; i++) {
        // Validate that the owner can be removed, which means that either:
        //   1. the ownersCount is now less than MIN_OWNERS, in which case all owners should be removed regardless
        //      of liveness,
        //   2. the owner has not signed a transaction during the liveness interval.
        if (ownersCount >= MIN_OWNERS) {
            require(canRemove(_ownersToRemove[i]), "LivenessModule: the owner to remove has signed recently");
        }

        // Pre-emptively update our local count of the number of owners.
        // This is safe because _removeOwner will bubble up any revert from the Safe if the owner cannot be removed.
        ownersCount--;

        // We now attempt remove the owner from the safe.
        _removeOwner({
            _prevOwner: _previousOwners[i],
            _ownerToRemove: _ownersToRemove[i],
            _newOwnersCount: ownersCount
        });

        // when all owners are removed and the sole owner is the fallback owner, the
        // ownersCount variable will be incorrectly set to zero.
        // This reflects the fact that all prior owners have been removed. The loop should naturally exit at this
        // point, but for safety we detect this condition and force the loop to terminate.
        if (ownersCount == 0) {
            break;
        }
    }
    _verifyFinalState();
}
```

```solidity
/// @notice A mapping of the timestamp at which an owner last participated in signing a
///         an executed transaction, or called showLiveness.
mapping(address => uint256) public lastLive;
```

**Recommendation:** 

Consider deleting anyone who is not the current owner, from `lastLive` at `SAFE.execTransaction()` which can be achieved, by cache all users in `lastLive` on each change and then on each transaction when looping through all `currentOwners`, compare with cached and removing them, this way `lastLive` will be updated for the current owners.