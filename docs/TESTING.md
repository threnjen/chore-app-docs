
**Unit Tests:**
```python
# tests/unit/test_auth.py
def test_password_hashing():
    password = "mysecurepassword"
    hashed = hash_password(password)
    assert verify_password(password, hashed) is True
    assert verify_password("wrongpassword", hashed) is False

# tests/unit/test_transactions.py
def test_transaction_updates_balance():
    account = create_account(balance=100)
    transaction = create_credit(account, amount=25)
    assert account.balance == 125
    assert transaction.balance_after == 125
```

**Integration Tests:**
```python
# tests/integration/test_chore_workflow.py
def test_complete_chore_credits_reward(client, auth_headers):
    # Create chore
    chore = create_chore(reward_amount=10)
    assignment = create_assignment(chore)
    
    # Complete as child
    response = client.post(
        f"/chore-assignments/{assignment.id}/complete",
        headers=auth_headers_child
    )
    assert response.status_code == 200
    
    # Approve as parent
    response = client.post(
        f"/chore-assignments/{assignment.id}/approve",
        headers=auth_headers_parent
    )
    assert response.status_code == 200
    
    # Verify balance updated
    account = get_account(child_id)
    assert account.balance == 10
```

**Multi-Tenant Isolation Tests:**
```python
def test_family_data_isolation(client):
    # User in Family A tries to access Family B's data
    response = client.get(
        "/chores",
        headers={"Authorization": f"Bearer {family_a_token}"},
        headers={"X-Family-Context": family_b_id}  # Mismatch
    )
    assert response.status_code == 403
```

### Frontend Testing (Vitest + RTL)

**Component Tests:**
```javascript
// tests/components/ChoreCard.test.jsx
import { render, screen, fireEvent } from '@testing-library/react';
import ChoreCard from '@/components/ChoreCard';

test('displays chore title and reward', () => {
  const chore = { title: 'Clean room', reward_amount: 5 };
  render(<ChoreCard chore={chore} />);
  
  expect(screen.getByText('Clean room')).toBeInTheDocument();
  expect(screen.getByText('$5')).toBeInTheDocument();
});

test('calls onComplete when button clicked', () => {
  const onComplete = vi.fn();
  render(<ChoreCard chore={chore} onComplete={onComplete} />);
  
  fireEvent.click(screen.getByText('Mark Complete'));
  expect(onComplete).toHaveBeenCalledWith(chore.id);
});
```

**Integration Tests:**
```javascript
// tests/integration/ChoreFlow.test.jsx
test('complete chore flow', async () => {
  const { user } = renderWithAuth(<Dashboard />);
  
  // See chore
  expect(screen.getByText('Clean your room')).toBeInTheDocument();
  
  // Complete
  await user.click(screen.getByText('Mark Complete'));
  expect(screen.getByText('Waiting for approval')).toBeInTheDocument();
  
  // Parent approves (switch context)
  await switchToParentView();
  await user.click(screen.getByText('Approve'));
  
  // Verify reward
  expect(screen.getByText('$5 credited')).toBeInTheDocument();
});
```

### E2E Testing (Playwright)

```javascript
// tests/e2e/chore-workflow.spec.js
test('parent creates chore and child completes it', async ({ page, browser }) => {
  // Parent login
  await page.goto('http://localhost:5173/login');
  await page.fill('[name="email"]', 'parent@test.com');
  await page.fill('[name="password"]', 'password123');
  await page.click('button[type="submit"]');
  
  // Create chore
  await page.click('text=Create Chore');
  await page.fill('[name="title"]', 'Test Chore');
  await page.fill('[name="reward_amount"]', '10');
  await page.selectOption('[name="assigned_to"]', 'Emma');
  await page.click('button:has-text("Create")');
  
  // Child login in new context
  const childContext = await browser.newContext();
  const childPage = await childContext.newPage();
  await childPage.goto('http://localhost:5173/login');
  await childPage.fill('[name="email"]', 'emma@test.com');
  await childPage.fill('[name="password"]', 'password123');
  await childPage.click('button[type="submit"]');
  
  // Complete chore
  await childPage.click('text=Test Chore');
  await childPage.click('button:has-text("Mark Complete")');
  
  // Parent approves
  await page.reload();
  await page.click('text=Pending Approvals');
  await page.click('text=Test Chore');
  await page.click('button:has-text("Approve")');
  
  // Verify transaction
  await page.click('text=Transactions');
  await expect(page.locator('text=Chore Reward: Test Chore')).toBeVisible();
  await expect(page.locator('text=$10.00')).toBeVisible();
});
```

### Load Testing (Phase 9)

```bash
# Using Locust
# locustfile.py
from locust import HttpUser, task, between

class PicklesAppUser(HttpUser):
    wait_time = between(1, 3)
    
    def on_start(self):
        # Login
        response = self.client.post("/api/v1/auth/login", json={
            "email": "parent@test.com",
            "password": "password123"
        })
        self.token = response.json()["data"]["access_token"]
        self.headers = {"Authorization": f"Bearer {self.token}"}
    
    @task(3)
    def view_chores(self):
        self.client.get("/api/v1/chores", headers=self.headers)
    
    @task(2)
    def view_accounts(self):
        self.client.get("/api/v1/accounts", headers=self.headers)
    
    @task(1)
    def create_transaction(self):
        self.client.post("/api/v1/transactions", headers=self.headers, json={
            "account_id": "...",
            "transaction_type": "CREDIT",
            "amount": 10.00
        })

# Run load test
locust -f locustfile.py --host=https://api.picklesapp.com
# Simulate 100 concurrent users
```

### Test Data Factories

```python
# tests/factories.py
import factory
from app.models import User, Family, Account

class FamilyFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = Family
        sqlalchemy_session = db.session
    
    name = factory.Faker('company')
    timezone = 'America/New_York'
    currency = 'USD'

class UserFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session = db.session
    
    email = factory.Faker('email')
    name = factory.Faker('name')
    password_hash = factory.LazyFunction(lambda: hash_password('password123'))
    role = 'CHILD'
    birthdate = factory.Faker('date_of_birth', minimum_age=5, maximum_age=17)

class AccountFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = Account
        sqlalchemy_session = db.session
    
    name = 'Spending'
    account_type = 'SPENDING'
    balance = 0.00
    family = factory.SubFactory(FamilyFactory)
    user = factory.SubFactory(UserFactory)

# Usage in tests
def test_transfer():
    family = FamilyFactory()
    user = UserFactory(family=family)
    account1 = AccountFactory(user=user, balance=100)
    account2 = AccountFactory(user=user, balance=0)
    
    transfer(account1, account2, 50)
    
    assert account1.balance == 50
    assert account2.balance == 50
```