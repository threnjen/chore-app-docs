# Backend Testing Strategy

## Overview

The backend testing strategy focuses on ensuring data integrity, multi-tenant isolation, financial transaction accuracy, and API contract compliance.

## Test Coverage Targets

- **Critical Paths** (financial transactions, auth, multi-tenant isolation): **100% coverage**
- **General Code**: **80% coverage minimum**
- **Integration Tests**: All major workflows
- **Load Tests**: Phase 9 (production readiness)

## Testing Framework

### Core Tools
- **pytest**: Test runner and framework
- **pytest-asyncio**: Async test support for FastAPI
- **pytest-cov**: Coverage reporting
- **factory-boy**: Test data generation
- **Faker**: Realistic fake data
- **httpx**: API client for integration tests
- **Locust**: Load and performance testing (Phase 9)

### Installation

```bash
# Install test dependencies
pip install pytest pytest-asyncio pytest-cov factory-boy faker httpx locust
```

## Test Structure

```
tests/
├── unit/                    # Unit tests for individual functions
│   ├── test_auth.py        # Authentication utilities
│   ├── test_transactions.py # Transaction logic
│   ├── test_chore_engine.py # Chore scheduling
│   └── test_models.py      # Model validation
├── integration/            # API integration tests
│   ├── test_auth_flow.py   # Complete auth workflows
│   ├── test_chore_workflow.py
│   ├── test_allowance_workflow.py
│   └── test_multi_household.py
├── factories.py            # Test data factories
├── conftest.py            # Pytest configuration and fixtures
└── load/                  # Load testing scripts
    └── locustfile.py
```

## Unit Tests

### Authentication Tests

```python
# tests/unit/test_auth.py
import pytest
from app.core.security import hash_password, verify_password, create_access_token

def test_password_hashing():
    """Test password hashing and verification"""
    password = "mysecurepassword"
    hashed = hash_password(password)
    
    assert verify_password(password, hashed) is True
    assert verify_password("wrongpassword", hashed) is False

def test_password_hash_uniqueness():
    """Test that same password produces different hashes"""
    password = "testpassword"
    hash1 = hash_password(password)
    hash2 = hash_password(password)
    
    assert hash1 != hash2  # bcrypt uses salts

def test_create_access_token():
    """Test JWT token creation"""
    data = {"sub": "user123", "family_id": "family456"}
    token = create_access_token(data)
    
    assert isinstance(token, str)
    assert len(token) > 0
```

### Transaction Tests

```python
# tests/unit/test_transactions.py
import pytest
from app.services.money_service import process_credit, process_debit, process_transfer

def test_credit_increases_balance():
    """Test that credit transactions increase account balance"""
    account = create_test_account(balance=100.00)
    
    transaction = process_credit(account, amount=25.00, description="Test credit")
    
    assert account.balance == 125.00
    assert transaction.transaction_type == "CREDIT"
    assert transaction.amount == 25.00
    assert transaction.balance_after == 125.00

def test_debit_decreases_balance():
    """Test that debit transactions decrease account balance"""
    account = create_test_account(balance=100.00)
    
    transaction = process_debit(account, amount=25.00, description="Test debit")
    
    assert account.balance == 75.00
    assert transaction.transaction_type == "DEBIT"
    assert transaction.amount == 25.00
    assert transaction.balance_after == 75.00

def test_debit_fails_when_insufficient_funds_and_negatives_not_allowed():
    """Test that debits fail when balance would go negative"""
    account = create_test_account(balance=10.00, allow_negative=False)
    
    with pytest.raises(InsufficientFundsError):
        process_debit(account, amount=25.00)

def test_transfer_moves_money_atomically():
    """Test that transfers are atomic (all or nothing)"""
    account1 = create_test_account(balance=100.00)
    account2 = create_test_account(balance=50.00)
    
    transactions = process_transfer(
        from_account=account1,
        to_account=account2,
        amount=30.00
    )
    
    assert account1.balance == 70.00
    assert account2.balance == 80.00
    assert len(transactions) == 2  # TRANSFER_OUT and TRANSFER_IN
    assert transactions[0].related_transaction_id == transactions[1].id
```

### Chore Scheduling Tests

```python
# tests/unit/test_chore_engine.py
import pytest
from datetime import datetime, timedelta
from app.services.chore_engine import generate_next_instance, parse_rrule

def test_daily_chore_scheduling():
    """Test daily chore recurrence"""
    chore = create_test_chore(
        recurrence_pattern={"frequency": "DAILY", "interval": 1},
        timing_mode="ABSOLUTE"
    )
    start_date = datetime(2026, 1, 1, 9, 0)
    
    next_date = generate_next_instance(chore, from_date=start_date)
    
    assert next_date == datetime(2026, 1, 2, 9, 0)

def test_mwf_pattern():
    """Test Monday-Wednesday-Friday chore pattern"""
    chore = create_test_chore(
        recurrence_pattern={
            "frequency": "WEEKLY",
            "interval": 1,
            "by_weekday": [0, 2, 4]  # Mon, Wed, Fri
        }
    )
    
    # Start on Monday Jan 1, 2026
    instances = generate_instances(chore, start=datetime(2026, 1, 6), count=5)
    
    # Should be Mon 1/6, Wed 1/8, Fri 1/10, Mon 1/13, Wed 1/15
    assert len(instances) == 5
    assert instances[0].weekday() == 0  # Monday
    assert instances[1].weekday() == 2  # Wednesday
    assert instances[2].weekday() == 4  # Friday

def test_relative_timing_mode():
    """Test relative timing (next instance from completion)"""
    chore = create_test_chore(
        recurrence_pattern={"frequency": "DAILY", "interval": 3},
        timing_mode="RELATIVE"
    )
    
    # Completed on Jan 5, next should be Jan 8
    completed_date = datetime(2026, 1, 5, 14, 30)
    next_date = generate_next_instance(chore, from_date=completed_date)
    
    assert next_date.date() == datetime(2026, 1, 8).date()
```

## Integration Tests

### Complete Chore Workflow

```python
# tests/integration/test_chore_workflow.py
import pytest
from fastapi.testclient import TestClient

def test_complete_chore_workflow(client: TestClient, auth_headers_parent, auth_headers_child):
    """Test full chore creation, completion, and approval flow"""
    
    # 1. Parent creates chore
    response = client.post(
        "/api/v1/chores",
        headers=auth_headers_parent,
        json={
            "title": "Clean your room",
            "description": "Vacuum and organize",
            "reward_amount": 10.00,
            "assignment_type": "ASSIGNED",
            "assigned_to": [child_id],
            "require_approval": True
        }
    )
    assert response.status_code == 201
    chore_id = response.json()["data"]["id"]
    
    # 2. Get chore assignment for child
    response = client.get(
        "/api/v1/chore-assignments",
        headers=auth_headers_child
    )
    assert response.status_code == 200
    assignments = response.json()["data"]["items"]
    assignment = next(a for a in assignments if a["chore_id"] == chore_id)
    assert assignment["status"] == "PENDING"
    
    # 3. Child completes chore
    response = client.post(
        f"/api/v1/chore-assignments/{assignment['id']}/complete",
        headers=auth_headers_child
    )
    assert response.status_code == 200
    assert response.json()["data"]["status"] == "COMPLETED"
    
    # 4. Parent approves
    response = client.post(
        f"/api/v1/chore-assignments/{assignment['id']}/approve",
        headers=auth_headers_parent,
        json={"feedback": "Great job!"}
    )
    assert response.status_code == 200
    assert response.json()["data"]["status"] == "APPROVED"
    
    # 5. Verify transaction created
    response = client.get(
        "/api/v1/transactions",
        headers=auth_headers_child
    )
    transactions = response.json()["data"]["items"]
    chore_transaction = next(t for t in transactions if t["description"].startswith("Chore Reward"))
    assert chore_transaction["amount"] == 10.00
    assert chore_transaction["transaction_type"] == "CHORE_REWARD"
    
    # 6. Verify balance updated
    response = client.get(
        f"/api/v1/accounts/{child_account_id}",
        headers=auth_headers_child
    )
    account = response.json()["data"]
    assert account["balance"] == 10.00
```

### Multi-Tenant Isolation Tests

```python
# tests/integration/test_multi_household.py
def test_family_data_isolation(client: TestClient):
    """Test that users cannot access other families' data"""
    
    # User in Family A tries to access Family B's chores
    response = client.get(
        "/api/v1/chores",
        headers={
            "Authorization": f"Bearer {family_a_token}",
            "X-Family-Context": family_b_id  # Wrong family!
        }
    )
    
    # Should get 403 Forbidden
    assert response.status_code == 403
    assert "not authorized" in response.json()["error"]["message"].lower()

def test_child_belongs_to_multiple_families(client: TestClient):
    """Test child can switch between mom's and dad's households"""
    
    # Child has two families
    child_user = create_child_with_two_families()
    
    # Login
    response = client.post("/api/v1/auth/login", json={
        "email": child_user.email,
        "password": "password123"
    })
    token = response.json()["data"]["access_token"]
    
    # Get user's families
    response = client.get(
        "/api/v1/families",
        headers={"Authorization": f"Bearer {token}"}
    )
    families = response.json()["data"]["items"]
    assert len(families) == 2
    
    # Switch to Mom's house
    mom_family_id = families[0]["id"]
    response = client.post(
        f"/api/v1/families/switch",
        headers={"Authorization": f"Bearer {token}"},
        json={"family_id": mom_family_id}
    )
    mom_token = response.json()["data"]["access_token"]
    
    # Get chores in Mom's house
    response = client.get(
        "/api/v1/chores",
        headers={"Authorization": f"Bearer {mom_token}"}
    )
    mom_chores = response.json()["data"]["items"]
    
    # Switch to Dad's house
    dad_family_id = families[1]["id"]
    response = client.post(
        f"/api/v1/families/switch",
        headers={"Authorization": f"Bearer {token}"},
        json={"family_id": dad_family_id}
    )
    dad_token = response.json()["data"]["access_token"]
    
    # Get chores in Dad's house
    response = client.get(
        "/api/v1/chores",
        headers={"Authorization": f"Bearer {dad_token}"}
    )
    dad_chores = response.json()["data"]["items"]
    
    # Chores should be completely different
    assert set(c["id"] for c in mom_chores) != set(c["id"] for c in dad_chores)
```

### Automated Allowance Processing

```python
# tests/integration/test_allowance_workflow.py
def test_weekly_allowance_processing(client: TestClient, db_session):
    """Test automated weekly allowance processing"""
    
    # Create allowance configuration
    allowance = create_test_allowance(
        user_id=child_id,
        amount=10.00,
        frequency="WEEKLY",
        day_of_week=1,  # Monday
        split_config_id=split_id  # 60/20/20 spending/saving/giving
    )
    
    # Manually trigger allowance processor (normally runs on schedule)
    from app.background_jobs.allowance_processor import process_allowances
    process_allowances()
    
    # Verify transactions created
    transactions = db_session.query(Transaction).filter_by(
        user_id=child_id,
        transaction_type="ALLOWANCE"
    ).all()
    
    assert len(transactions) == 3  # One per account
    
    # Check split distribution
    spending_tx = next(t for t in transactions if "Spending" in t.description)
    saving_tx = next(t for t in transactions if "Saving" in t.description)
    giving_tx = next(t for t in transactions if "Giving" in t.description)
    
    assert spending_tx.amount == 6.00  # 60%
    assert saving_tx.amount == 2.00    # 20%
    assert giving_tx.amount == 2.00    # 20%
```

## Test Data Factories

```python
# tests/factories.py
import factory
from datetime import datetime, timedelta
from app.models import User, Family, Account, Chore, ChoreAssignment
from app.core.security import hash_password

class FamilyFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = Family
        sqlalchemy_session = None  # Set in conftest.py
    
    id = factory.Faker('uuid4')
    name = factory.Faker('company')
    timezone = 'America/New_York'
    currency = 'USD'
    created_at = factory.LazyFunction(datetime.utcnow)

class UserFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session = None
    
    id = factory.Faker('uuid4')
    email = factory.Faker('email')
    name = factory.Faker('name')
    password_hash = factory.LazyFunction(lambda: hash_password('password123'))
    role = 'CHILD'
    birthdate = factory.Faker('date_of_birth', minimum_age=5, maximum_age=17)
    created_at = factory.LazyFunction(datetime.utcnow)

class AccountFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = Account
        sqlalchemy_session = None
    
    id = factory.Faker('uuid4')
    name = 'Spending'
    account_type = 'SPENDING'
    balance = 0.00
    allow_negative = False
    family = factory.SubFactory(FamilyFactory)
    user = factory.SubFactory(UserFactory)

class ChoreFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = Chore
        sqlalchemy_session = None
    
    id = factory.Faker('uuid4')
    title = factory.Faker('sentence', nb_words=3)
    description = factory.Faker('text')
    reward_amount = 5.00
    assignment_type = 'ASSIGNED'
    require_approval = True
    family = factory.SubFactory(FamilyFactory)
    created_by = factory.SubFactory(UserFactory)

# Usage in tests
def test_transfer_between_accounts(db_session):
    family = FamilyFactory()
    user = UserFactory(family=family)
    account1 = AccountFactory(user=user, family=family, balance=100.00)
    account2 = AccountFactory(user=user, family=family, balance=0.00)
    
    transfer(account1, account2, 50.00)
    
    db_session.commit()
    db_session.refresh(account1)
    db_session.refresh(account2)
    
    assert account1.balance == 50.00
    assert account2.balance == 50.00
```

## Pytest Configuration

```python
# tests/conftest.py
import pytest
from fastapi.testclient import TestClient
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from app.main import app
from app.database import Base, get_db
from app.core.security import create_access_token

# Test database URL
TEST_DATABASE_URL = "postgresql://picklesapp:dev_password@localhost:5432/picklesapp_test"

@pytest.fixture(scope="session")
def engine():
    """Create test database engine"""
    engine = create_engine(TEST_DATABASE_URL)
    Base.metadata.create_all(bind=engine)
    yield engine
    Base.metadata.drop_all(bind=engine)

@pytest.fixture(scope="function")
def db_session(engine):
    """Create a new database session for each test"""
    SessionLocal = sessionmaker(bind=engine)
    session = SessionLocal()
    
    # Set factory session
    import tests.factories as factories
    factories.FamilyFactory._meta.sqlalchemy_session = session
    factories.UserFactory._meta.sqlalchemy_session = session
    factories.AccountFactory._meta.sqlalchemy_session = session
    
    yield session
    
    session.rollback()
    session.close()

@pytest.fixture
def client(db_session):
    """Create test client with database session override"""
    def override_get_db():
        try:
            yield db_session
        finally:
            pass
    
    app.dependency_overrides[get_db] = override_get_db
    
    with TestClient(app) as test_client:
        yield test_client
    
    app.dependency_overrides.clear()

@pytest.fixture
def auth_headers_parent(db_session):
    """Create auth headers for parent user"""
    family = FamilyFactory()
    parent = UserFactory(role='PARENT', family=family)
    db_session.commit()
    
    token = create_access_token({
        "sub": str(parent.id),
        "family_id": str(family.id),
        "role": "PARENT"
    })
    
    return {"Authorization": f"Bearer {token}"}

@pytest.fixture
def auth_headers_child(db_session):
    """Create auth headers for child user"""
    family = FamilyFactory()
    child = UserFactory(role='CHILD', family=family)
    db_session.commit()
    
    token = create_access_token({
        "sub": str(child.id),
        "family_id": str(family.id),
        "role": "CHILD"
    })
    
    return {"Authorization": f"Bearer {token}"}
```

## Running Tests

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=app --cov-report=html

# Run specific test file
pytest tests/integration/test_chore_workflow.py

# Run tests matching pattern
pytest -k "test_multi_household"

# Run tests with verbose output
pytest -v

# Run tests in parallel (faster)
pytest -n auto

# Stop on first failure
pytest -x

# View coverage report
open htmlcov/index.html
```

## Load Testing (Phase 9)

```python
# tests/load/locustfile.py
from locust import HttpUser, task, between
import random

class PicklesAppUser(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        """Login before starting tasks"""
        response = self.client.post("/api/v1/auth/login", json={
            "email": f"user{random.randint(1, 100)}@test.com",
            "password": "password123"
        })
        
        if response.status_code == 200:
            data = response.json()["data"]
            self.token = data["access_token"]
            self.family_id = data["user"]["families"][0]["id"]
            self.headers = {
                "Authorization": f"Bearer {self.token}",
                "X-Family-Context": self.family_id
            }
    
    @task(5)
    def view_dashboard(self):
        """Most common action: view dashboard data"""
        self.client.get("/api/v1/accounts", headers=self.headers)
        self.client.get("/api/v1/chore-assignments", headers=self.headers)
    
    @task(3)
    def view_chores(self):
        """View available chores"""
        self.client.get("/api/v1/chores", headers=self.headers)
    
    @task(2)
    def view_transactions(self):
        """View transaction history"""
        self.client.get("/api/v1/transactions?page=1&limit=20", headers=self.headers)
    
    @task(1)
    def complete_chore(self):
        """Complete a chore (less frequent)"""
        # Get assignments
        response = self.client.get("/api/v1/chore-assignments", headers=self.headers)
        if response.status_code == 200:
            assignments = response.json()["data"]["items"]
            pending = [a for a in assignments if a["status"] == "PENDING"]
            
            if pending:
                assignment_id = pending[0]["id"]
                self.client.post(
                    f"/api/v1/chore-assignments/{assignment_id}/complete",
                    headers=self.headers
                )

# Run load test
# locust -f tests/load/locustfile.py --host=https://api.picklesapp.com
# Then open http://localhost:8089 to configure and start test
```

## Continuous Integration

```yaml
# .github/workflows/test.yml
name: Backend Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: picklesapp
          POSTGRES_PASSWORD: test_password
          POSTGRES_DB: picklesapp_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
      
      - name: Run tests with coverage
        env:
          DATABASE_URL: postgresql://picklesapp:test_password@localhost:5432/picklesapp_test
        run: |
          pytest --cov=app --cov-report=xml --cov-report=term
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
```

## Best Practices

1. **Test Isolation**: Each test should be independent and not rely on others
2. **Clear Test Names**: Use descriptive names that explain what is being tested
3. **Arrange-Act-Assert**: Structure tests with clear setup, execution, and verification
4. **Mock External Services**: Don't make real API calls to OAuth providers, AWS, etc.
5. **Test Edge Cases**: Include tests for boundary conditions and error scenarios
6. **Fast Tests**: Unit tests should run in milliseconds, integration tests in seconds
7. **Consistent Factories**: Use factories for all test data creation
8. **Clean Up**: Tests should clean up after themselves (handled by fixtures)

## Next Steps

- Review [Database Schema](DATABASE_SCHEMA.md) for data model understanding
- Check [API Specifications](../shared/API_SPECIFICATIONS.md) for endpoint contracts
- Read [Local Development Setup](LOCAL_DEVELOPMENT_SETUP.md) for environment configuration
