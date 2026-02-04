# Phase 2 QA: Child Workflow

> Child chore view, completion flow, and age-appropriate UI

**Last Updated**: February 3, 2026

---

## 🧒 Child Chore Workflow

### View My Chores

- [x] Login as child user (child1@test.com)
- [x] Navigate to `/chores/my`
- [x] Verify "Today" section shows due chores
- [x] Verify "Overdue" section if any (with days count badge)
- [x] Verify "Upcoming" section shows future chores
- [x] Verify "Waiting" section shows pending approval
- [x] Verify "Done" section shows approved
- [x] Verify "Rejected" section shows Try Again

### Complete a Chore (ASSIGNED type)

- [x] Find a PENDING chore assigned directly to child
- [x] Click "Done"/"Complete" button
- [x] Completion confirmation shown
- [x] Verify status changes to "Waiting for Approval" (if require_approval=true)
- [x] Verify chore moved to appropriate section

### Complete Overdue Chore

- [x] Find or create an overdue chore
- [x] Verify overdue badge displays (shows days count)
- [x] Complete the chore
- [x] Verify `reward_tier` is REDUCED (if grace configured)

### Complete Early

- [ ] Find chore due in future (2+ days)
- [ ] Complete it now
- [ ] Verify `reward_tier` is BONUS (if early_bonus configured)

### Auto-Approve Flow

- [x] Find chore with auto_approve_hours set (e.g., 24 hours)
- [x] Complete the chore
- [x] Verify status shows PENDING_APPROVAL
- [x] Wait for auto-approve job (or trigger manually):
  ```bash
  docker-compose exec api python -c "
  import asyncio
  from app.jobs.chore_jobs import auto_approve_stale_assignments
  asyncio.run(auto_approve_stale_assignments())
  "
  ```
- [ ] Verify status changed to APPROVED
- [ ] Verify reward credited to account

### Photo Required Chore

- [ ] Find chore with photo_required=true
- [ ] Try to complete without photo URL
- [ ] Verify error message about photo requirement
- [ ] Complete with photo URL
- [ ] Verify success

### Rejected Chore

- [ ] Find chore with Rejected Status
- [ ] Press "Try Again" button

---

## 👶 Age-Appropriate UI Testing

### Youngest Child (6 years old - child4@test.com)

- [ ] Login as child4
- [ ] Verify simplified UI elements
- [ ] Verify large, easy-to-tap buttons
- [ ] Verify minimal text, more icons
- [ ] Verify star/emoji feedback on completion

### Middle Child (8-10 years old - child2@test.com, child1@test.com)

- [ ] Login as child1 or child2
- [ ] Verify age-appropriate complexity
- [ ] Verify reward amounts visible
- [ ] Verify simple descriptions readable

### Teenager (16 years old - child5@test.com)

- [ ] Login as child5
- [ ] Verify full-featured UI
- [ ] Verify detailed information visible
- [ ] Verify can handle complex chore descriptions
