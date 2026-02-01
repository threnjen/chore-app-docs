# Phase 2 Comprehensive QA Checklist

> Manual testing checklist for Phase 2: Chores MVP

**Last Updated**: January 31, 2026

---

## Prerequisites
- Docker Desktop running
- Backend: `docker-compose up` or `uvicorn app.main:app --reload`
- Database migrated: `docker-compose exec api alembic upgrade head`
- Seed data loaded: `docker-compose exec api python scripts/seed_dev_data.py`
- Frontend: `npm run dev` (in chore-app-frontend)
- DEV_MODE enabled (emails logged to console, not sent)
- Run QA on page http://localhost:5173/
 
### Resetting all data/containers
- `docker-compose down -v`
- `docker-compose up`
- `docker-compose exec api alembic upgrade head`
- `docker-compose exec api python scripts/seed_dev_data.py`

---

## 🔑 Test Accounts (Seed Data)

All accounts use password: `TestPass123`

### Parents
| Email | Name | Family | Role |
|-------|------|--------|------|
| parent1@test.com | Alice Parent | Parent Family | OWNER |
| parent2@test.com | Bob Parent | Parent Family | ADULT |
| parent3@test.com | Carol Parent | Third Family | OWNER |

### Children
| Email | Name | Age | Family | Account Types |
|-------|------|-----|--------|---------------|
| child1@test.com | Charlie Child | 10 | Parent Family | SPEND, SAVE, GIVE |
| child2@test.com | Diana Child | 8 | Parent Family | SPEND, SAVE |
| child3@test.com | Eddie Multi-Family | 13 | Parent Family + Third Family | SPEND only |
| child4@test.com | Fiona Youngest | 6 | Parent Family | SPEND only |
| child5@test.com | George Teen | 16 | Third Family | SPEND, SAVE, GIVE |

### Test Scenarios by User
- **child1**: Standard middle-age child, all account types
- **child2**: Younger child, limited accounts
- **child3**: Multi-household child (belongs to 2 families)
- **child4**: Youngest child (6yo), simplest setup
- **child5**: Teenager, Third Family only

---

## 🗃️ Database Access (Backend Verification)

### Quick Access Commands

```bash
# Connect to PostgreSQL CLI (interactive mode)
docker-compose exec db psql -U picklesapp -d picklesapp

# Run a single query and exit
docker-compose exec db psql -U picklesapp -d picklesapp -c "SELECT * FROM users LIMIT 5;"

# Pretty-print with expanded display
docker-compose exec db psql -U picklesapp -d picklesapp -c "\x" -c "SELECT * FROM users WHERE email='test@test.com';"
```

### Useful psql Commands (inside psql shell)
```sql
\dt                    -- List all tables
\d users               -- Describe users table (show columns)
\d+ users              -- Describe with more detail
\x                     -- Toggle expanded display (easier to read)
\q                     -- Quit psql
```

### Common Verification Queries

```sql
-- View all users
SELECT id, email, first_name, last_name, role, email_verified, created_at 
FROM users ORDER BY created_at DESC;

-- View a specific user by email
SELECT * FROM users WHERE email = 'test@example.com';

-- View all families
SELECT id, name, timezone, created_at FROM families;

-- View family memberships (who belongs to which family)
SELECT fm.id, u.email, u.first_name, f.name as family_name, fm.role 
FROM family_memberships fm
JOIN users u ON fm.user_id = u.id
JOIN families f ON fm.family_id = f.id;

-- View join requests / invitations
SELECT i.id, i.invitation_type, i.status, 
       requester.email as requester_email,
       target.email as target_email,
       f.name as family_name,
       i.message, i.created_at, i.expires_at
FROM invitations i
LEFT JOIN users requester ON i.requester_id = requester.id
LEFT JOIN users target ON i.target_email = target.email
LEFT JOIN families f ON i.family_id = f.id
ORDER BY i.created_at DESC;

-- View chores
SELECT id, title, reward_amount, is_active, created_at 
FROM chores ORDER BY created_at DESC;

-- View chore assignments
SELECT ca.id, c.title, u.first_name as assigned_to, ca.status, 
       ca.due_date, ca.reward_tier, ca.effective_reward
FROM chore_assignments ca
JOIN chores c ON ca.chore_id = c.id
LEFT JOIN users u ON ca.assigned_to_id = u.id
ORDER BY ca.due_date DESC;

-- View accounts
SELECT a.id, u.first_name as owner, a.account_type, a.balance 
FROM accounts a
JOIN users u ON a.owner_id = u.id;

-- View transactions
SELECT t.id, t.amount, t.transaction_type, t.category, t.description, t.created_at
FROM transactions t
ORDER BY t.created_at DESC LIMIT 10;
```

---

## 🏠 Parent Chore Management

### Create Chore

- [ ] Navigate to `/chores` from sidebar
- [ ] Click "New Chore" button
- [ ] **Form Fields**:
  - [ ] Enter title "Test Daily Chore"
  - [ ] Enter description "Test description"
  - [ ] Set reward amount $5.00
  - [ ] Select child to assign to
  - [ ] Select frequency "Daily"
  - [ ] Verify weekday picker hidden for Daily
  - [ ] Select frequency "Weekly"
  - [ ] Verify weekday picker appears
  - [ ] Select M-W-F
  - [ ] Can set interval (every 2 weeks)
  - [ ] Can toggle between Absolute and Relative timing mode
  - [ ] Verify preview shows next 5 occurrences correctly
  - [ ] Set deadline time (optional)
  - [ ] Set estimated duration (optional)
  - [ ] Toggle "Require Photo"
  - [ ] Toggle "Require Approval"
  - [ ] Set auto-approve after hours (optional)
  - [ ] Toggle "Show Advanced" mode
  - [ ] Set grace period (1 day)
  - [ ] Set late reward percentage (50%)
  - [ ] Set early bonus ($1.00)
  - [ ] Set penalty behavior (On Expiration)
  - [ ] Set penalty amount ($1.00)
- [ ] Submit form
- [ ] Verify redirect to `/chores`
- [ ] Verify chore appears in list
- [ ] Verify initial assignments generated (check via API or database)

### List Chores

- [ ] Verify all chores display correctly
- [ ] Verify reward badges show correctly
- [ ] Verify penalty badges (if configured)
- [ ] Verify recurrence pattern descriptions
- [ ] Verify assignment count displays
- [ ] Can filter by active/inactive
- [ ] Can click to view chore details

### View/Edit Chore

- [ ] Click on a chore to view details
- [ ] Verify `/chores/{id}` page loads
- [ ] Click "Edit" button
- [ ] Modify title
- [ ] Save changes
- [ ] Verify update reflected

### Delete Chore

- [ ] Click "Delete" on a chore
- [ ] Confirm deletion (with confirmation dialog)
- [ ] Verify chore removed (soft-delete)
- [ ] Verify with `include_inactive=true` query

---

## 🧒 Child Chore Workflow

### View My Chores

- [ ] Login as child user (child1@test.com)
- [ ] Navigate to `/chores/my`
- [ ] Verify "Today" section shows due chores
- [ ] Verify "Overdue" section if any (with days count badge)
- [ ] Verify "Upcoming" section shows future chores
- [ ] Verify "Waiting" section shows pending approval
- [ ] Verify "Done" section shows approved

### Complete a Chore (ASSIGNED type)

- [ ] Find a PENDING chore assigned directly to child
- [ ] Click "Done"/"Complete" button
- [ ] Completion confirmation shown
- [ ] Verify status changes to "Waiting for Approval" (if require_approval=true)
- [ ] Verify chore moved to appropriate section

### Claim and Complete a Chore (FIRST_DIBS type)

- [ ] Login as child (any)
- [ ] Find an AVAILABLE chore (assignment_type=FIRST_DIBS)
- [ ] Click "Claim" button
- [ ] Verify status changes to CLAIMED
- [ ] Complete the claimed chore
- [ ] Verify status changes to PENDING_APPROVAL or APPROVED

### Complete Overdue Chore

- [ ] Find or create an overdue chore
- [ ] Verify overdue badge displays (shows days count)
- [ ] Complete the chore
- [ ] Verify `reward_tier` is REDUCED (if grace configured)

### Complete Early

- [ ] Find chore due in future (2+ days)
- [ ] Complete it now
- [ ] Verify `reward_tier` is BONUS (if early_bonus configured)

### Auto-Approve Flow

- [ ] Find chore with auto_approve_hours set (e.g., 24 hours)
- [ ] Complete the chore
- [ ] Verify status shows PENDING_APPROVAL
- [ ] Wait for auto-approve job (or trigger manually):
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

---

## 👨‍👩‍👧 Parent Approval Workflow

### View Pending Approvalsswed

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

## 💰 Tiered Rewards Testing

### Full Reward (On-Time)

- [ ] Create chore with $10 reward
- [ ] Complete before due date
- [ ] Approve
- [ ] Verify $10 credited

### Bonus Reward (Early)

- [ ] Create chore with $10 reward + $2 early bonus + 24h threshold
- [ ] Complete 48+ hours early
- [ ] Approve
- [ ] Verify $12 credited (effective_reward)

### Reduced Reward (Late)

- [ ] Create chore with $10 reward + 1 day grace + 50% late percentage
- [ ] Complete 2 days after due
- [ ] Approve
- [ ] Verify $5 credited (50% of reward)

**Backend Verification for Tiered Rewards**:
```bash
# Check assignment reward tier after completion
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT ca.id, c.title, ca.status, ca.reward_tier, ca.effective_reward, c.reward_amount
   FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id
   ORDER BY ca.completed_at DESC NULLS LAST LIMIT 5;"
# Expected: reward_tier=BONUS/FULL/REDUCED, effective_reward calculated

# Check transaction amount matches effective_reward
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT t.amount, t.description FROM transactions t 
   WHERE t.category='CHORE_REWARD' ORDER BY t.created_at DESC LIMIT 3;"
```

---

## 🎯 Reward Types Testing

### MONEY Reward Type

- [ ] Create chore with reward_type=MONEY, reward_amount=5.00
- [ ] Complete and approve
- [ ] Verify transaction shows dollar amount
- [ ] Verify child's SPEND account balance increases by $5.00

### POINTS Reward Type

- [ ] Create chore with reward_type=POINTS, reward_amount=100
- [ ] Complete and approve
- [ ] Verify transaction shows points (not dollars)
- [ ] Verify UI displays "100 points" not "$100.00"
- [ ] Verify points tracked correctly

**Backend Verification for POINTS**:
```bash
# Check chore reward type
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT id, title, reward_type, reward_amount FROM chores WHERE reward_type='POINTS';"

# Check assignment effective_reward for points
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT ca.effective_reward, c.reward_type FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id WHERE c.reward_type='POINTS';"
```

### STARS Reward Type

- [ ] Create chore with reward_type=STARS, reward_amount=5
- [ ] Complete and approve
- [ ] Verify UI displays "5 stars" with star icon
- [ ] Verify stars tracked correctly

### Mixed Reward Types in Approval Queue

- [ ] Have pending approvals with MONEY, POINTS, and STARS
- [ ] Verify approval queue shows correct icons/labels for each type
- [ ] Verify batch approve handles different types correctly

---

## ⚠️ Penalty Testing

### Penalty on Rejection

- [ ] Create chore with penalty_behavior=ON_REJECTION, penalty=$2
- [ ] Complete the chore
- [ ] Reject with reason
- [ ] Verify $2 debited from child account
- [ ] Verify transaction category is CHORE_PENALTY

### Penalty on Expiration

- [ ] Create chore with penalty_behavior=ON_EXPIRATION, expiration_days=1
- [ ] Let chore expire (or trigger expire job manually)
- [ ] Verify assignment status is EXPIRED
- [ ] Verify penalty applied

### Escalating Penalty

- [ ] Create chore with base penalty $1 + $0.50/day + max $5
- [ ] Let expire 10 days overdue
- [ ] Verify effective_penalty = $5 (capped)

**Backend Verification for Penalties**:
```bash
# Check chore penalty configuration
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT id, title, penalty_behavior, penalty_amount, penalty_per_day, max_penalty
   FROM chores WHERE penalty_amount > 0;"

# Check assignment status and penalty
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT ca.id, c.title, ca.status, ca.effective_penalty, ca.rejection_reason
   FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id
   WHERE ca.status IN ('REJECTED', 'EXPIRED')
   ORDER BY ca.updated_at DESC LIMIT 5;"

# Check penalty transaction was created
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT t.amount, t.transaction_type, t.category, t.description 
   FROM transactions t WHERE t.category='CHORE_PENALTY' 
   ORDER BY t.created_at DESC LIMIT 3;"
# Expected: transaction_type=DEBIT, category=CHORE_PENALTY

# Manually trigger expire job (if needed)
docker-compose exec api python -c "
import asyncio
from app.jobs.chore_jobs import expire_overdue_assignments
asyncio.run(expire_overdue_assignments())
"
```

---

## 🔄 Recurrence Testing

### One-Time Chore (No Recurrence)

- [ ] Create chore without recurrence rule
- [ ] Verify only single assignment created
- [ ] Preview shows just the one date
- [ ] Complete and approve
- [ ] Verify no future assignments generated

### Daily Pattern

- [ ] Create daily chore
- [ ] Verify assignments created for next 7 days
- [ ] Preview shows consecutive days

**Backend Verification for Recurrence**:
```bash
# Check chore recurrence rule
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT id, title, recurrence_rule FROM chores ORDER BY created_at DESC LIMIT 3;"

# Check generated assignments
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT ca.due_date, ca.status, c.title
   FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id
   WHERE c.title LIKE '%YOUR_CHORE_TITLE%'
   ORDER BY ca.due_date LIMIT 10;"
```

### Weekly M-W-F Pattern

- [ ] Create weekly M-W-F chore
- [ ] Preview shows only Mon/Wed/Fri dates

### Bi-Weekly Pattern

- [ ] Create chore with interval=2, weekly
- [ ] Preview shows every other week

### Monthly Pattern

- [ ] Create monthly on 15th
- [ ] Preview shows 15th of each month

### Last Day of Month

- [ ] Create monthly on by_monthday=[-1]
- [ ] Verify handles Feb correctly

### Timing Mode: ABSOLUTE vs RELATIVE

**ABSOLUTE timing** (default):
- [ ] Create chore with timing_mode=ABSOLUTE
- [ ] Complete chore on Day 1
- [ ] Next occurrence is still Day 2 (fixed schedule)

**RELATIVE timing**:
- [ ] Create chore with timing_mode=RELATIVE
- [ ] Complete chore on Day 1 (due originally Day 1)
- [ ] Verify next occurrence calculated from completion date
- [ ] Example: Complete 2 days late → next due shifts by 2 days

---

## 🔐 Permission Testing

### Child Restrictions

- [ ] Login as child
- [ ] Verify cannot access `/chores/create`
- [ ] Verify cannot see "New Chore" button
- [ ] Verify cannot approve/reject chores
- [ ] Verify can only see own assignments

### Parent Access

- [ ] Login as parent
- [ ] Verify can see all family assignments
- [ ] Verify can create/edit/delete chores
- [ ] Verify can approve/reject

---

## 📋 Assignment Types Testing

### ASSIGNED Type (Direct Assignment)

- [ ] Create chore with assignment_type=ASSIGNED
- [ ] Select specific child(ren) to assign
- [ ] Verify assignments created with status=PENDING
- [ ] Verify assigned children see chore in their list
- [ ] Verify non-assigned children do NOT see chore

### FIRST_DIBS Type (Claimable)

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

## 👨‍👩‍👧‍👦 Multi-Household Testing

### Child in Multiple Families

**Setup**: child3 (Eddie) belongs to both "Parent Family" and "Third Family"

- [ ] Login as child3@test.com
- [ ] Verify family switcher shows both families
- [ ] Switch to "Parent Family"
- [ ] Verify sees chores from Parent Family only
- [ ] Switch to "Third Family"
- [ ] Verify sees chores from Third Family only
- [ ] Complete chore in each family
- [ ] Verify rewards go to correct family's account

### Parent Managing Multi-Household Child

- [ ] Login as parent1@test.com (Parent Family)
- [ ] Create chore assigned to child3
- [ ] Login as parent3@test.com (Third Family)
- [ ] Create chore assigned to child3
- [ ] Verify each parent only sees their own family's assignments

**Backend Verification for Multi-Household**:
```bash
# Check child3's family memberships
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT fm.id, u.email, f.name as family_name, fm.role
   FROM family_memberships fm
   JOIN users u ON fm.user_id = u.id
   JOIN families f ON fm.family_id = f.id
   WHERE u.email='child3@test.com';"
# Expected: 2 rows (two family memberships)
```

---

## 🌐 Multi-Tenant Isolation

- [ ] Login as user in Family A
- [ ] Create a chore
- [ ] Login as user in Family B
- [ ] Verify cannot see Family A's chore
- [ ] Verify `/chores/{id}` returns 404 for other family's chore

---

## 💳 Reward Integration

- [ ] Approval creates transaction
- [ ] Transaction appears in child's account
- [ ] Balance increased by reward amount
- [ ] Transaction shows "Chore Reward" category (CHORE_REWARD)
- [ ] Transaction description includes chore title

**Backend Verification**:
```bash
# Check transaction was created after approving a chore
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT t.id, t.amount, t.transaction_type, t.category, t.description, a.account_type
   FROM transactions t
   JOIN accounts a ON t.account_id = a.id
   ORDER BY t.created_at DESC LIMIT 5;"
# Expected: transaction_type=CREDIT, category=CHORE_REWARD

# Check account balance updated
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT u.first_name, a.account_type, a.balance 
   FROM accounts a JOIN users u ON a.owner_id = u.id
   WHERE u.email='CHILD_EMAIL';"
```

---

## 📊 All Assignment Statuses Testing

Verify each status is correctly handled in the system:

### PENDING (Assigned, waiting to start)
- [ ] Create assigned chore with future due date
- [ ] Verify assignment status = PENDING
- [ ] Verify child sees in "Upcoming" section

### AVAILABLE (First-dibs, unclaimed)
- [ ] Create FIRST_DIBS chore
- [ ] Verify assignment status = AVAILABLE
- [ ] Verify multiple children can see it as claimable

### CLAIMED (First-dibs, claimed by child)
- [ ] Child claims AVAILABLE assignment
- [ ] Verify status changes to CLAIMED
- [ ] Verify assigned_to_id is set

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

### Concurrent Claim (FIRST_DIBS race condition)
- [ ] Two children try to claim same FIRST_DIBS chore
- [ ] Verify only one succeeds
- [ ] Verify second gets clear error message

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

---

## 👥 Phase 1 Amendment: Bidirectional Family Building

### Test Scenario 1: Adult Registration (Create New Family)

**Goal**: Verify an adult (18+) can register and automatically get a family created.

- [ ] Navigate to `http://localhost:5173/register`
- [ ] **Step 1 - Birth Year**: Select birth year for adult (e.g., 2000), Click "Continue"
- [ ] **Step 2 - Basic Info**:
  - [ ] Verify badge shows "👨‍👩‍👧 Parent Account"
  - [ ] Enter: First name, Last name, Email, Password, Confirm Password
  - [ ] Click "Continue"
- [ ] **Step 3 - Family Connection**:
  - [ ] Verify "Create a new family" is selected by default
  - [ ] Click "Create Account"
- [ ] **Success Screen**:
  - [ ] Verify success message appears
  - [ ] Verify "Your family has been created!" message
  - [ ] Verify "Please check your email to verify" message
- [ ] **Backend Verification** (run in terminal):
  ```bash
  # Check user was created with correct role
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT id, email, first_name, last_name, role, email_verified FROM users WHERE email='YOUR_EMAIL';"
  # Expected: role=PARENT, email_verified=false
  
  # Check family was created
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT f.id, f.name, fm.role FROM families f 
     JOIN family_memberships fm ON f.id = fm.family_id 
     JOIN users u ON fm.user_id = u.id WHERE u.email='YOUR_EMAIL';"
  # Expected: Family named "{LastName} Family", role=OWNER
  ```
  - [ ] Check Docker logs for verification email: `docker-compose logs api | grep -i "verification"`

### Test Scenario 2: Adult Registration (Join Existing Family)

**Goal**: Verify an adult can request to join an existing family.

- [ ] Navigate to register, complete as adult
- [ ] **Step 3 - Family Connection**:
  - [ ] Select "Join an existing family" radio button
  - [ ] Enter parent email of existing family owner
  - [ ] Enter message: "Hi, I'm the other parent!"
  - [ ] Click "Send Request"
- [ ] **Success Screen**:
  - [ ] Verify "Your request to join has been sent" message
- [ ] **Backend Verification** (run in terminal):
  ```bash
  # Check user was created
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT id, email, role, email_verified FROM users WHERE email='YOUR_EMAIL';"
  # Expected: role=PARENT, email_verified=false
  
  # Check join request was created
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT i.id, i.invitation_type, i.status, i.target_email, i.message
     FROM invitations i 
     JOIN users u ON i.requester_id = u.id 
     WHERE u.email='YOUR_EMAIL';"
  # Expected: invitation_type=JOIN_REQUEST, status=PENDING
  
  # Check user NOT in any family yet
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT fm.* FROM family_memberships fm 
     JOIN users u ON fm.user_id = u.id WHERE u.email='YOUR_EMAIL';"
  # Expected: 0 rows
  ```
  - [ ] Check Docker logs: `docker-compose logs api | grep -i "join request"`

### Test Scenario 3: Child Registration (Request to Join Parent)

**Goal**: Verify a child (<18) must provide parent email and creates a join request.

- [ ] Navigate to register
- [ ] **Step 1 - Birth Year**: Select birth year for child (e.g., 2015)
- [ ] **Step 2 - Basic Info**: Verify badge shows "🧒 Child Account"
- [ ] **Step 3 - Family Connection**:
  - [ ] Verify NO option to create family (only parent email input)
  - [ ] Enter parent email
  - [ ] Enter message: "Hi Mom, it's me!"
  - [ ] Click "Send Request"
- [ ] **Success Screen**:
  - [ ] Verify "We've sent a request to your parent" message
- [ ] **Backend Verification** (run in terminal):
  ```bash
  # Check child user was created
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT id, email, role, email_verified FROM users WHERE email='CHILD_EMAIL';"
  # Expected: role=CHILD, email_verified=false
  
  # Check join request links to parent's family
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT i.id, i.invitation_type, i.status, i.target_email, f.name as family_name
     FROM invitations i 
     LEFT JOIN families f ON i.family_id = f.id
     JOIN users u ON i.requester_id = u.id 
     WHERE u.email='CHILD_EMAIL';"
  # Expected: invitation_type=JOIN_REQUEST, status=PENDING, family linked
  
  # Check child NOT in family yet
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT * FROM family_memberships fm 
     JOIN users u ON fm.user_id = u.id WHERE u.email='CHILD_EMAIL';"
  # Expected: 0 rows
  ```

### Test Scenario 4: Email Verification Flow

**Goal**: Verify email verification works and returns tokens.

- [ ] Find verification token from Docker logs:
  ```bash
  docker-compose logs api | grep -i "verification" | tail -5
  ```
- [ ] Navigate to `http://localhost:5173/verify-email/<TOKEN>`
- [ ] **Verification Page**:
  - [ ] Verify "Verifying your email..." spinner
  - [ ] Verify "Email Verified!" success message
  - [ ] Verify automatic redirect to dashboard
- [ ] **Backend Verification**:
  ```bash
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT email, email_verified, email_verification_token FROM users WHERE email='YOUR_EMAIL';"
  # Expected: email_verified=true, email_verification_token=NULL
  ```

### Test Scenario 5: Parent Reviews Join Requests

**Goal**: Verify parent can see and approve/reject join requests.

- [ ] Login as parent with pending join request
- [ ] Navigate to dashboard
- [ ] **Dashboard Notification**:
  - [ ] Verify notification card: "X pending join request(s)"
  - [ ] Click the notification
- [ ] **Join Requests Page** (`/family/join-requests`):
  - [ ] Verify pending request shows requester name, email, message, created date, expiration
- [ ] **Approve Request**:
  - [ ] Click "Approve" button
  - [ ] If child email not verified: Verify error message about email verification
  - [ ] If child email verified: Verify success message
- [ ] **Backend Verification**:
  ```bash
  # Check invitation status updated
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT id, status, reviewed_at FROM invitations 
     WHERE requester_id = (SELECT id FROM users WHERE email='REQUESTER_EMAIL');"
  # Expected: status=ACCEPTED
  
  # Check user added to family
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT fm.role, f.name FROM family_memberships fm
     JOIN families f ON fm.family_id = f.id
     JOIN users u ON fm.user_id = u.id WHERE u.email='REQUESTER_EMAIL';"
  # Expected: 1 row showing family membership
  ```

### Test Scenario 6: Parent Rejects Join Request

- [ ] Login as parent with pending join request
- [ ] Go to Join Requests page
- [ ] Click "Reject" on the request
- [ ] Verify success message
- [ ] Verify request removed from list
- [ ] **Backend Verification**:
  ```bash
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT id, status, reviewed_at FROM invitations 
     WHERE requester_id = (SELECT id FROM users WHERE email='REQUESTER_EMAIL');"
  # Expected: status=REJECTED
  ```
  - [ ] Check rejection email logged: `docker-compose logs api | grep -i "reject"`

### Test Scenario 7: Child with Unregistered Parent Email

**Goal**: Verify registration invitation sent when parent email doesn't exist.

- [ ] Register as child with parent email that doesn't exist (e.g., `newparent@test.com`)
- [ ] **Backend Verification**:
  ```bash
  # Child account created
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT id, email, role FROM users WHERE email='CHILD_EMAIL';"
  
  # Join request created with NULL family_id
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT id, family_id, target_email, status FROM invitations 
     WHERE requester_id = (SELECT id FROM users WHERE email='CHILD_EMAIL');"
  # Expected: family_id=NULL, target_email='newparent@test.com'
  ```
  - [ ] Check invitation email: `docker-compose logs api | grep -i "invite"`

### Test Scenario 8: Resend Verification Email

- [ ] Register a new user but don't verify email
- [ ] Call resend API:
  ```bash
  curl -X POST http://localhost:8000/auth/resend-verification \
    -H "Content-Type: application/json" \
    -d '{"email": "YOUR_EMAIL"}'
  ```
- [ ] Verify new token in logs: `docker-compose logs api | grep -i "verification" | tail -3`
- [ ] Response always says "If email registered..." (security)

### Test Scenario 9: Cancel Join Request

- [ ] Login as user with pending join request
- [ ] Cancel via API:
  ```bash
  # Get your join request ID first
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT id FROM invitations WHERE requester_id = (SELECT id FROM users WHERE email='YOUR_EMAIL');"
  
  # Cancel it (need auth token)
  curl -X DELETE http://localhost:8000/join-requests/<REQUEST_ID> \
    -H "Authorization: Bearer YOUR_TOKEN"
  ```
- [ ] Verify deleted:
  ```bash
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT * FROM invitations WHERE id='REQUEST_ID';"
  # Expected: 0 rows
  ```

### Test Scenario 10: Edge Cases

- [ ] **10a**: Child tries to register without parent email → 400 error
- [ ] **10b**: Register with existing email → 400 error "Email already registered"
- [ ] **10c**: Verify with invalid token → Error page "Invalid or expired token"
- [ ] **10d**: Approve unverified user → 400 error explaining child must verify first
  ```bash
  # Check if user is verified before approving
  docker-compose exec db psql -U picklesapp -d picklesapp -c \
    "SELECT email, email_verified FROM users WHERE email='CHILD_EMAIL';"
  ```
- [ ] **10e**: Review already-reviewed request → 400 error
- [ ] **10f**: Cancel non-pending request → 400 error

---

## 📊 API Testing with cURL

### Chore Endpoints

```bash
# Set token and family
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "parent1@test.com", "password": "TestPass123"}' \
  | jq -r '.access_token')
FAMILY_ID="<your-family-uuid>"

# Create chore (with full options)
curl -X POST http://localhost:8000/chores \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Clean your room",
    "description": "Make bed, pick up toys, vacuum",
    "reward_amount": "5.00",
    "reward_type": "money",
    "assignment_type": "ASSIGNED",
    "assigned_to_user_ids": ["<child-uuid>"],
    "recurrence_rule": {
      "frequency": "WEEKLY",
      "interval": 1,
      "by_weekday": ["MO", "WE", "FR"],
      "timing_mode": "absolute",
      "start_date": "'$(date +%Y-%m-%d)'"
    },
    "deadline_time": "20:00:00",
    "require_approval": true
  }'

# List chores
curl http://localhost:8000/chores \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"

# Get chore details
curl http://localhost:8000/chores/<chore-id> \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"

# Preview occurrences
curl -X POST http://localhost:8000/chores/preview \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "recurrence_rule": {"frequency": "WEEKLY", "by_weekday": ["MO","WE","FR"], "start_date": "'$(date +%Y-%m-%d)'"},
    "limit": 5
  }'

# Update chore
curl -X PATCH http://localhost:8000/chores/<chore-id> \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{"reward_amount": "7.50"}'

# Delete chore (soft delete)
curl -X DELETE http://localhost:8000/chores/<chore-id> \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"
```

### Assignment Endpoints

```bash
# Get child token
CHILD_TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "child1@test.com", "password": "TestPass123"}' \
  | jq -r '.access_token')

# List my assignments (as child)
curl http://localhost:8000/chore-assignments/my \
  -H "Authorization: Bearer $CHILD_TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"

# Complete assignment
curl -X POST http://localhost:8000/chore-assignments/<assignment-id>/complete \
  -H "Authorization: Bearer $CHILD_TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{"photo_urls": []}'

# Get pending approvals (as parent)
curl http://localhost:8000/chore-assignments/pending-approval \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"

# Approve assignment
curl -X POST http://localhost:8000/chore-assignments/<assignment-id>/approve \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"

# Reject assignment
curl -X POST http://localhost:8000/chore-assignments/<assignment-id>/reject \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{"rejection_reason": "Room still messy, please try again"}'

# Batch approve
curl -X POST http://localhost:8000/chore-assignments/batch-approve \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{"assignment_ids": ["<id1>", "<id2>", "<id3>"]}'
```

### Auth & Join Request Endpoints

```bash
BASE=http://localhost:8000/api/v1

# Register new user
curl -X POST $BASE/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "TestPass123",
    "first_name": "Test",
    "last_name": "User",
    "birth_year": 2000
  }'

# Verify email
curl -X POST $BASE/auth/verify-email \
  -H "Content-Type: application/json" \
  -d '{"token": "TOKEN_FROM_EMAIL"}'

# Resend verification email
curl -X POST $BASE/auth/resend-verification \
  -H "Content-Type: application/json" \
  -d '{"email": "unverified@example.com"}'

# Create join request (authenticated)
curl -X POST $BASE/join-requests \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"parent_email": "parent@example.com", "message": "Please add me!"}'

# List my join requests
curl $BASE/me/join-requests \
  -H "Authorization: Bearer <token>"

# List family's pending join requests (parent only)
curl $BASE/families/{family_id}/join-requests \
  -H "Authorization: Bearer <token>"

# Approve join request
curl -X POST $BASE/join-requests/{request_id}/review \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"action": "approve"}'

# Reject join request
curl -X POST $BASE/join-requests/{request_id}/review \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"action": "reject"}'

# Cancel join request
curl -X DELETE $BASE/join-requests/{request_id} \
  -H "Authorization: Bearer <token>"
```

---

## 🧪 Run Automated Tests

### Backend Tests

```bash
cd chore-app-backend

# All tests with coverage
docker-compose exec api pytest tests/ -v --cov=app --cov-report=term-missing

# Phase 2 specific tests
docker-compose exec api pytest tests/test_chores.py tests/test_chore_assignments.py tests/test_recurrence.py tests/test_tiered_rewards.py -v

# Individual test files
docker-compose exec api pytest tests/test_chores.py -v
docker-compose exec api pytest tests/test_chore_assignments.py -v
docker-compose exec api pytest tests/test_recurrence.py -v
docker-compose exec api pytest tests/test_tiered_rewards.py -v

# Reset test database if needed
docker-compose exec db psql -U postgres -c "DROP DATABASE IF EXISTS picklesapp_test;"
docker-compose exec db psql -U postgres -c "CREATE DATABASE picklesapp_test;"
docker-compose exec api alembic upgrade head
```

### Frontend Tests

```bash
cd chore-app-frontend

# Unit tests
npm run test

# Unit tests with coverage
npm run test:coverage

# E2E tests
npm run test:e2e

# E2E tests headed (see browser)
npm run test:e2e -- --headed

# Specific E2E test file
npm run test:e2e -- chores.spec.js

# Run specific E2E test by name
npm run test:e2e -- --grep "Create simple daily chore"
```

---

## ✅ Sign-Off

| Area | Tester | Date | Status |
|------|--------|------|--------|
| Parent Chore Management | | | ⬜ |
| Child Chore Workflow | | | ⬜ |
| FIRST_DIBS Claiming | | | ⬜ |
| Parent Approval Workflow | | | ⬜ |
| Tiered Rewards | | | ⬜ |
| Reward Types (MONEY/POINTS/STARS) | | | ⬜ |
| Penalty System | | | ⬜ |
| Recurrence Patterns | | | ⬜ |
| Timing Modes (ABSOLUTE/RELATIVE) | | | ⬜ |
| Auto-Approve | | | ⬜ |
| Photo Required | | | ⬜ |
| Permissions | | | ⬜ |
| Multi-Tenant Isolation | | | ⬜ |
| Multi-Household Child | | | ⬜ |
| Assignment Statuses (all 8) | | | ⬜ |
| Reward Integration | | | ⬜ |
| Age-Appropriate UI | | | ⬜ |
| Edge Cases | | | ⬜ |
| Bidirectional Family Building | | | ⬜ |
| API Testing | | | ⬜ |
| Automated Tests Pass | | | ⬜ |

---

## Notes

Use this space to document any issues found during QA:

| Issue | Severity | Status | Notes |
|-------|----------|--------|-------|
| | | | |
| | | | |
| | | | |
