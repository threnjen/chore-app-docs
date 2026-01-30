# API Specifications

> **Source of Truth**: The backend repository (`chore-app-backend`) is the authoritative source for API implementation. This document serves as the contract between frontend and backend services.

### API Conventions

**Base URL**: `https://api.picklesapp.com/api/v1`

**Authentication**: Bearer token in Authorization header
```
Authorization: Bearer <JWT_TOKEN>
```

**Request Headers**:
```
Content-Type: application/json
X-Family-Id: <family_id>  // Required for family-scoped operations
```

**Response Format**:
```json
{
  "success": true,
  "data": { ... },
  "message": "Operation successful",
  "timestamp": "2026-01-29T12:00:00Z"
}
```

**Error Format**:
```json
{
  "success": false,
  "error": {
    "code": "INVALID_CHORE_SCHEDULE",
    "message": "Cannot schedule chore in the past",
    "details": {
      "field": "due_date",
      "value": "2026-01-01"
    }
  },
  "timestamp": "2026-01-29T12:00:00Z"
}
```

**Pagination**:
```
GET /api/v1/transactions?page=2&limit=50
```

Response includes:
```json
{
  "data": [...],
  "pagination": {
    "page": 2,
    "limit": 50,
    "total": 237,
    "pages": 5
  }
}
```

### Authentication Endpoints

#### POST /auth/register
Register new user (parent or child during onboarding)
```json
Request:
{
  "email": "parent@example.com",
  "password": "securepassword123",
  "name": "Jane Doe",
  "role": "PARENT",
  "birthdate": "1985-05-15" // Optional for parents
}

Response:
{
  "success": true,
  "data": {
    "user": {
      "id": "uuid",
      "email": "parent@example.com",
      "name": "Jane Doe",
      "role": "PARENT"
    },
    "access_token": "jwt_token",
    "refresh_token": "refresh_token",
    "token_type": "bearer",
    "expires_in": 3600
  }
}
```

#### POST /auth/login
Login with email/password
```json
Request:
{
  "email": "parent@example.com",
  "password": "securepassword123"
}

Response:
{
  "success": true,
  "data": {
    "user": { ... },
    "access_token": "jwt_token",
    "refresh_token": "refresh_token",
    "families": [
      {
        "id": "family_uuid_1",
        "name": "Mom's House",
        "role": "PARENT"
      },
      {
        "id": "family_uuid_2",
        "name": "Dad's House",
        "role": "PARENT"
      }
    ]
  }
}
```

#### POST /auth/social/google
Login with Google OAuth
```json
Request:
{
  "id_token": "google_id_token",
  "access_token": "google_access_token"
}

Response: Same as /auth/login
```

#### POST /auth/social/apple
Login with Apple Sign In

#### POST /auth/social/facebook
Login with Facebook Login

#### POST /auth/refresh
Refresh access token
```json
Request:
{
  "refresh_token": "refresh_token"
}

Response:
{
  "access_token": "new_jwt_token",
  "refresh_token": "new_refresh_token",
  "expires_in": 3600
}
```

#### POST /auth/link-social
Link social provider to existing account (Phase 6)

### Family Endpoints

#### POST /families
Create new family (during onboarding)
```json
Request:
{
  "name": "The Smith Family",
  "timezone": "America/New_York",
  "currency": "USD",
  "settings": {
    "default_allow_negative_balances": false,
    "default_transfer_limit": null,
    "default_parenting_preset": "balanced"
  }
}

Response:
{
  "success": true,
  "data": {
    "family": {
      "id": "uuid",
      "name": "The Smith Family",
      "timezone": "America/New_York",
      "currency": "USD",
      "subscription_tier": "free",
      "trial_ends_at": "2026-02-28T23:59:59Z"
    }
  }
}
```

#### GET /families/{family_id}
Get family details

#### PATCH /families/{family_id}
Update family settings

#### GET /families/{family_id}/members
List all family members

#### POST /families/{family_id}/members
Add family member (parent or child)
```json
Request:
{
  "user_id": "existing_user_uuid", // If adding existing user to family
  "email": "child@example.com", // If creating new user
  "name": "Emma Smith",
  "role": "CHILD",
  "birthdate": "2015-03-10",
  "parenting_preset": "balanced" // Applies age-appropriate defaults
}

Response:
{
  "success": true,
  "data": {
    "membership": {
      "id": "uuid",
      "user": {
        "id": "uuid",
        "name": "Emma Smith",
        "age": 10,
        "age_bracket": "tween"
      },
      "family_id": "uuid",
      "role": "CHILD",
      "joined_at": "2026-01-29T12:00:00Z"
    }
  }
}
```

#### DELETE /families/{family_id}/members/{user_id}
Remove member from family (soft delete, marks inactive)

#### POST /families/switch
Switch family context (for multi-household users)
```json
Request:
{
  "family_id": "uuid"
}

Response:
{
  "success": true,
  "data": {
    "access_token": "new_jwt_with_updated_family_context",
    "family": {
      "id": "uuid",
      "name": "Dad's House"
    }
  }
}
```

### Account Endpoints

#### GET /accounts
List accounts for current family context
```
Query params:
  - user_id: Filter by specific user
  - include_hidden: true/false
```

#### POST /accounts
Create new account
```json
Request:
{
  "user_id": "child_uuid",
  "name": "Spending",
  "account_type": "SPENDING",
  "balance": 0,
  "allow_negative": false,
  "interest_rate": null
}
```

#### GET /accounts/{account_id}
Get account details with recent transactions

#### PATCH /accounts/{account_id}
Update account settings

#### DELETE /accounts/{account_id}
Soft delete account (mark hidden)

#### GET /accounts/{account_id}/transactions
List transactions for account (paginated)

### Transaction Endpoints

#### POST /transactions
Create manual transaction
```json
Request:
{
  "account_id": "uuid",
  "transaction_type": "CREDIT",
  "amount": 10.50,
  "description": "Allowance"
}

Response:
{
  "success": true,
  "data": {
    "transaction": {
      "id": "uuid",
      "account_id": "uuid",
      "transaction_type": "CREDIT",
      "amount": 10.50,
      "balance_after": 35.50,
      "description": "Allowance",
      "created_at": "2026-01-29T12:00:00Z"
    }
  }
}
```

#### POST /transactions/transfer
Transfer between accounts
```json
Request:
{
  "from_account_id": "uuid",
  "to_account_id": "uuid",
  "amount": 25.00,
  "description": "Moving to savings"
}

Response: Creates two linked transactions (TRANSFER_OUT, TRANSFER_IN)
```

#### POST /transactions/split-credit
Credit with split across multiple accounts
```json
Request:
{
  "user_id": "child_uuid",
  "amount": 100.00,
  "split_id": "uuid", // or inline percentages
  "description": "Birthday money"
}
```

#### GET /transactions/{transaction_id}
Get transaction details

#### GET /transactions/export
Export transactions as CSV
```
Query params:
  - start_date, end_date
  - account_id
  - format: csv/json
```

### Chore Endpoints

#### GET /chores
List chores for family
```
Query params:
  - assigned_to: user_id
  - is_template: true/false
  - is_active: true/false
```

#### POST /chores
Create new chore
```json
Request:
{
  "title": "Clean your room",
  "description": "Make bed, pick up toys, vacuum floor",
  "reward_amount": 5.00,
  "reward_type": "money",
  "assignment_type": "ASSIGNED",
  "assigned_to_user_ids": ["child_uuid"],
  "recurrence_rule": {
    "frequency": "WEEKLY",
    "interval": 1,
    "by_weekday": ["MO", "WE", "FR"], // M-W-F
    "timing_mode": "relative", // or "absolute"
    "start_date": "2026-02-01"
  },
  "deadline_time": "20:00:00",
  "require_approval": true
}

Response:
{
  "success": true,
  "data": {
    "chore": {
      "id": "uuid",
      "title": "Clean your room",
      "reward_amount": 5.00,
      "recurrence_rule": { ... },
      "next_instance_date": "2026-02-03"
    }
  }
}
```

#### GET /chores/{chore_id}
Get chore details

#### PATCH /chores/{chore_id}
Update chore

#### DELETE /chores/{chore_id}
Delete chore (soft delete)

#### POST /chores/{chore_id}/generate-instances
Manually trigger instance generation (for testing)

#### GET /chores/templates
Get library of chore templates

#### POST /chores/from-template
Create chore from template

### Chore Assignment Endpoints

#### GET /chore-assignments
List chore assignments
```
Query params:
  - assigned_to: user_id
  - status: PENDING/COMPLETED/APPROVED/etc
  - due_date: YYYY-MM-DD
  - overdue: true/false
```

#### GET /chore-assignments/{assignment_id}
Get assignment details

#### POST /chore-assignments/{assignment_id}/claim
Claim first-dibs chore
```json
Request: {}  // User inferred from JWT

Response:
{
  "success": true,
  "data": {
    "assignment": {
      "id": "uuid",
      "status": "CLAIMED",
      "claimed_by_user_id": "uuid",
      "claimed_at": "2026-01-29T12:00:00Z"
    }
  }
}
```

#### POST /chore-assignments/{assignment_id}/complete
Mark chore complete
```json
Request:
{
  "photo_urls": ["s3://bucket/photo1.jpg"] // Optional
}

Response:
{
  "success": true,
  "data": {
    "assignment": {
      "id": "uuid",
      "status": "COMPLETED",
      "completed_at": "2026-01-29T18:30:00Z",
      "awaiting_approval": true
    }
  }
}
```

#### POST /chore-assignments/{assignment_id}/approve
Parent approves completed chore
```json
Request: {}

Response: Creates transaction crediting reward to kid's account
```

#### POST /chore-assignments/{assignment_id}/reject
Parent rejects completed chore
```json
Request:
{
  "reason": "Room still messy, please redo"
}

Response:
{
  "success": true,
  "data": {
    "assignment": {
      "status": "REJECTED",
      "rejection_reason": "Room still messy, please redo"
    }
  }
}
```

#### POST /chore-assignments/batch-approve
Approve multiple chores at once
```json
Request:
{
  "assignment_ids": ["uuid1", "uuid2", "uuid3"]
}
```

### Allowance Endpoints

#### GET /allowances
List allowances for family

#### POST /allowances
Create allowance
```json
Request:
{
  "user_id": "child_uuid",
  "name": "Weekly Allowance",
  "amount": 10.00,
  "amount_formula": null, // or "age * 2"
  "frequency": "weekly",
  "day_of_week": 0, // Monday
  "start_date": "2026-02-03",
  "split_id": "uuid" // Split 60/20/20 spending/saving/giving
}
```

#### GET /allowances/{allowance_id}
Get allowance details

#### PATCH /allowances/{allowance_id}
Update allowance

#### DELETE /allowances/{allowance_id}
Delete allowance

#### POST /allowances/{allowance_id}/process
Manually trigger allowance payment (for testing)

### Split Endpoints

#### GET /splits
List splits for user

#### POST /splits
Create split definition
```json
Request:
{
  "user_id": "child_uuid",
  "name": "My Standard Split",
  "percentages": {
    "spending_account_id": 60,
    "saving_account_id": 30,
    "giving_account_id": 10
  }
}
```

#### GET /splits/{split_id}

#### PATCH /splits/{split_id}

#### DELETE /splits/{split_id}

### Savings Goal Endpoints

#### GET /savings-goals
List goals for family

#### POST /savings-goals
Create savings goal
```json
Request:
{
  "account_id": "saving_account_uuid",
  "name": "Nintendo Switch",
  "target_amount": 299.99,
  "target_date": "2026-12-25",
  "image_url": "https://..."
}
```

#### GET /savings-goals/{goal_id}

#### PATCH /savings-goals/{goal_id}

#### DELETE /savings-goals/{goal_id}

#### POST /savings-goals/{goal_id}/complete
Mark goal as completed

### Budget Endpoints

#### GET /budgets

#### POST /budgets

#### GET /budgets/{budget_id}

#### PATCH /budgets/{budget_id}

#### DELETE /budgets/{budget_id}

### Notification Endpoints

#### GET /notifications
List notifications for current user
```
Query params:
  - is_read: true/false
  - limit: 50
```

#### PATCH /notifications/{notification_id}/read
Mark notification as read

#### POST /notifications/mark-all-read
Mark all notifications as read

#### GET /notifications/preferences
Get notification preferences

#### PATCH /notifications/preferences
Update notification preferences
```json
Request:
{
  "chore_reminders": true,
  "chore_completions": true,
  "allowance_payments": false,
  "email_digest": "daily" // daily/weekly/never
}
```