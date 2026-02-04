# Phase 2 QA: Edge Cases

> Edge cases and unusual scenarios

**Last Updated**: February 3, 2026

---

## ⚡ Edge Cases

### General Edge Cases

- [ ] Child CAN view `/chores` page but cannot see "New Chore" button
- [ ] Child cannot access `/chores/create` (shows Access Denied)
- [ ] Child cannot approve/reject chores
- [ ] Cannot complete future chores (disabled button)
- [ ] Cannot complete already approved chores
- [ ] Rejected chores can be re-completed by child
- [ ] Deactivating chore cancels future assignments
- [ ] Pending count badge appears in nav for parent

### Chore with No Reward

- [ ] Create chore with reward_amount=0
- [ ] Complete and approve
- [ ] Verify no transaction created (or $0 transaction)
- [ ] Verify UI handles gracefully

### Re-completing Rejected Chore

- [ ] Parent rejects a chore
- [ ] Child views rejection reason
- [ ] Child clicks "Try Again" / completes again
- [ ] Verify new completion timestamp
- [ ] Verify status returns to PENDING_APPROVAL

### Inactive Chore Handling

- [ ] Deactivate a chore (is_active=false)
- [ ] Verify future assignments cancelled
- [ ] Verify existing pending assignments handled
- [ ] Verify chore hidden from child view
- [ ] Verify parent can see with `include_inactive=true`
