# Phase 1 Testing Guide

> Complete testing documentation for Phase 1: Foundation + Email/Password Auth

---

## Table of Contents

1. [Testing Overview](#testing-overview)
2. [Backend Test Scenarios](#backend-test-scenarios)
3. [Frontend Test Scenarios](#frontend-test-scenarios)
4. [Manual Testing Checklist](#manual-testing-checklist)
5. [API Testing with cURL](#api-testing-with-curl)
6. [Running Tests](#running-tests)

---

## Testing Overview

### Coverage Targets

| Component | Target Coverage | Priority |
|-----------|----------------|----------|
| Authentication (backend) | 100% | Critical |
| Transactions (backend) | 100% | Critical |
| Multi-tenant isolation | 100% | Critical |
| Account operations | 80% | High |
| Family operations | 80% | High |
| Frontend auth flows | 80% | High |
| Frontend components | 60% | Medium |

### Test Types

- **Unit Tests**: Isolated function/method testing
- **Integration Tests**: API endpoint testing with database
- **Component Tests**: React component testing with mocks
- **E2E Tests**: Full user flow testing with Playwright

---

## Backend Test Scenarios

### Authentication (100% Coverage Required)

#### Registration

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| AUTH-01 | Register with valid credentials | `{email, password, first_name, last_name}` | 201, returns user + tokens |
| AUTH-02 | Register with duplicate email | Existing email | 400, "Email already registered" |
| AUTH-03 | Register with invalid email format | `{email: "notanemail"}` | 422, validation error |
| AUTH-04 | Register with short password | `{password: "123"}` | 422, "Password too short" |
| AUTH-05 | Register with missing required fields | `{}` | 422, validation errors |

#### Login

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| AUTH-06 | Login with valid credentials | Correct email + password | 200, returns tokens |
| AUTH-07 | Login with invalid password | Correct email, wrong password | 401, "Invalid credentials" |
| AUTH-08 | Login with non-existent email | Unknown email | 401, "Invalid credentials" |
| AUTH-09 | Login with inactive user | Deactivated account | 401, "Account disabled" |

#### Token Management

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| AUTH-10 | Access protected route with valid token | Valid JWT | 200, returns resource |
| AUTH-11 | Access protected route without token | No Authorization header | 401 |
| AUTH-12 | Access protected route with expired token | Expired JWT | 401 |
| AUTH-13 | Access protected route with malformed token | `Bearer invalid` | 401 |
| AUTH-14 | Refresh with valid refresh token | Valid refresh token | 200, new tokens |
| AUTH-15 | Refresh with expired refresh token | Expired refresh token | 401 |
| AUTH-16 | Refresh with invalid refresh token | Random string | 401 |

#### OAuth Stubs

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| AUTH-17 | Google OAuth endpoint | Any | 501, "Not implemented" |
| AUTH-18 | Apple OAuth endpoint | Any | 501, "Not implemented" |
| AUTH-19 | Facebook OAuth endpoint | Any | 501, "Not implemented" |

---

### Transactions (100% Coverage Required)

#### Manual Transactions

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| TXN-01 | Credit transaction | `{amount: 10.00, type: CREDIT}` | 201, balance increases by 10.00 |
| TXN-02 | Debit with sufficient funds | Balance 50, debit 20 | 201, balance = 30.00 |
| TXN-03 | Debit with insufficient funds (no negative) | Balance 10, debit 20 | 400, "Insufficient funds" |
| TXN-04 | Debit within negative limit | Balance 10, debit 20, limit -50 | 201, balance = -10.00 |
| TXN-05 | Debit exceeds negative limit | Balance 10, debit 100, limit -50 | 400, "Exceeds negative limit" |
| TXN-06 | Transaction with zero amount | `{amount: 0}` | 422, "Amount must be positive" |
| TXN-07 | Transaction with negative amount | `{amount: -10}` | 422, "Amount must be positive" |
| TXN-08 | Decimal precision (cents) | `{amount: 10.99}` | 201, balance reflects exact cents |

#### Transfers

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| TXN-09 | Transfer between accounts | Source has funds | 201, both balances updated |
| TXN-10 | Transfer to same account | Same source/dest | 400, "Cannot transfer to same account" |
| TXN-11 | Transfer with insufficient source funds | Source balance < amount | 400, "Insufficient funds" |
| TXN-12 | Transfer atomicity on failure | Source fails mid-transfer | Both balances unchanged |

#### Transaction Queries

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| TXN-13 | List transactions for account | Valid account ID | 200, array of transactions |
| TXN-14 | Filter by transaction type | `?type=CREDIT` | 200, only CREDIT transactions |
| TXN-15 | Filter by date range | `?start_date=...&end_date=...` | 200, transactions in range |
| TXN-16 | List transactions for wrong family | Other family's account | 403/404 |

---

### Multi-Tenant Isolation (100% Coverage Required)

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| MT-01 | Request without X-Family-Id header | No header | 400, "X-Family-Id header required" |
| MT-02 | Request with invalid UUID | `X-Family-Id: notauuid` | 400, "Invalid family ID format" |
| MT-03 | Request with non-member family | Valid UUID, not member | 403, "Not a member" |
| MT-04 | Query account in different family | Other family's account ID | 404 |
| MT-05 | Parent sees all family accounts | Parent token | 200, all accounts |
| MT-06 | Child sees only own accounts | Child token | 200, only child's accounts |
| MT-07 | Child cannot access sibling's account | Sibling's account ID | 403/404 |

---

### Accounts

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| ACCT-01 | Create account | Valid data | 201, account created |
| ACCT-02 | Create account with invalid type | `{type: "INVALID"}` | 422, validation error |
| ACCT-03 | Create account for other user (parent) | Parent creating for child | 201, success |
| ACCT-04 | Create account for other user (child) | Child creating for sibling | 403, forbidden |
| ACCT-05 | Update account name | `{name: "New Name"}` | 200, name updated |
| ACCT-06 | Update account balance directly | `{balance: 1000}` | 400/ignored (balance via transactions only) |
| ACCT-07 | Delete account with balance | Balance > 0 | 400, "Must have zero balance" |
| ACCT-08 | Delete account with zero balance | Balance = 0 | 204, deleted |

---

### Families

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| FAM-01 | Create family | `{name: "Smith Family"}` | 201, creator is OWNER |
| FAM-02 | Create family with empty name | `{name: ""}` | 422, validation error |
| FAM-03 | Add member to family (owner) | Valid user ID | 201, membership created |
| FAM-04 | Add member to family (non-owner) | MEMBER trying to add | 403, forbidden |
| FAM-05 | Remove member (owner) | Valid membership | 204, membership removed |
| FAM-06 | Remove self (owner, last member) | Owner removes self | 400, "Cannot remove last owner" |
| FAM-07 | Update family name (owner) | `{name: "New Name"}` | 200, updated |
| FAM-08 | Update family name (member) | Non-owner | 403, forbidden |

---

## Frontend Test Scenarios

### Unit Tests (Vitest)

#### AuthContext

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| FC-01 | Initial state | Load context | `isAuthenticated: false`, `user: null` |
| FC-02 | Login success | Call `login()` | Stores tokens, sets user, `isAuthenticated: true` |
| FC-03 | Login failure | Invalid credentials | Error thrown, state unchanged |
| FC-04 | Logout | Call `logout()` | Clears tokens, `user: null`, `isAuthenticated: false` |
| FC-05 | Token refresh | Expired access token | Automatic refresh, new tokens stored |
| FC-06 | Token persistence | Reload page | Tokens loaded from localStorage |

#### FamilyContext

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| FC-07 | Load families | On mount | Fetches user's families |
| FC-08 | Switch family | Call `switchFamily(id)` | Updates `currentFamily`, stores in localStorage |
| FC-09 | Create family | Call `createFamily(name)` | API call, adds to list, sets as current |

### Component Tests

#### LoginForm

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| CP-01 | Empty form submission | Submit with empty fields | Validation errors displayed |
| CP-02 | Invalid email | Enter invalid email, submit | "Invalid email" error |
| CP-03 | Short password | Enter < 8 chars, submit | "Password too short" error |
| CP-04 | Valid submission | Enter valid data, submit | `onSuccess` callback called |
| CP-05 | Loading state | Submit form | Button shows loading spinner |
| CP-06 | API error | Backend returns 401 | Error message displayed |

#### RegisterForm

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| CP-07 | Missing required fields | Submit empty | All required field errors shown |
| CP-08 | Password requirements | Weak password | Specific requirement errors |
| CP-09 | Successful registration | Valid data | `onSuccess` callback called |

### E2E Tests (Playwright)

#### Authentication Flows

| ID | Scenario | Steps | Expected Result |
|----|----------|-------|-----------------|
| E2E-01 | Complete registration | Fill form → Submit | Redirect to dashboard |
| E2E-02 | Complete login | Fill form → Submit | Redirect to dashboard |
| E2E-03 | Protected route redirect | Visit /dashboard without auth | Redirect to /login |
| E2E-04 | Logout flow | Click logout | Redirect to /login, tokens cleared |
| E2E-05 | Session persistence | Login → Refresh page | Still authenticated |

#### Account Flows

| ID | Scenario | Steps | Expected Result |
|----|----------|-------|-----------------|
| E2E-06 | View accounts | Login → Navigate to /accounts | Account list displayed |
| E2E-07 | View empty state | New user → /accounts | "No accounts" message |

#### Family Flows

| ID | Scenario | Steps | Expected Result |
|----|----------|-------|-----------------|
| E2E-08 | Family switcher | Multiple families → Click switcher | Dropdown shows all families |
| E2E-09 | Switch family | Select different family | Dashboard updates with new family data |

---

## Manual Testing Checklist

### Prerequisites

- [x] Docker Desktop running
- [x] Backend started (`docker-compose up`)
- [x] Database migrated (`docker-compose exec api alembic upgrade head`)
- [x] Load seed data in backed (`docker-compose exec api python scripts/seed_dev_data.py`)
- [x] Frontend started (`npm run dev`) http://localhost:5173/
- [ ] Seed data loaded (optional)

### Resetting
- `docker-compose down -v`
- `docker-compose up -d`
- `docker-compose exec api python scripts/seed_dev_data.py`


### Authentication

- [x] Can register new parent account
- [x] Cannot register with duplicate email
- [x] Can login with registered account
- [x] Cannot login with wrong password
- [ ] Token refreshes automatically when expired
- [x] Logout clears session
- [x] OAuth buttons show "Coming Soon" alert

### Family Management

- [x] New user prompted to create family
- [x] Can create family with valid name
- [x] User becomes OWNER of created family
- [x] Family switcher shows all memberships
- [x] Can switch between families (if multiple)

### Child Account Management

- [x] Can navigate to Family Members page from sidebar
- [x] Family Members page shows current members with roles
- [x] Parent can click "Add Child" button
- [x] Can fill in child's email, first name, last name
- [x] Can optionally set child's birthdate
- [x] Child account is created with default password `TestPass123`
- [x] Success message shows with password hint
- [x] New child appears in Family Members list with "Child" badge
- [x] Child can log in with their email and `TestPass123`
- [x] Adding existing user to family shows appropriate message
- [x] Cannot add user who is already a family member (shows error)

### Accounts

- [x] Can navigate to create account page
- [x] Can create new account (SPEND, SAVE, GIVE, INVEST types)
- [x] Can select account owner from family members (including children)
- [x] Account appears in list grouped by type
- [x] Account shows correct balance
- [x] Can view account details
- [x] Icons match account types

### Transactions

- [x] Can navigate to create transaction page
- [x] Can create deposit (CREDIT)
- [x] Can create withdrawal (DEBIT)
- [x] Balance updates after transaction
- [x] Cannot overdraw without negative balance setting
- [x] Can transfer between accounts (using transfer toggle)
- [x] Both balances update on transfer
- [x] Transaction appears in history
- [x] Can filter by transaction type

### Responsive Design

> **Deferred to Later Phase**: Mobile testing requires physical device access. Responsive design testing will be completed in a future phase.

- [ ] Login page renders on mobile
- [ ] Dashboard layout adapts to mobile
- [ ] Bottom navigation shows on mobile
- [ ] Sidebar shows on desktop

---

## API Testing with cURL

### Setup

```bash
# Set base URL
API_URL="http://localhost:8000"

# Register a test user
curl -X POST "$API_URL/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "SecurePass123",
    "first_name": "Test",
    "last_name": "User"
  }'

# Login and capture tokens
RESPONSE=$(curl -s -X POST "$API_URL/auth/login" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "test@example.com",
    "password": "SecurePass123"
  }')

ACCESS_TOKEN=$(echo $RESPONSE | jq -r '.access_token')
REFRESH_TOKEN=$(echo $RESPONSE | jq -r '.refresh_token')

echo "Access Token: $ACCESS_TOKEN"
```

### Create Family

```bash
# Create a family
FAMILY_RESPONSE=$(curl -s -X POST "$API_URL/families" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Family"}')

FAMILY_ID=$(echo $FAMILY_RESPONSE | jq -r '.id')
echo "Family ID: $FAMILY_ID"
```

### Create Account

```bash
# Create a savings account
curl -X POST "$API_URL/accounts" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Savings",
    "account_type": "SAVE"
  }'
```

### Create Transaction

```bash
# Get account ID first
ACCOUNT_ID=$(curl -s "$API_URL/accounts" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" | jq -r '.[0].id')

# Create a deposit
curl -X POST "$API_URL/transactions" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d "{
    \"account_id\": \"$ACCOUNT_ID\",
    \"amount\": 50.00,
    \"transaction_type\": \"CREDIT\",
    \"category\": \"MANUAL_CREDIT\",
    \"description\": \"Initial deposit\"
  }"
```

### Test Multi-Tenant Isolation

```bash
# Try accessing without X-Family-Id (should fail)
curl -X GET "$API_URL/accounts" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
# Expected: 400 "X-Family-Id header is required"

# Try accessing with invalid family (should fail)
curl -X GET "$API_URL/accounts" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "X-Family-Id: 00000000-0000-0000-0000-000000000000"
# Expected: 403 "You are not a member of this family"
```

---

## Running Tests

### Backend Tests

```bash
cd chore-app-backend

# Build and start containers
docker-compose up -d

# Run migrations (first time or after pulling new migrations)
docker-compose exec api alembic upgrade head

# Run all tests
docker-compose exec api pytest -v

# Run with coverage
docker-compose exec api pytest --cov=app --cov-report=html

# Run specific test file
docker-compose exec api pytest tests/test_auth.py -v

# Run specific test
docker-compose exec api pytest tests/test_auth.py::test_register_success -v
```

### Frontend Tests

```bash
cd chore-app-frontend

# install npm if needed
npm install

# Run unit tests
npm run test

# Run with coverage
npm run test:coverage

# Run E2E tests
npm run test:e2e

# Run E2E with UI
npm run test:e2e:ui
```

### Coverage Reports

- Backend: `chore-app-backend/htmlcov/index.html`
- Frontend: `chore-app-frontend/coverage/index.html`

---

## Test Data

### Seed Users

| Email | Password | Role | Notes |
|-------|----------|------|-------|
| parent1@test.com | TestPass123 | PARENT | Owner of Family A |
| parent2@test.com | TestPass123 | PARENT | Owner of Family B, member of A |
| child1@test.com | TestPass123 | CHILD | Member of Family A (age 8) |
| child2@test.com | TestPass123 | CHILD | Member of Family A (age 12) |
| child3@test.com | TestPass123 | CHILD | Member of Family B, also in A (age 15) |

### Test Families

| Name | Scenario |
|------|----------|
| Family A | Primary test family with multiple children |
| Family B | Secondary family for multi-household testing |

### Test Accounts

Each child should have SPEND, SAVE, and GIVE accounts created via seed script.
