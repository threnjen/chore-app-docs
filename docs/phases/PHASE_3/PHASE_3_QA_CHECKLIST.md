# Phase 3 QA Checklist

> Features deferred from Phase 2

**Last Updated**: February 1, 2026

---

## 📋 FIRST_DIBS Assignment Type

### Claim and Complete a Chore (FIRST_DIBS type)

- [ ] Login as child (any)
- [ ] Find an AVAILABLE chore (assignment_type=FIRST_DIBS)
- [ ] Click "Claim" button
- [ ] Verify status changes to CLAIMED
- [ ] Complete the claimed chore
- [ ] Verify status changes to PENDING_APPROVAL or APPROVED

### FIRST_DIBS Type (Claimable) - Full Testing

- [ ] Create chore with assignment_type=FIRST_DIBS
- [ ] Verify assignment created with status=AVAILABLE
- [ ] Verify all eligible children can see the chore
- [ ] **Child A Claims**:
  - [ ] Login as first child
  - [ ] Click "Claim" on available chore
  - [ ] Verify status changes to CLAIMED
  - [ ] Verify chore assigned to this child
- [ ] **Child B Cannot Claim**:
  - [ ] Login as different child
  - [ ] Verify chore no longer shows as claimable
  - [ ] Verify chore shows "Claimed by [Child A]" or hidden

**Backend Verification for FIRST_DIBS**:
```bash
# Check assignment status transitions
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT ca.id, c.title, ca.status, ca.assigned_to_id, u.first_name
   FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id
   LEFT JOIN users u ON ca.assigned_to_id = u.id
   WHERE c.assignment_type='FIRST_DIBS'
   ORDER BY ca.created_at DESC LIMIT 5;"
# Expected: status=AVAILABLE (unclaimed) or CLAIMED (after claim)
```

---

## 📊 FIRST_DIBS Assignment Statuses

### AVAILABLE (First-dibs, unclaimed)
- [ ] Create FIRST_DIBS chore
- [ ] Verify assignment status = AVAILABLE
- [ ] Verify multiple children can see it as claimable

### CLAIMED (First-dibs, claimed by child)
- [ ] Child claims AVAILABLE assignment
- [ ] Verify status changes to CLAIMED
- [ ] Verify assigned_to_id is set

---

## ⚡ FIRST_DIBS Edge Cases

### Concurrent Claim (FIRST_DIBS race condition)
- [ ] Two children try to claim same FIRST_DIBS chore
- [ ] Verify only one succeeds
- [ ] Verify second gets clear error message

---

## 🔧 Implementation Notes

The FIRST_DIBS feature requires the following to be implemented:

### Backend
- [ ] Add `get_available_assignments()` method to fetch AVAILABLE assignments
- [ ] Add `/chore-assignments/available` endpoint to get claimable chores for a child
- [ ] Add `claim_assignment()` method to claim a FIRST_DIBS chore
- [ ] Add `/chore-assignments/{id}/claim` endpoint to claim an assignment
- [ ] Update `/chore-assignments/my` endpoint to optionally include AVAILABLE assignments

### Frontend
- [ ] Add "Available" section to MyChoresPage
- [ ] Add "Claim" button that calls the claim endpoint
- [ ] Handle CLAIMED status display

---

## ✅ Sign-Off

| Area | Tester | Date | Status |
|------|--------|------|--------|
| FIRST_DIBS Claiming | | | ⬜ |
| AVAILABLE/CLAIMED Statuses | | | ⬜ |
| Concurrent Claim Edge Case | | | ⬜ |

---

## Notes

| Issue | Severity | Status | Notes |
|-------|----------|--------|-------|
| | | | |
