# Phase 1 QA: Family Building

> Bidirectional family building flows (registration, join requests, email verification)

**Last Updated**: February 3, 2026

---

## 👥 Bidirectional Family Building

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
