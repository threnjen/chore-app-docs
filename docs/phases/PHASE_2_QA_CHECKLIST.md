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
 
### Resetting
- `docker-compose down -v`
- `docker-compose up -d`
- `docker-compose exec api python scripts/seed_dev_data.py`

---

## 🏠 Parent Chore Management

### Create Chore

- [x] Navigate to `/chores` from sidebar
- [x] Click "New Chore" button
- [x] **Form Fields**:
  - [x] Enter title "Test Daily Chore"
  - [x] Enter description "Test description"
  - [x] Set reward amount $5.00
  - [x] Select child to assign to
  - [x] Select frequency "Daily"
  - [x] Verify weekday picker hidden for Daily
  - [x] Select frequency "Weekly"
  - [x] Verify weekday picker appears
  - [x] Select M-W-F
  - [x] Can set interval (every 2 weeks)
  - [x] Can toggle between Absolute and Relative timing mode
  - [x] Verify preview shows next 5 occurrences correctly
  - [x] Set deadline time (optional)
  - [x] Set estimated duration (optional)
  - [x] Toggle "Require Photo"
  - [x] Toggle "Require Approval"
  - [x] Set auto-approve after hours (optional)
  - [x] Toggle "Show Advanced" mode
  - [x] Set grace period (1 day)
  - [x] Set late reward percentage (50%)
  - [x] Set early bonus ($1.00)
  - [x] Set penalty behavior (On Expiration)
  - [x] Set penalty amount ($1.00)
- [x] Submit form
- [x] Verify redirect to `/chores`
- [x] Verify chore appears in list
- [x] Verify initial assignments generated (check via API or database)

### List Chores

- [x] Verify all chores display correctly
- [x] Verify reward badges show correctly
- [x] Verify penalty badges (if configured)
- [x] Verify recurrence pattern descriptions
- [x] Verify assignment count displays
- [x] Can filter by active/inactive
- [x] Can click to view chore details

### View/Edit Chore

- [x] Click on a chore to view details
- [x] Verify `/chores/{id}` page loads
- [x] Click "Edit" button
- [x] Modify title
- [x] Save changes
- [x] Verify update reflected

### Delete Chore

- [ ] Click "Delete" on a chore
- [ ] Confirm deletion (with confirmation dialog)
- [ ] Verify chore removed (soft-delete)
- [ ] Verify with `include_inactive=true` query

---

## 🧒 Child Chore Workflow

### View My Chores

- [ ] Login as child user
- [ ] Navigate to `/chores/my`
- [ ] Verify "Today" section shows due chores
- [ ] Verify "Overdue" section if any (with days count badge)
- [ ] Verify "Upcoming" section shows future chores
- [ ] Verify "Waiting" section shows pending approval
- [ ] Verify "Done" section shows approved

### Complete a Chore

- [ ] Find a pending chore
- [ ] Click "Done"/"Complete" button
- [ ] Completion confirmation shown
- [ ] Verify status changes to "Waiting for Approval"
- [ ] Verify chore moved to appropriate section

### Complete Overdue Chore

- [ ] Find or create an overdue chore
- [ ] Verify overdue badge displays (shows days count)
- [ ] Complete the chore
- [ ] Verify `reward_tier` is REDUCED (if grace configured)

### Complete Early

- [ ] Find chore due in future (2+ days)
- [ ] Complete it now
- [ ] Verify `reward_tier` is BONUS (if early_bonus configured)

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

---

## 🔄 Recurrence Testing

### Daily Pattern

- [ ] Create daily chore
- [ ] Verify assignments created for next 7 days
- [ ] Preview shows consecutive days

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

---

## ⚡ Edge Cases

- [ ] Child CAN view `/chores` page but cannot see "New Chore" button
- [ ] Child cannot access `/chores/create` (shows Access Denied)
- [ ] Child cannot approve/reject chores
- [ ] Cannot complete future chores (disabled button)
- [ ] Cannot complete already approved chores
- [ ] Rejected chores can be re-completed by child
- [ ] Deactivating chore cancels future assignments
- [ ] Pending count badge appears in nav for parent

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
- [ ] **Backend Verification**:
  - [ ] User created with `role=PARENT`, `email_verified=false`
  - [ ] Family created with name "{LastName} Family"
  - [ ] User added as family OWNER
  - [ ] Verification email logged to console

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
- [ ] **Backend Verification**:
  - [ ] User created with `role=PARENT`, `email_verified=false`
  - [ ] Join request created with `status=PENDING`, `invitation_type=JOIN_REQUEST`
  - [ ] Notification email to target parent logged to console
  - [ ] User NOT added to family yet

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
- [ ] **Backend Verification**:
  - [ ] User created with `role=CHILD`, `email_verified=false`
  - [ ] Join request created linking to parent's family
  - [ ] Notification email to parent logged
  - [ ] Child NOT added to family yet

### Test Scenario 4: Email Verification Flow

**Goal**: Verify email verification works and returns tokens.

- [ ] Find verification token from console logs
- [ ] Navigate to `http://localhost:5173/verify-email/<TOKEN>`
- [ ] **Verification Page**:
  - [ ] Verify "Verifying your email..." spinner
  - [ ] Verify "Email Verified!" success message
  - [ ] Verify automatic redirect to dashboard
- [ ] **Backend Verification**:
  - [ ] User's `email_verified` set to `true`
  - [ ] `email_verification_token` set to `null`
  - [ ] Access and refresh tokens returned

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
- [ ] **Verify Member Added**:
  - [ ] Navigate to Family Members page
  - [ ] Verify new member appears in member list

### Test Scenario 6: Parent Rejects Join Request

- [ ] Login as parent with pending join request
- [ ] Go to Join Requests page
- [ ] Click "Reject" on the request
- [ ] Verify success message
- [ ] Verify request removed from list
- [ ] **Backend Verification**:
  - [ ] Join request status changed to `REJECTED`
  - [ ] Rejection email sent to requester (logged)

### Test Scenario 7: Child with Unregistered Parent Email

**Goal**: Verify registration invitation sent when parent email doesn't exist.

- [ ] Register as child with parent email that doesn't exist
- [ ] **Backend Verification**:
  - [ ] Child account created
  - [ ] Registration invitation email sent to parent email (logged)
  - [ ] Join request created with `family_id=null` (no family yet)
  - [ ] Message indicates parent will be invited

### Test Scenario 8: Resend Verification Email

- [ ] Register a new user but don't verify email
- [ ] Hit resend verification API or UI button
- [ ] Verify new verification token generated
- [ ] Verify new verification email sent (logged)
- [ ] Response always says "If email registered..." (security)

### Test Scenario 9: Cancel Join Request

- [ ] Login as user with pending join request
- [ ] Cancel the join request (via API)
- [ ] Verify join request deleted
- [ ] Verify request no longer appears in parent's list

### Test Scenario 10: Edge Cases

- [ ] **10a**: Child tries to register without parent email → 400 error
- [ ] **10b**: Register with existing email → 400 error "Email already registered"
- [ ] **10c**: Verify with invalid token → Error page "Invalid or expired token"
- [ ] **10d**: Approve unverified user → 400 error explaining child must verify first
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
| Parent Approval Workflow | | | ⬜ |
| Tiered Rewards | | | ⬜ |
| Penalty System | | | ⬜ |
| Recurrence Patterns | | | ⬜ |
| Permissions | | | ⬜ |
| Multi-Tenant Isolation | | | ⬜ |
| Reward Integration | | | ⬜ |
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
