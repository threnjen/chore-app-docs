# Phase 2 QA: API & Automation Tests

> cURL commands and automated test execution

**Last Updated**: February 3, 2026

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
