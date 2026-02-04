# Phase 6: Goals, OAuth & Advanced Features
**Duration: 2-3 weeks**

---

## Overview

Phase 6 adds OAuth 2.0 authentication (Google, Apple, Facebook), savings goals, budgets, loans, and real-time updates. This phase significantly expands both authentication options and financial features.

---

## Goals

- **OAuth 2.0 authentication** (Google, Apple, Facebook)
- Savings goals tracking
- Budget worksheets
- Loan tracking
- Real-time updates

---

## Deliverables

### 1. Backend

- **OAuth 2.0 integration** (Google, Apple, Facebook via authlib)
- Account linking (connect social to existing email account)
- SavingsGoal, Budget, Loan models
- Goal progress calculations
- Budget tracking
- Loan amortization
- Weekly email report generator (SES)
- Transfer request workflow
- WebSocket support (Socket.io)

### 2. Frontend

- **OAuth login buttons** (Google, Apple, Facebook)
- Account linking UI
- Savings goal creation/tracking UI
- Progress visualization (progress bars, charts)
- Budget worksheet
- Loan calculator and tracker
- Transfer request UI
- Real-time notification badges (WebSocket)

### 3. Features

- **Social login/registration**
- **Link social accounts to existing account**
- Visual goal progress
- Target date projections
- Budget categories
- Income vs expense tracking
- Loan repayment schedules
- Weekly email summary
- Transfer request approval
- Real-time chore completion updates

---

## OAuth 2.0 Implementation

### Database Schema

```sql
-- OAuth accounts table
CREATE TABLE oauth_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(20) NOT NULL CHECK (provider IN ('google', 'apple', 'facebook')),
    provider_user_id VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    access_token TEXT,
    refresh_token TEXT,
    token_expires_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    UNIQUE(provider, provider_user_id)
);

CREATE INDEX idx_oauth_accounts_user ON oauth_accounts(user_id);
CREATE INDEX idx_oauth_accounts_provider ON oauth_accounts(provider, provider_user_id);
```

### OAuth Configuration

```python
# app/auth/oauth.py

from authlib.integrations.starlette_client import OAuth
from starlette.config import Config

oauth = OAuth()

# Google OAuth
oauth.register(
    name='google',
    client_id=settings.GOOGLE_CLIENT_ID,
    client_secret=settings.GOOGLE_CLIENT_SECRET,
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid email profile'}
)

# Apple OAuth
oauth.register(
    name='apple',
    client_id=settings.APPLE_CLIENT_ID,
    client_secret=settings.APPLE_CLIENT_SECRET,
    authorize_url='https://appleid.apple.com/auth/authorize',
    access_token_url='https://appleid.apple.com/auth/token',
    client_kwargs={'scope': 'name email'}
)

# Facebook OAuth
oauth.register(
    name='facebook',
    client_id=settings.FACEBOOK_CLIENT_ID,
    client_secret=settings.FACEBOOK_CLIENT_SECRET,
    authorize_url='https://www.facebook.com/v18.0/dialog/oauth',
    access_token_url='https://graph.facebook.com/v18.0/oauth/access_token',
    api_base_url='https://graph.facebook.com/v18.0/',
    client_kwargs={'scope': 'email public_profile'}
)
```

### OAuth Endpoints

```python
# app/routers/auth.py

@router.get("/auth/{provider}")
async def oauth_login(
    provider: str,
    request: Request,
    link_to_user: Optional[UUID] = None  # For account linking
):
    """
    Redirect to OAuth provider for login/registration.
    """
    if provider not in ['google', 'apple', 'facebook']:
        raise HTTPException(400, "Unsupported provider")
    
    redirect_uri = f"{settings.API_URL}/auth/{provider}/callback"
    
    # Store linking info in session
    if link_to_user:
        request.session['link_to_user'] = str(link_to_user)
    
    return await oauth.create_client(provider).authorize_redirect(request, redirect_uri)

@router.get("/auth/{provider}/callback")
async def oauth_callback(
    provider: str,
    request: Request,
    db: Session = Depends(get_db)
):
    """
    Handle OAuth callback and create/login user.
    """
    client = oauth.create_client(provider)
    token = await client.authorize_access_token(request)
    
    # Get user info from provider
    if provider == 'google':
        user_info = token.get('userinfo')
    elif provider == 'facebook':
        resp = await client.get('me', params={'fields': 'id,name,email,picture'})
        user_info = resp.json()
    elif provider == 'apple':
        user_info = parse_apple_id_token(token['id_token'])
    
    provider_user_id = user_info.get('sub') or user_info.get('id')
    email = user_info.get('email')
    
    # Check if linking to existing account
    link_to_user = request.session.pop('link_to_user', None)
    
    if link_to_user:
        return await link_oauth_account(link_to_user, provider, provider_user_id, email, token, db)
    
    # Check if OAuth account exists
    oauth_account = db.query(OAuthAccount).filter(
        OAuthAccount.provider == provider,
        OAuthAccount.provider_user_id == provider_user_id
    ).first()
    
    if oauth_account:
        # Existing OAuth account - login
        user = oauth_account.user
    else:
        # Check if email exists (for account linking prompt)
        existing_user = db.query(User).filter(User.email == email).first()
        
        if existing_user:
            # Prompt to link accounts
            return RedirectResponse(
                f"{settings.FRONTEND_URL}/link-account?email={email}&provider={provider}"
            )
        
        # Create new user
        user = User(
            email=email,
            first_name=user_info.get('given_name', user_info.get('name', '').split()[0]),
            last_name=user_info.get('family_name', ''),
            email_verified=True  # OAuth emails are verified
        )
        db.add(user)
        db.flush()
        
        # Create OAuth account
        oauth_account = OAuthAccount(
            user_id=user.id,
            provider=provider,
            provider_user_id=provider_user_id,
            email=email,
            access_token=token.get('access_token'),
            refresh_token=token.get('refresh_token')
        )
        db.add(oauth_account)
        db.commit()
    
    # Generate JWT tokens
    access_token = create_access_token(user)
    refresh_token = create_refresh_token(user)
    
    # Redirect to frontend with tokens
    return RedirectResponse(
        f"{settings.FRONTEND_URL}/auth/callback?access_token={access_token}&refresh_token={refresh_token}"
    )

async def link_oauth_account(
    user_id: UUID,
    provider: str,
    provider_user_id: str,
    email: str,
    token: dict,
    db: Session
):
    """
    Link OAuth account to existing user.
    """
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(404, "User not found")
    
    # Check if OAuth account already linked to another user
    existing = db.query(OAuthAccount).filter(
        OAuthAccount.provider == provider,
        OAuthAccount.provider_user_id == provider_user_id
    ).first()
    
    if existing:
        raise HTTPException(400, f"This {provider} account is already linked to another user")
    
    # Create OAuth account link
    oauth_account = OAuthAccount(
        user_id=user.id,
        provider=provider,
        provider_user_id=provider_user_id,
        email=email,
        access_token=token.get('access_token'),
        refresh_token=token.get('refresh_token')
    )
    db.add(oauth_account)
    db.commit()
    
    return RedirectResponse(f"{settings.FRONTEND_URL}/settings/accounts?linked={provider}")
```

### Frontend OAuth Buttons

```jsx
// components/auth/OAuthButtons.jsx

import { GoogleIcon, AppleIcon, FacebookIcon } from '@/components/icons';

export function OAuthButtons({ mode = 'login', linkToUser = null }) {
  const providers = [
    { id: 'google', name: 'Google', icon: GoogleIcon, color: 'bg-white hover:bg-gray-50' },
    { id: 'apple', name: 'Apple', icon: AppleIcon, color: 'bg-black hover:bg-gray-900 text-white' },
    { id: 'facebook', name: 'Facebook', icon: FacebookIcon, color: 'bg-blue-600 hover:bg-blue-700 text-white' }
  ];

  const handleOAuth = (provider) => {
    const baseUrl = `${import.meta.env.VITE_API_URL}/auth/${provider}`;
    const url = linkToUser 
      ? `${baseUrl}?link_to_user=${linkToUser}`
      : baseUrl;
    window.location.href = url;
  };

  return (
    <div className="space-y-3">
      {providers.map(({ id, name, icon: Icon, color }) => (
        <button
          key={id}
          onClick={() => handleOAuth(id)}
          className={`w-full flex items-center justify-center gap-3 px-4 py-3 rounded-lg border ${color}`}
        >
          <Icon className="w-5 h-5" />
          <span>{mode === 'login' ? 'Continue with' : 'Link'} {name}</span>
        </button>
      ))}
    </div>
  );
}
```

---

## Savings Goals

### Database Schema

```sql
CREATE TABLE savings_goals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    target_amount DECIMAL(10, 2) NOT NULL,
    current_amount DECIMAL(10, 2) DEFAULT 0,
    target_date DATE,
    icon VARCHAR(50),  -- Emoji or icon name
    color VARCHAR(7),  -- Hex color
    is_active BOOLEAN DEFAULT true,
    completed_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_savings_goals_user ON savings_goals(user_id);
CREATE INDEX idx_savings_goals_account ON savings_goals(account_id);
```

### Goal Progress Calculation

```python
def calculate_goal_progress(goal: SavingsGoal) -> dict:
    """
    Calculate goal progress and projections.
    """
    progress_percent = (goal.current_amount / goal.target_amount * 100) if goal.target_amount else 0
    remaining = goal.target_amount - goal.current_amount
    
    result = {
        'progress_percent': min(progress_percent, 100),
        'remaining_amount': max(remaining, 0),
        'is_complete': goal.current_amount >= goal.target_amount
    }
    
    if goal.target_date and remaining > 0:
        days_remaining = (goal.target_date - date.today()).days
        if days_remaining > 0:
            result['daily_savings_needed'] = remaining / days_remaining
            result['weekly_savings_needed'] = remaining / (days_remaining / 7)
            result['on_track'] = calculate_on_track_status(goal)
        else:
            result['overdue'] = True
    
    return result

def calculate_projected_completion_date(goal: SavingsGoal) -> Optional[date]:
    """
    Project when goal will be reached based on savings rate.
    """
    # Get recent deposits to this account
    thirty_days_ago = datetime.utcnow() - timedelta(days=30)
    recent_deposits = db.query(func.sum(Transaction.amount)).filter(
        Transaction.account_id == goal.account_id,
        Transaction.amount > 0,
        Transaction.created_at >= thirty_days_ago
    ).scalar() or 0
    
    if recent_deposits <= 0:
        return None
    
    daily_rate = recent_deposits / 30
    remaining = goal.target_amount - goal.current_amount
    
    if remaining <= 0:
        return date.today()
    
    days_to_goal = remaining / daily_rate
    return date.today() + timedelta(days=int(days_to_goal))
```

### Goal Frontend Component

```jsx
// components/goals/GoalCard.jsx

export function GoalCard({ goal }) {
  const progress = useMemo(() => calculateProgress(goal), [goal]);
  
  return (
    <div className="bg-white rounded-xl shadow p-6">
      <div className="flex items-center gap-3 mb-4">
        <span className="text-3xl">{goal.icon || '🎯'}</span>
        <div>
          <h3 className="font-semibold text-lg">{goal.name}</h3>
          <p className="text-sm text-gray-500">
            ${goal.current_amount.toFixed(2)} of ${goal.target_amount.toFixed(2)}
          </p>
        </div>
      </div>
      
      <div className="mb-4">
        <div className="h-4 bg-gray-200 rounded-full overflow-hidden">
          <div 
            className="h-full bg-green-500 transition-all duration-500"
            style={{ 
              width: `${progress.progress_percent}%`,
              backgroundColor: goal.color || '#10B981'
            }}
          />
        </div>
        <p className="text-right text-sm text-gray-500 mt-1">
          {progress.progress_percent.toFixed(0)}%
        </p>
      </div>
      
      {goal.target_date && !progress.is_complete && (
        <div className="text-sm">
          {progress.on_track ? (
            <p className="text-green-600">
              ✓ On track! Save ${progress.weekly_savings_needed?.toFixed(2)}/week
            </p>
          ) : (
            <p className="text-orange-600">
              ⚠ Need ${progress.weekly_savings_needed?.toFixed(2)}/week to reach goal
            </p>
          )}
        </div>
      )}
      
      {progress.is_complete && (
        <div className="text-center py-2">
          <span className="text-2xl">🎉</span>
          <p className="text-green-600 font-semibold">Goal Reached!</p>
        </div>
      )}
    </div>
  );
}
```

---

## Budget Worksheets

### Database Schema

```sql
CREATE TABLE budgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    period VARCHAR(20) NOT NULL CHECK (period IN ('weekly', 'monthly')),
    start_date DATE NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE budget_categories (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    budget_id UUID NOT NULL REFERENCES budgets(id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL,
    budgeted_amount DECIMAL(10, 2) NOT NULL,
    spent_amount DECIMAL(10, 2) DEFAULT 0,
    icon VARCHAR(50),
    color VARCHAR(7),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_budgets_user ON budgets(user_id);
CREATE INDEX idx_budget_categories_budget ON budget_categories(budget_id);
```

### Budget Categories (Default)

```python
DEFAULT_BUDGET_CATEGORIES = [
    {'name': 'Entertainment', 'icon': '🎮', 'color': '#8B5CF6'},
    {'name': 'Food & Snacks', 'icon': '🍕', 'color': '#F59E0B'},
    {'name': 'Clothes', 'icon': '👕', 'color': '#EC4899'},
    {'name': 'Savings', 'icon': '🏦', 'color': '#10B981'},
    {'name': 'Gifts', 'icon': '🎁', 'color': '#EF4444'},
    {'name': 'Activities', 'icon': '⚽', 'color': '#3B82F6'},
    {'name': 'Other', 'icon': '📦', 'color': '#6B7280'},
]
```

---

## Loan Tracking

### Database Schema

```sql
CREATE TABLE loans (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    lender_id UUID NOT NULL REFERENCES users(id),  -- Parent
    borrower_id UUID NOT NULL REFERENCES users(id),  -- Child
    name VARCHAR(100) NOT NULL,
    principal_amount DECIMAL(10, 2) NOT NULL,
    interest_rate DECIMAL(5, 4) DEFAULT 0,  -- Annual rate
    remaining_balance DECIMAL(10, 2) NOT NULL,
    monthly_payment DECIMAL(10, 2),
    status VARCHAR(20) NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'paid_off', 'forgiven')),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    paid_off_at TIMESTAMP WITH TIME ZONE
);

CREATE TABLE loan_payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    loan_id UUID NOT NULL REFERENCES loans(id) ON DELETE CASCADE,
    amount DECIMAL(10, 2) NOT NULL,
    principal_portion DECIMAL(10, 2) NOT NULL,
    interest_portion DECIMAL(10, 2) NOT NULL,
    balance_after DECIMAL(10, 2) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

### Loan Amortization Calculator

```python
from decimal import Decimal
from math import pow, log

def calculate_loan_schedule(
    principal: Decimal,
    annual_rate: Decimal,
    monthly_payment: Decimal
) -> list:
    """
    Calculate loan amortization schedule.
    """
    if annual_rate == 0:
        # No interest - simple division
        num_payments = int(principal / monthly_payment) + 1
        return [{
            'payment_number': i + 1,
            'payment': min(monthly_payment, principal - (monthly_payment * i)),
            'principal': min(monthly_payment, principal - (monthly_payment * i)),
            'interest': Decimal('0'),
            'balance': max(Decimal('0'), principal - (monthly_payment * (i + 1)))
        } for i in range(num_payments)]
    
    monthly_rate = annual_rate / 12
    schedule = []
    balance = principal
    payment_num = 0
    
    while balance > 0:
        payment_num += 1
        interest = balance * monthly_rate
        principal_payment = min(monthly_payment - interest, balance)
        actual_payment = principal_payment + interest
        balance = balance - principal_payment
        
        schedule.append({
            'payment_number': payment_num,
            'payment': actual_payment.quantize(Decimal('0.01')),
            'principal': principal_payment.quantize(Decimal('0.01')),
            'interest': interest.quantize(Decimal('0.01')),
            'balance': max(Decimal('0'), balance.quantize(Decimal('0.01')))
        })
        
        if payment_num > 120:  # Safety limit (10 years)
            break
    
    return schedule
```

---

## Real-Time Updates (WebSocket)

### Backend WebSocket Setup

```python
# app/websocket.py

from fastapi import WebSocket, WebSocketDisconnect
from typing import Dict, Set
import json

class ConnectionManager:
    def __init__(self):
        # family_id -> set of websocket connections
        self.active_connections: Dict[str, Set[WebSocket]] = {}
    
    async def connect(self, websocket: WebSocket, family_id: str):
        await websocket.accept()
        if family_id not in self.active_connections:
            self.active_connections[family_id] = set()
        self.active_connections[family_id].add(websocket)
    
    def disconnect(self, websocket: WebSocket, family_id: str):
        self.active_connections[family_id].discard(websocket)
    
    async def broadcast_to_family(self, family_id: str, message: dict):
        """Send message to all connections in a family."""
        if family_id in self.active_connections:
            for connection in self.active_connections[family_id]:
                try:
                    await connection.send_json(message)
                except:
                    pass  # Connection may be stale

manager = ConnectionManager()

@app.websocket("/ws/{family_id}")
async def websocket_endpoint(
    websocket: WebSocket,
    family_id: str,
    token: str = Query(...)
):
    # Verify token and family access
    try:
        payload = verify_token(token)
        if not verify_family_access(payload['user_id'], family_id):
            await websocket.close(code=4003)
            return
    except:
        await websocket.close(code=4001)
        return
    
    await manager.connect(websocket, family_id)
    try:
        while True:
            data = await websocket.receive_text()
            # Handle incoming messages if needed
    except WebSocketDisconnect:
        manager.disconnect(websocket, family_id)

# Usage in endpoints
async def complete_chore(assignment_id: UUID, ...):
    # ... complete chore logic ...
    
    # Broadcast to family
    await manager.broadcast_to_family(str(family_id), {
        'type': 'CHORE_COMPLETED',
        'data': {
            'assignment_id': str(assignment_id),
            'chore_name': chore.name,
            'completed_by': user.first_name
        }
    })
```

### Frontend WebSocket Hook

```jsx
// hooks/useWebSocket.js

import { useEffect, useRef, useCallback } from 'react';
import { useAuth } from '@/contexts/AuthContext';
import { useFamily } from '@/contexts/FamilyContext';

export function useWebSocket(onMessage) {
  const { token } = useAuth();
  const { currentFamily } = useFamily();
  const wsRef = useRef(null);
  
  useEffect(() => {
    if (!token || !currentFamily) return;
    
    const ws = new WebSocket(
      `${import.meta.env.VITE_WS_URL}/ws/${currentFamily.id}?token=${token}`
    );
    
    ws.onopen = () => console.log('WebSocket connected');
    ws.onclose = () => console.log('WebSocket disconnected');
    ws.onerror = (error) => console.error('WebSocket error:', error);
    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      onMessage(data);
    };
    
    wsRef.current = ws;
    
    return () => ws.close();
  }, [token, currentFamily?.id, onMessage]);
  
  return wsRef.current;
}

// Usage in component
function Dashboard() {
  const { refetchChores } = useChores();
  
  const handleMessage = useCallback((message) => {
    switch (message.type) {
      case 'CHORE_COMPLETED':
        toast.success(`${message.data.completed_by} completed "${message.data.chore_name}"!`);
        refetchChores();
        break;
      case 'CHORE_APPROVED':
        toast.success(`Chore approved! $${message.data.reward} earned`);
        break;
      // ... other message types
    }
  }, [refetchChores]);
  
  useWebSocket(handleMessage);
  
  return <DashboardContent />;
}
```

---

## Weekly Email Reports

### Email Template

```python
# app/services/email.py

def generate_weekly_report(family_id: UUID) -> dict:
    """Generate weekly report data for a family."""
    week_start = date.today() - timedelta(days=7)
    
    # Get family members
    members = get_family_members(family_id)
    
    report = {
        'family_name': get_family(family_id).name,
        'week_of': week_start.strftime('%B %d, %Y'),
        'members': []
    }
    
    for member in members:
        if member.role == 'CHILD':
            member_data = {
                'name': member.first_name,
                'chores_completed': count_completed_chores(member.id, week_start),
                'chores_pending': count_pending_chores(member.id),
                'money_earned': sum_earnings(member.id, week_start),
                'account_balances': get_account_balances(member.id, family_id)
            }
            report['members'].append(member_data)
    
    return report

async def send_weekly_reports():
    """Send weekly email reports to all parents."""
    families = get_all_families_with_email_reports_enabled()
    
    for family in families:
        report = generate_weekly_report(family.id)
        parents = get_parents(family.id)
        
        for parent in parents:
            await send_email(
                to=parent.email,
                subject=f"PicklesApp Weekly Summary - {report['week_of']}",
                template='weekly_report',
                context=report
            )
```

---

## API Endpoints

### OAuth

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/auth/{provider}` | GET | Initiate OAuth flow |
| `/api/v1/auth/{provider}/callback` | GET | OAuth callback |
| `/api/v1/auth/accounts` | GET | List linked OAuth accounts |
| `/api/v1/auth/accounts/{provider}` | DELETE | Unlink OAuth account |

### Goals

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/goals` | GET | List user's savings goals |
| `/api/v1/goals` | POST | Create savings goal |
| `/api/v1/goals/{id}` | GET | Get goal details with progress |
| `/api/v1/goals/{id}` | PUT | Update goal |
| `/api/v1/goals/{id}` | DELETE | Delete goal |
| `/api/v1/goals/{id}/contribute` | POST | Add money to goal |

### Budgets

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/budgets` | GET | List budgets |
| `/api/v1/budgets` | POST | Create budget |
| `/api/v1/budgets/{id}` | GET | Get budget with spending |
| `/api/v1/budgets/{id}/categories` | PUT | Update categories |
| `/api/v1/budgets/{id}/log` | POST | Log expense to category |

### Loans

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/loans` | GET | List loans |
| `/api/v1/loans` | POST | Create loan |
| `/api/v1/loans/{id}` | GET | Get loan with schedule |
| `/api/v1/loans/{id}/payments` | POST | Make payment |
| `/api/v1/loans/{id}/forgive` | POST | Forgive remaining balance |

---

## Testing Requirements

### OAuth Tests

```python
def test_google_oauth_new_user(mock_google):
    """Test OAuth registration with Google."""
    mock_google.return_value = {
        'sub': 'google-123',
        'email': 'newuser@gmail.com',
        'given_name': 'New',
        'family_name': 'User'
    }
    
    response = client.get('/auth/google/callback', params={'code': 'test-code'})
    
    assert response.status_code == 302
    assert 'access_token=' in response.headers['location']
    
    user = db.query(User).filter(User.email == 'newuser@gmail.com').first()
    assert user is not None
    assert user.oauth_accounts[0].provider == 'google'

def test_oauth_link_existing_account():
    """Test linking OAuth to existing email account."""
    user = create_user(email='existing@test.com')
    
    response = client.get(
        '/auth/google',
        params={'link_to_user': str(user.id)},
        headers=auth_headers(user)
    )
    
    # Simulate callback
    # Verify OAuth account linked to user

def test_goal_progress_calculation():
    """Test savings goal progress."""
    goal = create_goal(target_amount=100, current_amount=75)
    
    progress = calculate_goal_progress(goal)
    
    assert progress['progress_percent'] == 75
    assert progress['remaining_amount'] == 25
    assert not progress['is_complete']
```

### Coverage Requirements

- OAuth flows: 100%
- Account linking: 100%
- Goal calculations: 100%
- Budget tracking: 80%
- Loan amortization: 100%
- WebSocket: 80%
- Email delivery: 80%

---

## Definition of Done

### Must Have
- [ ] Google OAuth working (login + registration)
- [ ] Apple OAuth working
- [ ] Facebook OAuth working
- [ ] Account linking (OAuth to existing email account)
- [ ] Savings goals with progress tracking
- [ ] Target date projections
- [ ] Budget categories
- [ ] Loan calculator with amortization
- [ ] Weekly email summary
- [ ] Real-time notifications via WebSocket
- [ ] All tests passing with required coverage

### Nice to Have
- [ ] Goal achievement celebrations
- [ ] Budget vs actual charts
- [ ] Multiple loan comparison
- [ ] Push notifications (prep for Phase 10)

---

## Dependencies

- **Requires**: Phase 5 (AWS deployment for OAuth callback URLs)
- **Enables**: Phase 7 (age-adaptive UI), Phase 10 (native push notifications)

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| OAuth provider downtime | Fallback to email/password login |
| WebSocket connection drops | Auto-reconnect with exponential backoff |
| Account linking confusion | Clear UI guidance, confirmation emails |
| Email delivery issues | Monitor SES metrics, implement retry |

---

*Phase 6 Document - Last Updated: February 2026*
