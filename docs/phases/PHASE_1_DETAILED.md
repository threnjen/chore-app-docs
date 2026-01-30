# Phase 1: Foundation Implementation Guide

**Estimated Duration**: 2-3 weeks  
**Status**: In Progress  
**Last Updated**: January 30, 2026

---

## Overview

Phase 1 establishes the foundation for PicklesApp: Docker-based local development, JWT authentication with email/password (OAuth stubbed for Phase 6), multi-household data models, and manual transaction system.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Database | Docker PostgreSQL | Cross-platform (Windows WSL/macOS) |
| Auth Method | Email/password first | Simpler MVP, OAuth deferred to Phase 6 |
| Docker Strategy | Separate per-repo | Backend owns DB, frontend is standalone |
| Windows Support | WSL required | Consistent Unix-like environment |

---

## Prerequisites

### All Developers

- [ ] Docker Desktop installed and running
- [ ] Git configured with SSH keys
- [ ] VS Code with recommended extensions
- [ ] Node.js 20+ (for frontend development outside Docker)
- [ ] Python 3.11+ (for backend development outside Docker)

### Windows Developers

- [ ] WSL 2 installed and configured
- [ ] Ubuntu 22.04 LTS in WSL
- [ ] Docker Desktop WSL integration enabled
- [ ] Repos cloned inside WSL filesystem (`/home/user/repos/`)

---

## Repository Structure

### chore-app-backend

```
chore-app-backend/
├── .env.example              # Environment template
├── .env                      # Local environment (gitignored)
├── .gitignore
├── Dockerfile
├── docker-compose.yml        # PostgreSQL + FastAPI
├── requirements.txt
├── requirements-dev.txt
├── alembic.ini
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
├── app/
│   ├── __init__.py
│   ├── main.py               # FastAPI application
│   ├── config.py             # Pydantic settings
│   ├── database.py           # SQLAlchemy setup
│   ├── dependencies.py       # Dependency injection
│   ├── models/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── family.py
│   │   ├── account.py
│   │   └── transaction.py
│   ├── schemas/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── family.py
│   │   ├── account.py
│   │   ├── transaction.py
│   │   └── auth.py
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── users.py
│   │   ├── families.py
│   │   ├── accounts.py
│   │   └── transactions.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── auth_service.py
│   │   ├── user_service.py
│   │   ├── family_service.py
│   │   ├── account_service.py
│   │   └── transaction_service.py
│   └── middleware/
│       ├── __init__.py
│       └── family_context.py
└── tests/
    ├── __init__.py
    ├── conftest.py
    ├── test_auth.py
    ├── test_accounts.py
    └── test_transactions.py
```

### chore-app-frontend

```
chore-app-frontend/
├── .env.example
├── .env                      # Local environment (gitignored)
├── .gitignore
├── Dockerfile
├── docker-compose.yml        # Vite dev server only
├── package.json
├── vite.config.js
├── tailwind.config.js
├── postcss.config.js
├── index.html
├── public/
│   └── manifest.json
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── index.css
    ├── contexts/
    │   ├── AuthContext.jsx
    │   └── FamilyContext.jsx
    ├── hooks/
    │   ├── useAuth.js
    │   └── useFamily.js
    ├── services/
    │   ├── api.js
    │   └── authService.js
    ├── components/
    │   ├── common/
    │   │   ├── Button.jsx
    │   │   ├── Input.jsx
    │   │   ├── Card.jsx
    │   │   └── Loading.jsx
    │   ├── layout/
    │   │   ├── Header.jsx
    │   │   ├── Sidebar.jsx
    │   │   └── Layout.jsx
    │   └── auth/
    │       ├── LoginForm.jsx
    │       ├── RegisterForm.jsx
    │       └── ProtectedRoute.jsx
    └── pages/
        ├── LoginPage.jsx
        ├── RegisterPage.jsx
        ├── DashboardPage.jsx
        ├── AccountsPage.jsx
        └── TransactionsPage.jsx
```

---

## Implementation Checklist

### 1. Backend Docker Infrastructure

**Files to create:**

- [ ] `Dockerfile`
- [ ] `docker-compose.yml`
- [ ] `requirements.txt`
- [ ] `requirements-dev.txt`
- [ ] Update `.env.example`

**docker-compose.yml services:**

| Service | Image | Port | Purpose |
|---------|-------|------|---------|
| `db` | postgres:15 | 5432 | PostgreSQL database |
| `api` | (build) | 8000 | FastAPI application |

**Acceptance Criteria:**

- [ ] `docker-compose up` starts PostgreSQL and FastAPI
- [ ] FastAPI auto-reloads on code changes (volume mount)
- [ ] PostgreSQL data persists across restarts (named volume)
- [ ] Health check endpoint returns 200

---

### 2. Frontend Docker Infrastructure

**Files to create:**

- [ ] `Dockerfile`
- [ ] `docker-compose.yml`
- [ ] Update `.env.example`

**Acceptance Criteria:**

- [ ] `docker-compose up` starts Vite dev server
- [ ] Hot module replacement works
- [ ] API proxy to `localhost:8000` works
- [ ] Accessible at `http://localhost:5173`

---

### 3. Backend FastAPI Structure

**Files to create:**

- [ ] `app/__init__.py`
- [ ] `app/main.py` - FastAPI app with CORS, routers
- [ ] `app/config.py` - Pydantic Settings class
- [ ] `app/database.py` - SQLAlchemy engine, session
- [ ] `app/dependencies.py` - get_db, get_current_user

**Acceptance Criteria:**

- [ ] `/health` endpoint returns `{"status": "healthy"}`
- [ ] `/docs` shows OpenAPI documentation
- [ ] CORS allows frontend origin
- [ ] Database connection established

---

### 4. SQLAlchemy Models

**Files to create:**

- [ ] `app/models/__init__.py` - Export all models
- [ ] `app/models/user.py` - User model
- [ ] `app/models/family.py` - Family, FamilyMembership models
- [ ] `app/models/account.py` - Account model
- [ ] `app/models/transaction.py` - Transaction model

**Model Specifications:**

#### User
| Column | Type | Constraints |
|--------|------|-------------|
| id | UUID | PK, default uuid4 |
| email | String(255) | Unique, not null |
| password_hash | String(255) | Not null |
| first_name | String(100) | Not null |
| last_name | String(100) | Not null |
| birthdate | Date | Nullable |
| role | Enum(PARENT, CHILD) | Not null |
| created_at | DateTime | default now |
| updated_at | DateTime | onupdate now |

#### Family
| Column | Type | Constraints |
|--------|------|-------------|
| id | UUID | PK |
| name | String(100) | Not null |
| timezone | String(50) | default 'America/New_York' |
| created_at | DateTime | default now |

#### FamilyMembership
| Column | Type | Constraints |
|--------|------|-------------|
| id | UUID | PK |
| user_id | UUID | FK → users.id |
| family_id | UUID | FK → families.id |
| role | Enum(OWNER, ADMIN, MEMBER) | Not null |
| age_bracket_override | String(20) | Nullable |
| is_active | Boolean | default true |
| joined_at | DateTime | default now |

#### Account
| Column | Type | Constraints |
|--------|------|-------------|
| id | UUID | PK |
| family_id | UUID | FK → families.id |
| owner_id | UUID | FK → users.id |
| name | String(100) | Not null |
| account_type | Enum(SPEND, SAVE, GIVE, INVEST) | Not null |
| balance | Decimal(10,2) | default 0.00 |
| allow_negative | Boolean | default false |
| negative_limit | Decimal(10,2) | Nullable |
| created_at | DateTime | default now |

#### Transaction
| Column | Type | Constraints |
|--------|------|-------------|
| id | UUID | PK |
| account_id | UUID | FK → accounts.id |
| amount | Decimal(10,2) | Not null |
| transaction_type | Enum(CREDIT, DEBIT) | Not null |
| category | String(50) | Nullable |
| description | String(255) | Nullable |
| created_by_id | UUID | FK → users.id |
| created_at | DateTime | default now |

**Acceptance Criteria:**

- [ ] All models define `__tablename__`
- [ ] Relationships defined with `relationship()`
- [ ] Indexes on foreign keys
- [ ] Enums use SQLAlchemy Enum type

---

### 5. Alembic Migrations

**Files to create:**

- [ ] `alembic.ini`
- [ ] `alembic/env.py`
- [ ] `alembic/script.py.mako`
- [ ] `alembic/versions/` (directory)

**Acceptance Criteria:**

- [ ] `alembic revision --autogenerate` creates migration
- [ ] `alembic upgrade head` applies all migrations
- [ ] `alembic downgrade -1` reverts last migration
- [ ] Migrations run on container startup

---

### 6. Pydantic Schemas

**Files to create:**

- [ ] `app/schemas/__init__.py`
- [ ] `app/schemas/user.py`
- [ ] `app/schemas/family.py`
- [ ] `app/schemas/account.py`
- [ ] `app/schemas/transaction.py`
- [ ] `app/schemas/auth.py`

**Schema Patterns:**

```python
# Base → Create → Read → Update pattern
class UserBase(BaseModel):
    email: EmailStr
    first_name: str
    last_name: str

class UserCreate(UserBase):
    password: str

class UserRead(UserBase):
    id: UUID
    role: UserRole
    created_at: datetime
    
    model_config = ConfigDict(from_attributes=True)

class UserUpdate(BaseModel):
    first_name: str | None = None
    last_name: str | None = None
```

**Acceptance Criteria:**

- [ ] All schemas use Pydantic v2 syntax
- [ ] `model_config = ConfigDict(from_attributes=True)` for Read schemas
- [ ] Proper validation with Field() constraints
- [ ] EmailStr for email fields

---

### 7. JWT Authentication

**Files to create:**

- [ ] `app/routers/auth.py`
- [ ] `app/services/auth_service.py`

**Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| POST | `/auth/register` | Create new user account |
| POST | `/auth/login` | Login with email/password |
| POST | `/auth/refresh` | Refresh access token |
| POST | `/auth/logout` | Invalidate refresh token |
| GET | `/auth/me` | Get current user |
| POST | `/auth/google` | Stub - returns 501 |
| POST | `/auth/apple` | Stub - returns 501 |
| POST | `/auth/facebook` | Stub - returns 501 |

**Token Configuration:**

| Setting | Value |
|---------|-------|
| Access token expiry | 1 hour |
| Refresh token expiry | 30 days |
| Algorithm | HS256 |
| Password hashing | bcrypt |

**Acceptance Criteria:**

- [ ] Registration creates user with hashed password
- [ ] Login returns access + refresh tokens
- [ ] Access token verified on protected routes
- [ ] Refresh token rotation implemented
- [ ] OAuth stubs return 501 with message

---

### 8. Account & Transaction Endpoints

**Files to create:**

- [ ] `app/routers/accounts.py`
- [ ] `app/routers/transactions.py`
- [ ] `app/services/account_service.py`
- [ ] `app/services/transaction_service.py`

**Account Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/accounts` | List user's accounts (family-scoped) |
| POST | `/accounts` | Create account |
| GET | `/accounts/{id}` | Get account details |
| PATCH | `/accounts/{id}` | Update account |
| DELETE | `/accounts/{id}` | Soft delete account |

**Transaction Endpoints:**

| Method | Path | Description |
|--------|------|-------------|
| GET | `/transactions` | List transactions (filterable) |
| POST | `/transactions` | Create manual transaction |
| GET | `/transactions/{id}` | Get transaction details |

**Acceptance Criteria:**

- [ ] Transactions update account balance atomically
- [ ] Negative balance validation respects account settings
- [ ] Family context middleware filters results
- [ ] Parents can view all family accounts
- [ ] Children only see their own accounts

---

### 9. Multi-Tenant Middleware

**Files to create:**

- [ ] `app/middleware/__init__.py`
- [ ] `app/middleware/family_context.py`

**Behavior:**

1. Extract `X-Family-Id` header from request
2. Verify user is member of family
3. Inject `family_id` into request state
4. All queries filter by `family_id`

**Acceptance Criteria:**

- [ ] Missing header returns 400
- [ ] Invalid family returns 403
- [ ] User not in family returns 403
- [ ] Valid header sets `request.state.family_id`

---

### 10. Frontend React Structure

**Files to create:**

- [ ] `src/main.jsx`
- [ ] `src/App.jsx`
- [ ] `src/index.css` (Tailwind imports)
- [ ] `tailwind.config.js`
- [ ] `postcss.config.js`

**Acceptance Criteria:**

- [ ] React 18 with StrictMode
- [ ] React Router v6 configured
- [ ] Tailwind CSS working
- [ ] Path aliases configured (@/)

---

### 11. Auth Context & Service

**Files to create:**

- [ ] `src/contexts/AuthContext.jsx`
- [ ] `src/hooks/useAuth.js`
- [ ] `src/services/api.js`
- [ ] `src/services/authService.js`

**AuthContext State:**

```javascript
{
  user: User | null,
  isAuthenticated: boolean,
  isLoading: boolean,
  login: (email, password) => Promise,
  register: (data) => Promise,
  logout: () => void,
  refreshToken: () => Promise,
}
```

**Acceptance Criteria:**

- [ ] Tokens stored in httpOnly cookies (preferred) or localStorage
- [ ] Automatic token refresh before expiry
- [ ] Axios interceptor adds Authorization header
- [ ] 401 responses trigger logout

---

### 12. Auth Pages & Components

**Files to create:**

- [ ] `src/pages/LoginPage.jsx`
- [ ] `src/pages/RegisterPage.jsx`
- [ ] `src/components/auth/LoginForm.jsx`
- [ ] `src/components/auth/RegisterForm.jsx`
- [ ] `src/components/auth/ProtectedRoute.jsx`

**Form Validation (Zod):**

```javascript
const loginSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8, 'Password must be 8+ characters'),
});

const registerSchema = z.object({
  email: z.string().email('Invalid email'),
  password: z.string().min(8).regex(/[A-Z]/, 'Need uppercase'),
  firstName: z.string().min(1, 'Required'),
  lastName: z.string().min(1, 'Required'),
});
```

**Acceptance Criteria:**

- [ ] Client-side validation with Zod
- [ ] Error messages displayed inline
- [ ] Loading states during submission
- [ ] Redirect to dashboard on success
- [ ] OAuth buttons show "Coming soon" toast

---

### 13. Family Context UI

**Files to create:**

- [ ] `src/contexts/FamilyContext.jsx`
- [ ] `src/hooks/useFamily.js`
- [ ] `src/components/layout/FamilySwitcher.jsx`

**FamilyContext State:**

```javascript
{
  families: Family[],
  currentFamily: Family | null,
  switchFamily: (familyId) => void,
  isLoading: boolean,
}
```

**Acceptance Criteria:**

- [ ] Dropdown shows all user's families
- [ ] Switching updates X-Family-Id header
- [ ] Current family name in header
- [ ] Persists selection in localStorage

---

### 14. Test Setup

**Backend (pytest):**

- [ ] `tests/conftest.py` - Fixtures (test client, test db)
- [ ] `tests/test_auth.py` - Auth endpoint tests
- [ ] `tests/test_accounts.py` - Account CRUD tests
- [ ] `tests/test_transactions.py` - Transaction tests

**Frontend (Vitest):**

- [ ] `vitest.config.js`
- [ ] `src/__tests__/AuthContext.test.jsx`
- [ ] `src/__tests__/LoginForm.test.jsx`

**Coverage Targets:**

| Area | Target |
|------|--------|
| Auth flows | 80% |
| Transaction logic | 100% |
| Frontend components | 70% |

---

## Startup Commands

### Backend

```bash
# First time setup
cd chore-app-backend
cp .env.example .env
docker-compose up --build

# Run migrations
docker-compose exec api alembic upgrade head

# Run tests
docker-compose exec api pytest

# View logs
docker-compose logs -f api
```

### Frontend

```bash
# First time setup
cd chore-app-frontend
cp .env.example .env
docker-compose up --build

# Without Docker (faster for development)
npm install
npm run dev

# Run tests
npm test
```

---

## Definition of Done

Phase 1 is complete when:

- [ ] Both repos have working Docker setup
- [ ] User can register with email/password
- [ ] User can login and receive JWT tokens
- [ ] User can create a family
- [ ] User can create accounts within family
- [ ] User can create manual transactions
- [ ] Account balances update correctly
- [ ] Multi-household switching works
- [ ] All tests pass with target coverage
- [ ] OpenAPI docs accessible at `/docs`

---

## Next Phase Preview

**Phase 2: Chores MVP** will add:
- Chore and ChoreAssignment models
- Advanced recurrence scheduling engine
- Background job processing with APScheduler
- Kid dashboard with today's chores
- Parent approval workflow
