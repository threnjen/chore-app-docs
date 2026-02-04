# Phase 2 QA: Approvals

> Parent approval workflow and all assignment statuses

**Last Updated**: February 3, 2026

---

## 👨‍👩‍👧 Parent Approval Workflow

### View Pending Approvals

- [ ] Login as parent
- [ ] Navigate to `/chores/pending`
- [ ] Verify completed chores appear
- [ ] Shows child name and completion time
- [ ] Shows chore title and reward amount
- [ ] Empty state shows "No pending approvals" if none

### Tier Display in Approval Cards

- [ ] BONUS tier: "Early Bonus" badge displayed
- [ ] REDUCED tier: "Late" or "Late completion (50%)" badge displayed
- [ ] Effective reward shown (not base amount) in approval button text
- [ ] Penalty warning shown on reject if penalty configured

### Approve Chore

- [ ] Click "Approve" on a completed chore
- [ ] Verify success toast: "Approved! $X credited"
- [ ] Check child's account balance increased
- [ ] Navigate to Transactions
- [ ] Verify transaction with category "CHORE_REWARD"

### Reject Chore

- [ ] Click "Reject" on a completed chore
- [ ] Reject modal opens
- [ ] Reject requires reason (validation error if empty)
- [ ] "Apply penalty" checkbox if penalty configured (default checked)
- [ ] Submit rejection
- [ ] Verify assignment status shows rejected
- [ ] Verify child can see rejection reason
- [ ] Verify child can re-complete

### Batch Approve

- [ ] Select multiple chores (checkboxes)
- [ ] Click "Approve All"
- [ ] Verify all selected approved

---

## 📊 All Assignment Statuses Testing

Verify each status is correctly handled in the system:

### PENDING (Assigned, waiting to start)

- [ ] Create assigned chore with future due date
- [ ] Verify assignment status = PENDING
- [ ] Verify child sees in "Upcoming" section

### COMPLETED (Child marked done, auto-approved)

- [ ] Complete chore with require_approval=false
- [ ] Verify status = COMPLETED (skips PENDING_APPROVAL)
- [ ] Verify reward credited immediately

### PENDING_APPROVAL (Waiting for parent)

- [ ] Complete chore with require_approval=true
- [ ] Verify status = PENDING_APPROVAL
- [ ] Verify parent sees in approval queue

### APPROVED (Parent approved)

- [ ] Parent approves PENDING_APPROVAL assignment
- [ ] Verify status = APPROVED
- [ ] Verify reward credited

### REJECTED (Parent rejected)

- [ ] Parent rejects PENDING_APPROVAL assignment
- [ ] Verify status = REJECTED
- [ ] Verify rejection_reason is stored
- [ ] Verify child can see reason
- [ ] Verify child can re-complete

### EXPIRED (Passed expiration)

- [ ] Create chore with expiration_days set
- [ ] Let assignment expire (or trigger job)
- [ ] Verify status = EXPIRED
- [ ] Verify penalty applied if configured

**Backend Verification for All Statuses**:
```bash
# View assignments by status
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT status, COUNT(*) FROM chore_assignments GROUP BY status ORDER BY status;"

# View sample of each status
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT DISTINCT ON (ca.status) ca.status, c.title, u.first_name, ca.due_date
   FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id
   LEFT JOIN users u ON ca.assigned_to_id = u.id
   ORDER BY ca.status, ca.created_at DESC;"
```
