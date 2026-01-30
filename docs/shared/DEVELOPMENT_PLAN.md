# PicklesApp Development Plan
**Family Chore & Allowance Management Platform**

*"Your parenting, your way"*

---

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Technology Stack](#technology-stack)
4. [Multi-Household Architecture](#multi-household-architecture)
5. [Database Schema](#database-schema)
6. [API Specifications](#api-specifications)
7. [Chore Scheduling System](#chore-scheduling-system)
8. [Age-Adaptive UI System](#age-adaptive-ui-system)
9. [Development Phases](#development-phases)
10. [Local Development Setup](#local-development-setup)
11. [AWS Deployment Architecture](#aws-deployment-architecture)
12. [Security Framework](#security-framework)
13. [Testing Strategy](#testing-strategy)
14. [Cost Management](#cost-management)
15. [Business Model](#business-model)

---

## Executive Summary

PicklesApp is a family financial management platform that addresses critical pain points in existing solutions (particularly FamZoo):

### Core Differentiators
- **Advanced Chore Scheduling**: Custom recurrence patterns (M-W-F, bi-weekly), relative vs absolute timing
- **Multi-Household Support**: Kids in divorced/separated families can belong to multiple households with complete data isolation
- **Age-Adaptive Mobile UI**: Interface automatically adapts as children mature (Young 5-9, Tween 10-13, Teen 14-17)
- **Customization Philosophy**: "Your parenting, your way" - parents control negative balances, transfer limits, reward systems
- **Non-Monetary Rewards**: Support for points, stars, privileges in addition to money
- **Preset Parenting Profiles**: Age-appropriate defaults that parents can customize

### Technical Approach
- **Backend**: FastAPI (Python 3.11+) with async support
- **Frontend**: React 18+ with Vite, Context + Hooks for state
- **Database**: PostgreSQL 15+ with native installation (not containerized)
- **Containerization**: Docker for frontend/backend only
- **Deployment**: AWS Lightsail (POC), scaling to ECS/RDS (production)
- **Timeline**: 9 phases over ~20 weeks to production-ready

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Client Layer                         │
├──────────────────────┬──────────────────────────────────────┤
│   Desktop Web App    │        Mobile Web App (PWA)          │
│   (React + Vite)     │     (Age-Adaptive Components)        │
└──────────────────────┴──────────────────────────────────────┘
                              ↓ HTTPS/WSS
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                       │
│                  (FastAPI + Uvicorn)                        │
├─────────────────────────────────────────────────────────────┤
│  • REST API (v1)                                            │
│  • JWT Authentication + Social Auth (Google/Apple/Facebook) │
│  • Multi-Tenant Middleware (Family Context)                │
│  • Rate Limiting & Request Validation                       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     Business Logic Layer                     │
├──────────────┬──────────────┬──────────────┬───────────────┤
│ Auth Service │ Chore Engine │ Money Service│ Notification  │
│              │              │              │   Service     │
└──────────────┴──────────────┴──────────────┴───────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                       Data Layer                             │
├──────────────────────┬──────────────────────────────────────┤
│  PostgreSQL 15+      │        S3 Storage                    │
│  • Multi-Tenant RLS  │  • Profile Pictures                  │
│  • Connection Pool   │  • Future: Photo Evidence            │
└──────────────────────┴──────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                  Background Jobs Layer                       │
│              (APScheduler + PostgreSQL)                      │
├─────────────────────────────────────────────────────────────┤
│  • Allowance Processing                                     │
│  • Interest Calculations                                    │
│  • Chore Instance Generation                                │
│  • Weekly Email Reports                                     │
│  • Notification Delivery                                    │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

**Frontend (React + Vite)**
- Desktop-optimized web interface
- Mobile-optimized PWA with offline support
- Age-adaptive component rendering
- Family context switching UI
- Real-time updates via WebSockets (Phase 6+)

**Backend (FastAPI)**
- RESTful API with OpenAPI documentation
- JWT token management + OAuth 2.0 flows
- Multi-tenant data isolation middleware
- Business logic for chores, money, allowances
- Scheduled job management

**Database (PostgreSQL)**
- Multi-tenant data with Row-Level Security
- ACID compliance for financial transactions
- Full-text search capabilities
- JSON support for flexible settings

**Background Jobs (APScheduler)**
- Cron-like scheduling with PostgreSQL backend
- Allowance payment processing
- Chore instance generation from recurrence rules
- Email report generation
- Cleanup tasks

---

## Technology Stack

### Backend Stack
```yaml
Language: Python 3.11+
Framework: FastAPI 0.100+
ORM: SQLAlchemy 2.0+
Migrations: Alembic
Validation: Pydantic v2
Auth: 
  - python-jose (JWT)
  - passlib (bcrypt)
  - authlib (OAuth 2.0)
Scheduling: APScheduler 3.10+
Email: boto3 (AWS SES)
Testing:
  - pytest
  - pytest-asyncio
  - pytest-cov
  - factory-boy
Environment: python-dotenv
HTTP Client: httpx
```

### Frontend Stack
```yaml
Build Tool: Vite 5+
Framework: React 18+
Language: JavaScript (with JSX)
Routing: React Router v6
State Management: React Context + Hooks
HTTP Client: Axios
Forms: React Hook Form
Validation: Zod
UI Components: 
  - Tailwind CSS 3+
  - Headless UI
  - Heroicons
Date Handling: date-fns
Notifications: react-hot-toast
Testing:
  - Vitest
  - React Testing Library
  - Playwright (E2E)
PWA: vite-plugin-pwa
```

### Database
```yaml
Database: PostgreSQL 15+
Connection Pooling: SQLAlchemy pool (10-20 connections)
Backup: Automated daily snapshots
Extensions:
  - pgcrypto (UUID generation)
  - pg_trgm (fuzzy search)
```

### Infrastructure
```yaml
Containerization: Docker + Docker Compose
Development:
  - Native PostgreSQL installation
  - Docker for frontend/backend
  - Hot reload enabled
Production:
  - AWS Lightsail Containers (POC)
  - AWS ECS Fargate (Scale)
  - AWS RDS PostgreSQL (Production)
  - AWS S3 (Static assets)
  - AWS CloudFront (CDN)
  - AWS SES (Email)
  - AWS Route 53 (DNS)
IaC: Terraform
CI/CD: GitHub Actions
Monitoring:
  - AWS CloudWatch
  - Sentry (error tracking - optional)
```

---

## Multi-Household Architecture
**[Backend + Frontend]**

### Problem Statement
Children in divorced/separated families need to:
- Belong to multiple households (Mom's house, Dad's house)
- Maintain completely separate financial accounts per household
- Have separate chores, allowances, and transactions per household
- Switch between household contexts easily
- Ensure complete data isolation between households

### Solution: Multi-Tenant with Family Context Switching

#### Data Model

```python
# User can belong to multiple families via family_memberships
User (id, email, password_hash, name, birthdate, role: PARENT/CHILD)
  ↓
FamilyMembership (id, user_id, family_id, role, is_active, joined_at)
  ↓
Family (id, name, timezone, currency, created_at)
  ↓
Account (id, family_id, user_id, name, balance, type: SPENDING/SAVING/GIVING)
Chore (id, family_id, ...)
Transaction (id, family_id, account_id, ...)
Allowance (id, family_id, user_id, ...)
```

#### Context Switching Flow

```
1. Kid logs in with single credential (email/password or social)
   ↓
2. System queries FamilyMembership table
   → Finds user belongs to 2 families: Family A (Mom), Family B (Dad)
   ↓
3. UI shows: "Select Household: 
   - Mom's House 🏠
   - Dad's House 🏡"
   ↓
4. Kid selects Family A
   ↓
5. Backend sets family_context_id in JWT token or session
   ↓
6. All subsequent API calls automatically filtered by family_context_id
   ↓
7. Kid can switch context via menu: "Switch to Dad's House"
   ↓
8. New JWT token issued with updated family_context_id
```

#### Data Isolation Implementation

**Middleware Level (Primary)**
```python
# FastAPI dependency injection
async def get_current_family_context(
    token: str = Depends(oauth2_scheme),
    db: Session = Depends(get_db)
) -> FamilyContext:
    """
    Extracts family_id from JWT token.
    All database queries automatically scoped to this family.
    """
    payload = jwt.decode(token, SECRET_KEY)
    family_id = payload.get("family_id")
    
    # Verify user actually belongs to this family
    membership = db.query(FamilyMembership).filter(
        FamilyMembership.user_id == payload["user_id"],
        FamilyMembership.family_id == family_id,
        FamilyMembership.is_active == True
    ).first()
    
    if not membership:
        raise HTTPException(403, "Access denied to this family")
    
    return FamilyContext(family_id=family_id, user_id=payload["user_id"])

# All endpoints use this dependency
@router.get("/chores")
async def get_chores(
    context: FamilyContext = Depends(get_current_family_context),
    db: Session = Depends(get_db)
):
    # Query automatically scoped to context.family_id
    return db.query(Chore).filter(Chore.family_id == context.family_id).all()
```

**Database Level (Safety Net)**
```sql
-- PostgreSQL Row-Level Security (RLS)
ALTER TABLE chores ENABLE ROW LEVEL SECURITY;

CREATE POLICY chores_family_isolation ON chores
    USING (family_id = current_setting('app.current_family_id')::uuid);

-- Set family context at connection level
SET app.current_family_id = '<family_uuid>';
```

#### Moving Kids Between Families

**Scenario**: Custody changes, Emma moves from Family A to Family B

**Implementation**:
```python
# Option 1: Keep both memberships, mark one inactive
FamilyMembership.objects.filter(
    user_id=emma.id, 
    family_id=family_a.id
).update(is_active=False, ended_at=now())

# Option 2: Transfer kid permanently (rare)
# Archive all Family A data for Emma
# Create new accounts in Family B
# Kid loses access to Family A but data preserved
```

#### UI/UX Considerations

**Family Switcher Component**
```jsx
// Always visible in header for kids with multiple families
<FamilySwitcher>
  <CurrentFamily icon="🏠">Mom's House</CurrentFamily>
  <MenuItem onClick={switchTo(familyB)}>
    Switch to Dad's House 🏡
  </MenuItem>
</FamilySwitcher>
```

**Visual Indicators**
- Current family context always visible in header
- Color coding: Each family gets assigned color
- Confirmation dialog when switching: "Switch to Dad's House? You won't see Mom's House data until you switch back."

#### Edge Cases

1. **Kid claims chore in Family A, then switches to Family B**
   - Chore belongs to Family A
   - Family B context cannot see/modify that chore
   - Kid must switch back to Family A to complete it

2. **Parent belongs to multiple families (remarriage)**
   - Same context switching mechanism
   - Parent can manage multiple separate households

3. **Notification routing**
   - Chore reminder sent to kid
   - Notification includes family context: "Mom's House: Clean room due today"

4. **Reporting across families**
   - Kid cannot see combined report across families (privacy)
   - Each family's report is isolated

---

## Database Schema

See `docs/DATABASE_SCHEMA.md`

---

## API Specifications

See `docs/API_SPECIFICATIONS.md`

### Report Endpoints

#### GET /reports/weekly-summary
Get weekly account summary (email report data)

#### GET /reports/transaction-history
Get transaction report with filtering

#### GET /reports/chore-completion-rate
Get chore completion statistics

---

## Chore Scheduling System
**[Backend]**

### Problem Statement
FamZoo's chore scheduler had critical limitations:
1. **No custom recurrence**: Couldn't do M-W-F, bi-weekly, or custom intervals
2. **Absolute timing only**: If chore was 1.5 weeks overdue, next instance was in 0.5 weeks
3. **Backlog accumulation**: Overdue chores piled up (vacuum stairs 3x)

### Solution: Advanced Scheduling Engine

#### Recurrence Rule Format
Based on iCalendar RFC 5545 RRULE with extensions:

```json
{
  "frequency": "WEEKLY",  // DAILY, WEEKLY, MONTHLY, YEARLY
  "interval": 2,          // Every 2 weeks (bi-weekly)
  "by_weekday": ["MO", "WE", "FR"],  // M-W-F pattern
  "by_monthday": [1, 15],            // 1st and 15th of month
  "by_month": [6, 7, 8],             // Summer months only
  "timing_mode": "relative",          // "absolute" or "relative"
  "start_date": "2026-02-01",
  "end_date": null,                   // NULL = infinite
  "count": null                       // Or limit to X occurrences
}
```

#### Supported Patterns

**Daily**
```json
// Every day
{"frequency": "DAILY", "interval": 1}

// Weekdays only
{"frequency": "DAILY", "interval": 1, "by_weekday": ["MO", "TU", "WE", "TH", "FR"]}

// Every 3 days
{"frequency": "DAILY", "interval": 3}
```

**Weekly**
```json
// Every Monday
{"frequency": "WEEKLY", "interval": 1, "by_weekday": ["MO"]}

// M-W-F
{"frequency": "WEEKLY", "interval": 1, "by_weekday": ["MO", "WE", "FR"]}

// Bi-weekly on Sunday
{"frequency": "WEEKLY", "interval": 2, "by_weekday": ["SU"]}

// Every 3 weeks on Tuesday
{"frequency": "WEEKLY", "interval": 3, "by_weekday": ["TU"]}
```

**Monthly**
```json
// 1st of every month
{"frequency": "MONTHLY", "interval": 1, "by_monthday": [1]}

// 1st and 15th (semimonthly)
{"frequency": "MONTHLY", "interval": 1, "by_monthday": [1, 15]}

// Last day of month
{"frequency": "MONTHLY", "interval": 1, "by_monthday": [-1]}

// First Monday of month
{"frequency": "MONTHLY", "interval": 1, "by_weekday": ["1MO"]}
```

#### Timing Modes

**Absolute Mode**
- Chore scheduled at fixed intervals from start_date
- Example: "Vacuum stairs every 2 weeks starting Feb 1"
- Timeline: Feb 1, Feb 15, Mar 1, Mar 15...
- If completed late (Feb 20), next due date stays Mar 1
- **Use case**: Time-sensitive chores, routines tied to calendar

**Relative Mode**
- Next instance generated from COMPLETION date
- Example: "Vacuum stairs 2 weeks after last completion"
- Timeline: Feb 1 (assigned), Feb 20 (completed), Mar 6 (next due)
- **Use case**: Maintenance tasks, flexible chores

#### Instance Generation Algorithm

```python
from datetime import datetime, timedelta
from dateutil.rrule import rrule, DAILY, WEEKLY, MONTHLY

def generate_chore_instances(chore, limit=10):
    """
    Generate upcoming chore assignment instances.
    Called by background job hourly.
    """
    rule = chore.recurrence_rule
    
    if rule["timing_mode"] == "absolute":
        # Generate from chore start_date
        base_date = rule["start_date"]
        instances = rrule(
            freq=FREQUENCY_MAP[rule["frequency"]],
            interval=rule["interval"],
            byweekday=parse_weekdays(rule.get("by_weekday")),
            bymonthday=rule.get("by_monthday"),
            dtstart=base_date,
            count=limit
        )
        
        for due_date in instances:
            # Check if instance already exists
            if not assignment_exists(chore.id, due_date):
                create_assignment(
                    chore_id=chore.id,
                    due_date=due_date,
                    status="PENDING" if chore.assignment_type == "ASSIGNED" else "AVAILABLE"
                )
    
    elif rule["timing_mode"] == "relative":
        # Generate from last COMPLETION date
        last_completed = get_last_completed_assignment(chore.id)
        
        if last_completed:
            base_date = last_completed.completed_at.date()
        else:
            base_date = rule["start_date"]
        
        # Calculate next due date
        if rule["frequency"] == "DAILY":
            next_due = base_date + timedelta(days=rule["interval"])
        elif rule["frequency"] == "WEEKLY":
            next_due = base_date + timedelta(weeks=rule["interval"])
        elif rule["frequency"] == "MONTHLY":
            next_due = add_months(base_date, rule["interval"])
        
        # Create only ONE upcoming instance (no backlog)
        if not assignment_exists(chore.id, next_due):
            create_assignment(chore_id=chore.id, due_date=next_due)
```

#### Overdue Handling

**Single Instance Policy**
- Only ONE active instance exists at a time (no backlog)
- Overdue chore stays as single instance with cumulative overdue time

**Overdue Tracking**
```python
def update_overdue_time(assignment):
    """
    Called daily by background job.
    Tracks total overdue days for display.
    """
    if assignment.status in ["PENDING", "CLAIMED"]:
        if assignment.due_date < date.today():
            days_overdue = (date.today() - assignment.due_date).days
            assignment.total_overdue_days = days_overdue
            assignment.save()
```

**Display Logic**
```jsx
// Kid sees chore listed ONCE with overdue indicator
<ChoreCard chore={assignment}>
  {assignment.total_overdue_days > 0 && (
    <OverdueBadge>
      {assignment.total_overdue_days} days overdue 🔴
    </OverdueBadge>
  )}
</ChoreCard>
```

#### Edge Cases

**Leap Year**
- Monthly chore on 31st → February runs on 28th/29th (last day of month)

**Daylight Saving Time**
- Deadline time "8:00 AM" stays at 8:00 AM local time (shifts with DST)

**Timezone Changes**
- Family moves timezones → chore deadlines shift to new timezone
- Handled by storing deadline_time without timezone, applying family timezone

**Chore Expiration**
- Optional: Assignment expires X days after due date
- Status changes to EXPIRED, no reward possible
- Relative mode: Next instance still generates from last completion

#### Background Job Schedule

```python
# APScheduler configuration
scheduler.add_job(
    func=generate_upcoming_chore_instances,
    trigger="cron",
    hour="*/1",  # Every hour
    id="chore_instance_generator"
)

scheduler.add_job(
    func=update_all_overdue_chores,
    trigger="cron",
    hour=0,  # Midnight daily
    id="overdue_tracker"
)

scheduler.add_job(
    func=send_chore_reminders,
    trigger="cron",
    hour=8,  # 8 AM daily
    id="chore_reminder"
)
```

---

## Age-Adaptive UI System
**[Frontend]**

### Problem Statement
Kids of different ages have vastly different:
- Reading comprehension levels
- Attention spans
- Financial literacy
- Feature needs

A 6-year-old needs simple icons and gamification. A 16-year-old needs adult-level financial tools.

### Solution: Age-Bracket Component System

#### Age Brackets

| Bracket | Ages | Characteristics | UI Approach |
|---------|------|-----------------|-------------|
| **Young** | 5-9 | Early readers, concrete thinking, needs structure | Large icons, minimal text, gamified, limited features |
| **Tween** | 10-13 | Developing independence, abstract thinking emerging | Balanced text/icons, more features, educational |
| **Teen** | 14-17 | Adult-like cognition, financial planning | Full features, adult interface, investment focus |

#### Age Calculation

```python
from datetime import date

def calculate_age_bracket(birthdate):
    """
    Calculate current age and determine UI bracket.
    """
    if not birthdate:
        return "teen"  # Default for no birthdate
    
    today = date.today()
    age = today.year - birthdate.year
    
    # Adjust for birthday not yet occurred this year
    if (today.month, today.day) < (birthdate.month, birthdate.day):
        age -= 1
    
    if age < 10:
        return "young"
    elif age < 14:
        return "tween"
    else:
        return "teen"

def get_effective_age_bracket(user, family_membership):
    """
    Get age bracket with parent override support.
    """
    if family_membership.age_bracket_override:
        return family_membership.age_bracket_override
    
    return calculate_age_bracket(user.birthdate)
```

#### UI Differences by Bracket

**Navigation**

Young (5-9):
```jsx
<BottomNav>
  <NavItem icon="🎯" label="">Chores</NavItem>
  <NavItem icon="💰" label="">Money</NavItem>
  <NavItem icon="⭐" label="">Goals</NavItem>
</BottomNav>
```

Tween (10-13):
```jsx
<BottomNav>
  <NavItem icon={<CurrencyDollarIcon />} label="My Money" />
  <NavItem icon={<ClipboardListIcon />} label="Chores" />
  <NavItem icon={<ChartBarIcon />} label="Goals" />
  <NavItem icon={<CogIcon />} label="Settings" />
</BottomNav>
```

Teen (14-17):
```jsx
<SideNav>
  <NavSection title="Financial">
    <NavItem>Accounts</NavItem>
    <NavItem>Transactions</NavItem>
    <NavItem>Budget</NavItem>
  </NavSection>
  <NavSection title="Tasks">
    <NavItem>Chores</NavItem>
    <NavItem>Goals</NavItem>
  </NavSection>
</SideNav>
```

**Dashboard**

Young:
```jsx
<Dashboard>
  {/* Big cards with emoji */}
  <BalanceCard>
    <BigEmoji>💰</BigEmoji>
    <BigNumber>${balance}</BigNumber>
    <SimpleLabel>You have</SimpleLabel>
  </BalanceCard>
  
  <ChoresCard>
    <BigEmoji>🎯</BigEmoji>
    <BigNumber>{pendingChores}</BigNumber>
    <SimpleLabel>Chores to do</SimpleLabel>
  </ChoresCard>
  
  {/* Visual progress bars */}
  <GoalCard>
    <BigEmoji>🎮</BigEmoji>
    <ProgressBar value={75} max={100} />
    <SimpleLabel>$45 of $60 saved!</SimpleLabel>
  </GoalCard>
</Dashboard>
```

Tween:
```jsx
<Dashboard>
  <BalanceCard>
    <Label>Total Balance</Label>
    <Amount>${totalBalance}</Amount>
    <AccountBreakdown>
      <div>Spending: ${spending}</div>
      <div>Saving: ${saving}</div>
    </AccountBreakdown>
  </BalanceCard>
  
  <ChoreList>
    <Header>Today's Chores ({pending})</Header>
    {chores.map(chore => (
      <ChoreItem key={chore.id} chore={chore} />
    ))}
  </ChoreList>
  
  <RecentActivity>
    <Header>Recent</Header>
    {transactions.slice(0, 5).map(tx => (
      <TransactionItem key={tx.id} transaction={tx} />
    ))}
  </RecentActivity>
</Dashboard>
```

Teen:
```jsx
<Dashboard>
  {/* Multi-column layout */}
  <Grid cols={3}>
    <AccountSummary accounts={accounts} />
    <IncomeExpenseChart data={chartData} />
    <UpcomingPayments allowances={allowances} />
  </Grid>
  
  <Grid cols={2}>
    <ChoreSchedule />
    <GoalProgress goals={goals} />
  </Grid>
  
  <FullWidthCard>
    <TransactionTable 
      transactions={transactions} 
      filters={filters}
      exportable
    />
  </FullWidthCard>
</Dashboard>
```

**Chore Completion Flow**

Young:
```jsx
<ChoreCard chore={chore}>
  <BigIcon>🧹</BigIcon>
  <Title>Clean your room</Title>
  
  {/* Simple button */}
  <BigButton onClick={complete}>
    ✓ I did it!
  </BigButton>
</ChoreCard>

{/* Celebration on completion */}
<CompletionModal>
  <BigEmoji>🎉</BigEmoji>
  <Title>Great job!</Title>
  <Reward>You earned $5! 💰</Reward>
  <Button>Yay!</Button>
</CompletionModal>
```

Tween:
```jsx
<ChoreCard chore={chore}>
  <Header>
    <Title>Clean your room</Title>
    <Reward>$5</Reward>
  </Header>
  
  <Description>
    Make bed, pick up toys, vacuum
  </Description>
  
  <Footer>
    <DueDate>Due: Today at 8 PM</DueDate>
    <Button onClick={complete}>Mark Complete</Button>
  </Footer>
</ChoreCard>

{/* Optional photo upload */}
<CompletionModal>
  <Title>Mark Complete</Title>
  <PhotoUpload optional />
  <Note>Waiting for parent approval</Note>
  <Button>Submit</Button>
</CompletionModal>
```

Teen:
```jsx
<ChoreCard chore={chore}>
  <Header>
    <Title>{chore.title}</Title>
    <StatusBadge status={chore.status} />
  </Header>
  
  <Details>
    <Detail label="Reward">${chore.reward}</Detail>
    <Detail label="Due">{formatDate(chore.dueDate)}</Detail>
    <Detail label="Estimated time">{chore.duration}min</Detail>
  </Details>
  
  <Actions>
    <Button onClick={complete}>Complete</Button>
    <Button variant="secondary" onClick={viewHistory}>
      History
    </Button>
  </Actions>
</ChoreCard>
```

**Typography & Sizing**

```css
/* Young (5-9) */
.young {
  --font-size-base: 18px;
  --font-size-large: 32px;
  --button-height: 60px;
  --touch-target-min: 60px;
  --spacing-base: 24px;
}

/* Tween (10-13) */
.tween {
  --font-size-base: 16px;
  --font-size-large: 24px;
  --button-height: 48px;
  --touch-target-min: 48px;
  --spacing-base: 16px;
}

/* Teen (14-17) */
.teen {
  --font-size-base: 14px;
  --font-size-large: 20px;
  --button-height: 40px;
  --touch-target-min: 44px;
  --spacing-base: 12px;
}
```

#### Feature Gating

```python
# Feature access matrix
FEATURE_ACCESS = {
    "young": {
        "view_chores": True,
        "complete_chores": True,
        "view_balance": True,
        "view_goals": True,
        "request_money": False,
        "transfer_between_accounts": False,
        "view_transaction_history": False,
        "manage_budget": False,
    },
    "tween": {
        "view_chores": True,
        "complete_chores": True,
        "view_balance": True,
        "view_goals": True,
        "request_money": True,
        "transfer_between_accounts": True,  # With limits
        "view_transaction_history": True,
        "manage_budget": False,
        "view_investment_accounts": False,
    },
    "teen": {
        # All features enabled
        "view_chores": True,
        "complete_chores": True,
        "view_balance": True,
        "view_goals": True,
        "request_money": True,
        "transfer_between_accounts": True,
        "view_transaction_history": True,
        "manage_budget": True,
        "view_investment_accounts": True,
        "set_own_goals": True,
        "view_analytics": True,
    }
}
```

#### Component Implementation

```jsx
// Age-adaptive component wrapper
import { useUser } from '@/contexts/UserContext';

export function AdaptiveComponent({ children, feature }) {
  const { user, ageBracket } = useUser();
  
  // Check feature access
  if (!FEATURE_ACCESS[ageBracket][feature]) {
    return <FeatureLockedMessage ageBracket={ageBracket} />;
  }
  
  // Render age-appropriate version
  return (
    <div className={`ui-${ageBracket}`}>
      {children}
    </div>
  );
}

// Usage
<AdaptiveComponent feature="view_transaction_history">
  {ageBracket === 'young' && <SimpleTransactionList />}
  {ageBracket === 'tween' && <TransactionList />}
  {ageBracket === 'teen' && <TransactionTable />}
</AdaptiveComponent>
```

#### Age-Up Celebration

When kid's birthday triggers bracket change:

```jsx
<AgeUpModal>
  <Confetti />
  <Emoji>🎂</Emoji>
  <Title>Happy Birthday, {name}!</Title>
  <Subtitle>You've leveled up! 🎉</Subtitle>
  
  <FeatureShowcase>
    <Title>Check out your new features:</Title>
    <FeatureList>
      {newFeatures.map(feature => (
        <Feature key={feature.id}>
          <Icon>{feature.icon}</Icon>
          <Name>{feature.name}</Name>
          <Description>{feature.description}</Description>
        </Feature>
      ))}
    </FeatureList>
  </FeatureShowcase>
  
  <Button>Explore my new dashboard!</Button>
</AgeUpModal>
```

**Background Job**
```python
# Check for birthdays daily
scheduler.add_job(
    func=check_age_bracket_transitions,
    trigger="cron",
    hour=0,  # Midnight
    id="age_bracket_checker"
)

def check_age_bracket_transitions():
    """
    Check if any kids aged into new bracket today.
    Send notification if bracket changed.
    """
    today = date.today()
    users = User.query.filter(User.role == "CHILD").all()
    
    for user in users:
        if is_birthday_today(user.birthdate):
            old_bracket = get_age_bracket(user.birthdate - timedelta(days=1))
            new_bracket = get_age_bracket(user.birthdate)
            
            if old_bracket != new_bracket:
                send_age_up_notification(user, new_bracket)
```

#### Preset Profiles by Age

When adding kid to family, parent selects preset:

```python
PARENTING_PRESETS = {
    "young": {
        "strict": {
            "allow_negative_balances": False,
            "transfer_limit": None,  # No transfers
            "require_chore_approval": True,
            "auto_approve_hours": None,
            "notification_preferences": {
                "chore_reminders": True,
                "parent_gets_completion_notifications": True
            }
        },
        "balanced": {
            "allow_negative_balances": False,
            "transfer_limit": 10.00,
            "require_chore_approval": True,
            "auto_approve_hours": 24,
            "notification_preferences": {
                "chore_reminders": True,
                "parent_gets_completion_notifications": False
            }
        },
        "flexible": {
            "allow_negative_balances": True,
            "negative_limit": 5.00,
            "transfer_limit": 20.00,
            "require_chore_approval": False,
            "notification_preferences": {
                "chore_reminders": False,
                "parent_gets_completion_notifications": False
            }
        }
    },
    "tween": {
        "strict": {
            "allow_negative_balances": False,
            "transfer_limit": 25.00,
            "require_chore_approval": True,
            # ... more autonomy than young
        },
        "balanced": {
            "allow_negative_balances": True,
            "negative_limit": 20.00,
            "transfer_limit": 50.00,
            "require_chore_approval": True,
            "auto_approve_hours": 12,
        },
        "flexible": {
            "allow_negative_balances": True,
            "negative_limit": 50.00,
            "transfer_limit": None,  # No limit
            "require_chore_approval": False,
        }
    },
    "teen": {
        "strict": {
            "allow_negative_balances": True,
            "negative_limit": 50.00,
            "transfer_limit": 100.00,
            "require_chore_approval": False,
        },
        "balanced": {
            "allow_negative_balances": True,
            "negative_limit": 100.00,
            "transfer_limit": None,
            "require_chore_approval": False,
        },
        "flexible": {
            # Almost full autonomy
            "allow_negative_balances": True,
            "negative_limit": 200.00,
            "transfer_limit": None,
            "require_chore_approval": False,
        }
    }
}
```

---

## Development Phases

### Phase 1: Foundation + Social Auth (2-3 weeks)

**Goals:**
- Local development environment
- Basic authentication (JWT + social)
- Multi-household data models
- Manual transaction system

**Deliverables:**
1. **Infrastructure**
   - Docker setup (frontend/backend containers)
   - Native PostgreSQL installation guide
   - docker-compose.yml with hot reload
   - Environment configuration (.env files)

2. **Backend (FastAPI)**
   - Project structure with app/ directory
   - SQLAlchemy models (User, Family, FamilyMembership, Account, Transaction)
   - Alembic migrations setup
   - JWT authentication with refresh tokens
   - OAuth 2.0 integration (Google, Apple, Facebook)
   - Multi-tenant middleware (family context injection)
   - Account CRUD endpoints
   - Manual transaction endpoints
   - OpenAPI documentation

3. **Frontend (React + Vite)**
   - Project scaffolding with Vite
   - React Router setup
   - Authentication flows (login, register, social auth)
   - Family context switching UI
   - Account list/detail views
   - Manual transaction forms
   - Basic responsive layout

4. **Database**
   - PostgreSQL 15+ installation
   - Schema creation scripts
   - Row-Level Security policies
   - Test data seeders (4 test families with multi-household scenario)

**Testing:**
- pytest for backend (auth, accounts, transactions)
- Vitest for frontend components
- 80% coverage target for auth flows
- 100% coverage for money transactions

**Documentation:**
- README with setup instructions
- API documentation (auto-generated)
- PostgreSQL installation guide (macOS/Homebrew)

---

### Phase 2: Chores MVP (2-3 weeks)

**Goals:**
- Advanced chore scheduling engine
- Basic chore assignment workflow
- Parent approval system

**Deliverables:**
1. **Backend**
   - Chore and ChoreAssignment models
   - Recurrence rule parser (RFC 5545 + extensions)
   - Instance generation algorithm (absolute/relative modes)
   - APScheduler setup with PostgreSQL backend
   - Chore CRUD endpoints
   - Assignment endpoints (claim, complete, approve, reject)
   - Background job: generate upcoming instances hourly

2. **Frontend**
   - Chore creation form with advanced scheduling
     - Custom recurrence UI (M-W-F, bi-weekly, etc.)
     - Absolute vs Relative toggle
     - Preview of next 5 occurrences
   - Kid dashboard: Today's chores (basic responsive)
   - Parent dashboard: Pending approvals
   - Chore detail view
   - Complete/approve/reject actions

3. **Features**
   - Assigned chores (not first-dibs yet)
   - Daily, weekly, bi-weekly, monthly recurrence
   - Custom weekday patterns (M-W-F)
   - Relative timing mode (reset from completion)
   - Single overdue instance (no backlog)
   - Reward crediting on approval

**Testing:**
- Recurrence algorithm tests (edge cases: leap year, DST, month-end)
- Completion workflow tests
- Approval workflow tests
- Background job tests (mock time progression)

**Documentation:**
- Chore scheduling algorithm docs
- Recurrence pattern examples

---

### Phase 3: Enhanced Chores (2 weeks)

**Goals:**
- First-dibs chore system
- Chore templates library
- Improved mobile UX

**Deliverables:**
1. **Backend**
   - First-dibs claiming logic with race condition handling
   - Chore templates library (predefined chores)
   - Bulk chore creation endpoint
   - Notification system (basic push notifications)

2. **Frontend**
   - First-dibs UI (available chores, claim button)
   - Chore templates browser
   - Bulk chore creation wizard
   - Mobile swipe actions (swipe to approve/reject)
   - Pull-to-refresh
   - Overdue badges with cumulative days

3. **Features**
   - First-dibs assignment type
   - Chore claiming with timeout
   - Template library (10-20 common chores)
   - Clone existing chores
   - Batch approval
   - Push notifications (new chore, completion, approval)

**Testing:**
- Race condition tests (multiple kids claiming same chore)
- Notification delivery tests
- Mobile gesture tests

---

### Phase 4: Allowances & Automation (2 weeks)

**Goals:**
- Automated allowance payments
- Interest calculations
- Transaction reporting

**Deliverables:**
1. **Backend**
   - Allowance and Split models
   - Allowance payment processor (scheduled jobs)
   - Interest calculation engine
   - Automated debit system
   - CSV export endpoint

2. **Frontend**
   - Allowance creation form
     - Amount or formula (age-based)
     - Frequency selection
     - Split configuration
   - Split builder UI (drag percentages)
   - Interest calculator
   - Transaction export UI
   - Weekly report preview

3. **Features**
   - Weekly, bi-weekly, monthly, semi-monthly allowances
   - Age-based formulas (age * 2)
   - Split deposits (60/20/20)
   - Parent-paid interest (compound)
   - Interest caps
   - Automated recurring debits
   - CSV transaction export

**Testing:**
- Allowance payment accuracy tests
- Interest calculation tests (compound, caps)
- Split percentage validation tests
- Timezone handling tests

**Documentation:**
- Allowance configuration guide
- Interest calculation examples

---

### Phase 5: Infrastructure & Deployment (2-3 weeks)

**Goals:**
- AWS infrastructure setup
- CI/CD pipeline
- Production deployment
- Cost monitoring

**Deliverables:**
1. **Infrastructure as Code (Terraform)**
   - AWS Lightsail containers (POC)
   - PostgreSQL (Lightsail managed DB)
   - S3 bucket for static assets
   - CloudFront distribution
   - SES for email
   - Route 53 DNS
   - IAM roles and policies
   - Cost budgets and alarms

2. **CI/CD (GitHub Actions)**
   - Automated testing on PR
   - Docker image builds
   - Deployment to dev environment
   - Deployment to prod (manual approval)
   - Database migrations automation

3. **Environments**
   - Dev: Lightsail Micro container + Standard DB ($25/mo)
   - Prod: Lightsail Small container + HA DB ($50/mo)

4. **Monitoring**
   - CloudWatch dashboards
   - Error alerting
   - Cost monitoring
   - Performance metrics

5. **Security**
   - HTTPS/TLS certificates
   - Secrets management (AWS Secrets Manager)
   - Database encryption at rest
   - Backup automation

**Testing:**
- Deployment smoke tests
- Infrastructure validation tests

**Documentation:**
- Deployment runbook
- AWS cost optimization guide
- Disaster recovery procedures

---

### Phase 6: Goals & Advanced Features (2 weeks)

**Goals:**
- Savings goals tracking
- Budget worksheets
- Loan tracking
- Real-time updates

**Deliverables:**
1. **Backend**
   - SavingsGoal, Budget, Loan models
   - Goal progress calculations
   - Budget tracking
   - Loan amortization
   - Weekly email report generator (SES)
   - Transfer request workflow
   - WebSocket support (Socket.io)

2. **Frontend**
   - Savings goal creation/tracking UI
   - Progress visualization (progress bars, charts)
   - Budget worksheet
   - Loan calculator and tracker
   - Transfer request UI
   - Real-time notification badges (WebSocket)

3. **Features**
   - Visual goal progress
   - Target date projections
   - Budget categories
   - Income vs expense tracking
   - Loan repayment schedules
   - Weekly email summary
   - Transfer request approval
   - Real-time chore completion updates

**Testing:**
- Goal progress calculation tests
- Budget tracking tests
- Email delivery tests
- WebSocket connection tests

---

### Phase 7: Age-Adaptive Mobile UI (2-3 weeks)

**Goals:**
- Age-bracket UI system
- Mobile PWA optimization
- Feature gating

**Deliverables:**
1. **Backend**
   - Age bracket calculation logic
   - Feature access matrix
   - Preset profile system
   - Age-up notification system

2. **Frontend**
   - Age-adaptive component system
   - Young (5-9) UI components
   - Tween (10-13) UI components
   - Teen (14-17) UI components
   - Feature gating logic
   - Parent override for age bracket
   - Age-up celebration modal
   - PWA manifest and service worker
   - Offline support

3. **Features**
   - Automatic age bracket detection
   - Age-appropriate dashboards
   - Simplified young kid UI (big icons, minimal text)
   - Full-featured teen UI
   - Feature gating (transfers, history, budgets)
   - Birthday celebrations
   - PWA installation
   - Offline chore viewing

**Testing:**
- Age bracket transition tests
- Feature access tests
- PWA functionality tests
- Offline mode tests

**Documentation:**
- Age-adaptive UI design guide
- Preset profile recommendations

---

### Phase 8: GraphQL Migration (2 weeks) [OPTIONAL]

**Goals:**
- GraphQL API layer
- Frontend migration to GraphQL
- Maintain REST backwards compatibility

**Deliverables:**
1. **Backend**
   - Strawberry GraphQL setup
   - Schema definition
   - Resolvers for all entities
   - Subscriptions (real-time)
   - REST endpoints marked deprecated

2. **Frontend**
   - Apollo Client setup
   - GraphQL queries/mutations
   - Subscription handling
   - Optimistic updates

3. **Features**
   - GraphQL playground
   - Efficient nested data fetching
   - Real-time subscriptions
   - Reduced over-fetching

**Note:** This phase is optional. Can proceed with REST API if GraphQL adds unnecessary complexity.

---

### Phase 9: Polish & Scale (2 weeks)

**Goals:**
- Production readiness
- Performance optimization
- Security hardening
- Account linking

**Deliverables:**
1. **Backend**
   - Performance optimization (query optimization, caching)
   - Rate limiting (Redis-backed)
   - Account linking (link social auth to existing accounts)
   - Advanced audit logging
   - Soft delete cleanup job

2. **Frontend**
   - Error boundaries
   - Loading states
   - Empty states with helpful hints
   - Onboarding flow
   - Keyboard shortcuts
   - Accessibility improvements (WCAG 2.1 AA)

3. **Features**
   - Social auth account linking
   - Advanced filtering/search
   - Data export (full account data)
   - 30-day soft delete with recovery
   - Performance dashboard
   - Parent coaching tips

4. **Testing**
   - Load testing (100 concurrent users)
   - Security testing (OWASP Top 10)
   - E2E tests (Playwright)
   - Accessibility audit

5. **Documentation**
   - User guides
   - Video tutorials
   - FAQ
   - Privacy policy
   - Terms of service

**Launch Readiness Checklist:**
- [ ] Security audit passed
- [ ] Load testing passed
- [ ] Backup/restore tested
- [ ] Monitoring/alerting working
- [ ] Documentation complete
- [ ] Support system ready

---

### Test Data

Four test families are seeded:

**Family 1: Single Household (Young Kids)**
- Parent: parent1@test.com / password123
- Kids: Emma (6), Liam (8)
- Accounts: Spending, Saving, Giving for each kid
- Preset: Balanced
- Sample chores, allowances, transactions

**Family 2: Single Household (Tweens)**
- Parent: parent2@test.com / password123
- Kids: Olivia (11), Noah (13)
- More autonomy, transfer capabilities
- Preset: Flexible

**Family 3: Single Household (Teens)**
- Parent: parent3@test.com / password123
- Kids: Ava (15), Ethan (17)
- Full features, investment accounts
- Preset: Flexible

**Family 4: Multi-Household (Mixed Ages)**
- Mom's household: mom@test.com / password123
  - Kids: Sophia (7), Mason (10), Isabella (14)
- Dad's household: dad@test.com / password123
  - Kids: Sophia (7), Mason (10), Isabella (14) [SAME kids, different family]
- Demonstrates multi-household feature

**Shared Kid Credentials:**
- sophia@test.com / password123 (belongs to both Mom & Dad families)
- mason@test.com / password123 (belongs to both families)
- isabella@test.com / password123 (belongs to both families)

### Running Tests

```bash
# Backend tests
cd backend
pytest -v
pytest --cov=app --cov-report=html

# Frontend tests
cd frontend
npm test
npm run test:coverage

# E2E tests (Phase 9)
npm run test:e2e
```

### Development Workflow

1. **Create feature branch**
   ```bash
   git checkout -b feature/chore-scheduling
   ```

2. **Make changes with hot reload**
   - Backend: Edit Python files → uvicorn auto-reloads
   - Frontend: Edit React files → Vite HMR updates instantly

3. **Run tests**
   ```bash
   pytest  # Backend
   npm test  # Frontend
   ```

4. **Commit with conventional commits**
   ```bash
   git commit -m "feat: add bi-weekly chore recurrence"
   git commit -m "fix: correct overdue calculation"
   git commit -m "docs: update API documentation"
   ```

5. **Push and create PR**
   ```bash
   git push origin feature/chore-scheduling
   ```

### Troubleshooting

**PostgreSQL connection error**
```bash
# Check PostgreSQL is running
brew services list

# Restart if needed
brew services restart postgresql@15

# Check connection
psql -U picklesapp -d picklesapp_dev
```

**Port already in use**
```bash
# Find process using port 8000
lsof -i :8000

# Kill process
kill -9 <PID>
```

**Docker containers not starting**
```bash
# Clean rebuild
docker-compose down -v
docker-compose up -d --build
```

---

## AWS Deployment Architecture

### POC Phase ($25-30/mo)

**Architecture:**
```
Internet
   ↓
Route 53 / Lightsail DNS
   ↓
CloudFront (CDN)
   ↓
┌─────────────────────────────────┐
│   Lightsail Container (Micro)   │
│   - Backend + Frontend serve    │
│   - 1GB RAM, 0.25 vCPU          │
│   - $10/mo                      │
└─────────────────────────────────┘
   ↓
┌─────────────────────────────────┐
│ Lightsail Managed PostgreSQL    │
│ - Standard (40GB)               │
│ - 1GB RAM, 1 vCPU               │
│ - $15/mo                        │
└─────────────────────────────────┘
```

**Services:**
- **Lightsail Container**: $10/mo (Micro) - Backend API + serve frontend static
- **Lightsail PostgreSQL**: $15/mo (Standard) - Database with backups
- **S3**: $0-1/mo - Static assets, future photo storage
- **CloudFront**: Free tier - CDN for static assets
- **SES**: $0-1/mo - Email notifications
- **Route 53** OR **Lightsail DNS**: $0.50/mo or free

**Total: ~$25-30/mo**

### Beta Phase ($50-75/mo)

**Improvements:**
- Lightsail Container **Small** ($20/mo) - 2GB RAM
- Lightsail PostgreSQL **High Availability** ($30/mo) - Multi-AZ
- CloudWatch enhanced monitoring ($5-10/mo)

### Production Phase ($150-300/mo)

**Migration to:**
- **ECS Fargate**: Auto-scaling containers
- **RDS PostgreSQL**: db.t3.small with Multi-AZ
- **Application Load Balancer**: $16/mo
- **ElastiCache**: Redis for sessions/rate limiting ($15/mo)
- **AWS Cognito** (optional): User pool management
- **CloudWatch**: Enhanced logs and dashboards
- **AWS WAF**: Web application firewall ($10/mo)

### Terraform Structure

```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── modules/
│   ├── lightsail/
│   │   ├── container.tf
│   │   ├── database.tf
│   │   └── dns.tf
│   ├── s3/
│   ├── cloudfront/
│   └── monitoring/
└── README.md
```

### Deployment Process

**Dev Environment:**
```bash
cd infrastructure/environments/dev
terraform init
terraform plan
terraform apply

# Deploy application
aws lightsail push-container-image \
  --service-name picklesapp-dev \
  --label backend \
  --image backend:latest

aws lightsail create-container-service-deployment \
  --service-name picklesapp-dev \
  --containers file://containers.json \
  --public-endpoint file://public-endpoint.json
```

**Production Deployment:**
```bash
# Via GitHub Actions (manual approval required)
git tag v1.0.0
git push origin v1.0.0

# Triggers workflow:
# 1. Run tests
# 2. Build Docker images
# 3. Push to registry
# 4. Wait for manual approval
# 5. Deploy to production
# 6. Run smoke tests
# 7. Notify team
```

---

## Security Framework

### Authentication & Authorization

**Password Security:**
- Bcrypt hashing (cost factor 12)
- Minimum 8 characters (no special char requirement)
- No password in JWT payload
- Refresh tokens stored hashed in DB

**JWT Tokens:**
- Access token: 1 hour expiry
- Refresh token: 30 days expiry
- HMAC SHA256 signing
- Payload includes: user_id, family_id (current context), role

**Social Auth:**
- OAuth 2.0 authorization code flow
- State parameter for CSRF protection
- Token storage encrypted at rest
- Regular token refresh

**Session Management:**
- Mobile: Stay logged in (refresh token)
- Web: 30-day session with sliding expiration
- Logout: Blacklist refresh token

### Data Protection

**Encryption:**
- TLS 1.3 in transit (HTTPS)
- Database encryption at rest (AWS)
- Sensitive fields encrypted in DB (social tokens)

**Multi-Tenancy Isolation:**
- Application middleware (primary)
- PostgreSQL RLS (safety net)
- All queries scoped by family_id
- Audit logging of cross-family access attempts

**PII Protection:**
- Minimal data collection
- No SSN or credit card storage
- Birthdates hashed in logs
- Email addresses masked in non-prod environments

### Input Validation

**API Layer:**
- Pydantic models for request validation
- SQLAlchemy ORM (SQL injection prevention)
- Parameterized queries only
- Input sanitization

**Validation Rules:**
- Email format validation
- Amount limits (max $10,000 per transaction)
- Date range validation
- Recurrence rule validation

### Rate Limiting

**API Endpoints:**
- 100 requests/minute per user (general)
- 5 login attempts per 15 minutes per IP
- 10 password reset requests per hour per email

**Implementation:**
- Redis-backed (production)
- In-memory (development)

### Audit Logging

**Logged Events:**
- Authentication (login, logout, failed attempts)
- Money operations (transactions, transfers)
- Family changes (add/remove member)
- Chore approvals/rejections
- Settings changes

**Log Contents:**
- User ID, family ID
- Action type, entity type, entity ID
- Before/after values (for updates)
- IP address, user agent
- Timestamp (UTC)

**Retention:**
- 90 days in CloudWatch
- 1 year in S3 (compressed)

### Compliance

**GDPR (if EU users):**
- Data export functionality
- Right to deletion (30-day soft delete)
- Privacy policy
- Cookie consent

**COPPA (children under 13):**
- Parental consent for account creation
- Minimal data collection from children
- No advertising to children
- Parental control over child data

**CCPA (California):**
- Data sale opt-out (N/A - no data selling)
- Data disclosure
- Deletion rights

### Security Checklist

**Phase 1:**
- [x] Password hashing (bcrypt)
- [x] JWT implementation
- [x] HTTPS only
- [x] SQL injection prevention
- [x] XSS prevention
- [x] CORS configuration
- [ ] Rate limiting (Phase 5)

**Phase 5:**
- [ ] Secrets management (AWS Secrets Manager)
- [ ] CloudWatch alerting
- [ ] Backup encryption
- [ ] DDoS protection (AWS Shield)

**Phase 9:**
- [ ] Security audit (OWASP Top 10)
- [ ] Penetration testing
- [ ] Dependency vulnerability scanning
- [ ] WAF rules (SQL injection, XSS)

---

## Testing Strategy

### Test Coverage Targets

**Critical Paths (100% coverage):**
- Authentication flows
- Money transactions
- Account balance updates
- Chore reward crediting
- Multi-tenant data isolation

**General (80% coverage):**
- API endpoints
- Business logic
- UI components

### Backend Testing (pytest)

See `docs/TESTING.md`

---

## Cost Management

### POC Cost Breakdown

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| Lightsail Container | Micro (1GB RAM) | $10 |
| Lightsail PostgreSQL | Standard (40GB) | $15 |
| S3 | Standard tier | $0-1 |
| CloudFront | Free tier | $0 |
| SES | Email sending | $0-1 |
| Route 53 | DNS (optional) | $0.50 |
| **Total** | | **$25-28** |

### Cost Monitoring Setup

**AWS Budgets:**
```bash
# Create budget via Terraform
resource "aws_budgets_budget" "picklesapp_dev" {
  name              = "picklesapp-dev-monthly"
  budget_type       = "COST"
  limit_amount      = "30"
  limit_unit        = "USD"
  time_period_start = "2026-02-01_00:00"
  time_unit         = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = ["alerts@picklesapp.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = ["alerts@picklesapp.com"]
  }
}
```

**CloudWatch Alarms:**
```bash
# Daily cost alarm
resource "aws_cloudwatch_metric_alarm" "daily_cost" {
  alarm_name          = "picklesapp-daily-cost-alert"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name        = "EstimatedCharges"
  namespace          = "AWS/Billing"
  period             = "86400"  # 1 day
  statistic          = "Maximum"
  threshold          = "2"  # $2/day = ~$60/mo
  alarm_description  = "Alert if daily cost exceeds $2"
  alarm_actions      = [aws_sns_topic.alerts.arn]
}
```

### Cost Optimization Strategies

**Development:**
- Use local Docker for primary development
- Deploy to AWS 2-3x per week for integration testing only
- Stop dev environment overnight (manual or Lambda)

**Database:**
- Start with minimal storage (40GB Lightsail)
- Monitor growth weekly
- Vacuum/analyze regularly to prevent bloat

**Logs:**
- 3-day retention in POC
- ERROR level only in production
- Filter verbose logs before CloudWatch ingestion

**Data Transfer:**
- Keep all services in same region (us-east-1)
- Use CloudFront for static assets (free tier)
- Compress API responses (gzip)

**Backups:**
- Lightsail includes automated backups
- Don't create manual snapshots unless needed
- Delete old snapshots regularly

### Scaling Costs

| Phase | Families | Monthly Cost | Notes |
|-------|----------|--------------|-------|
| POC | 0-100 | $25-30 | Lightsail only |
| Beta | 100-500 | $50-75 | HA database, enhanced monitoring |
| Production | 500-2000 | $150-300 | ECS, RDS, ALB, ElastiCache |
| Scale | 2000+ | $300-800 | Reserved instances, multi-region |

### Revenue Model

**Free Tier:**
- Up to 4 family members
- Basic features (chores, allowances, accounts)
- 90-day transaction history
- Community support

**Premium Tier: $5/month or $50/year**
- Unlimited family members
- Advanced features:
  - Savings goals with projections
  - Budget worksheets
  - Investment account tracking
  - Loan amortization
  - Analytics dashboard
- Full transaction history
- Priority support
- Photo evidence upload

**Target:**
- 5% conversion to premium
- 1000 families = 50 premium = $250/mo revenue
- Covers $150-200/mo infrastructure at scale

---

## Business Model

### Freemium Structure

**Free Forever:**
- 2 parents, 2 kids max
- 3 accounts per kid (spending, saving, giving)
- Basic chores (unlimited)
- Simple allowances
- 90-day transaction history
- Email support

**Premium ($5/mo or $50/year):**
- Unlimited family members
- Unlimited accounts
- Advanced chore scheduling
- Photo evidence
- Savings goals with projections
- Budget worksheets
- Investment tracking
- Loan calculator
- Full transaction history
- CSV export
- Priority support
- Early access to new features

**Why Freemium:**
1. Low barrier to entry (try before buy)
2. Free tier serves most small families
3. Power users upgrade for advanced features
4. Sustainable revenue for infrastructure

### Pricing Philosophy

- **Family-friendly pricing**: $5/mo = 2 Starbucks coffees
- **Annual discount**: $50/year = 17% savings (2 months free)
- **No hidden fees**: No per-kid pricing, no transaction fees
- **Free trial**: 30 days premium, no credit card required
- **Educational discount**: Free premium for teachers (Phase 8+)

### Go-to-Market Strategy

**Phase 5-6 (Beta Launch):**
- Invite-only beta
- Personal network + parenting forums
- Target: 50-100 families
- Gather feedback, iterate

**Phase 7-8 (Public Launch):**
- Product Hunt launch
- Parenting subreddits (r/Parenting, r/personalfinance)
- Parenting blogs/influencers
- Facebook parenting groups
- Target: 500 families

**Phase 9+ (Growth):**
- SEO content (parenting + financial literacy)
- Paid ads (Facebook, Google)
- Referral program (free month for referrals)
- Partnerships (schools, financial literacy orgs)
- Target: 2000+ families

### Success Metrics

**Phase 5-6:**
- 50 active families
- 70% user retention (week 2)
- <0.5% churn
- NPS score >50

**Phase 7-8:**
- 500 active families
- 5% conversion to premium
- 80% user retention
- <1% churn
- NPS score >60

**Phase 9:**
- 2000 active families
- 10% conversion to premium
- $1000+/mo revenue
- 85% user retention
- <2% churn
- NPS score >70

---

## Appendix

### Glossary

- **Family**: Group of users (parents + kids) sharing accounts and chores
- **Multi-Household**: Kid belongs to 2+ families (divorced parents)
- **Family Context**: Current family scope for data isolation
- **Age Bracket**: UI tier based on child's age (young/tween/teen)
- **Preset Profile**: Age-appropriate default settings for kids
- **IOU Account**: Virtual account tracking money (not real cards)
- **Absolute Timing**: Chore scheduled at fixed intervals from start date
- **Relative Timing**: Chore scheduled from last completion date
- **First-Dibs**: Chore available to multiple kids, first to claim gets it
- **Overdue Days**: Cumulative days a chore is past due date

### Key Decision Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-01-29 | Native PostgreSQL vs Docker | Performance, developer familiarity |
| 2026-01-29 | FastAPI over Flask | Async support, auto-docs, modern |
| 2026-01-29 | Vite over CRA | Faster builds, modern tooling |
| 2026-01-29 | Context + Hooks over Redux | Simpler for app complexity level |
| 2026-01-29 | REST first, GraphQL later | REST proven, GraphQL optional |
| 2026-01-29 | Multi-tenant via middleware + RLS | Defense in depth |
| 2026-01-29 | Option B for multi-household | One login, context switching |
| 2026-01-29 | Freemium model | Accessible + sustainable |
| 2026-01-29 | Lightsail for POC | Predictable costs, simplicity |

### References

- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [React Documentation](https://react.dev/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Vite Guide](https://vitejs.dev/guide/)
- [RFC 5545 (iCalendar)](https://tools.ietf.org/html/rfc5545)
- [OAuth 2.0 Specification](https://oauth.net/2/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

### Contact & Support

- **Project Owner**: Jenny Wadkins
- **Repository**: https://github.com/threnjen/chore-app
- **Documentation**: This file (DEVELOPMENT_PLAN.md)

---

*Last Updated: January 29, 2026*
*Version: 1.0*
*Status: Planning Complete, Ready for Phase 1 Implementation*
