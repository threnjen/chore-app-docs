# Phase 2: Chores MVP Implementation Guide

**Estimated Duration**: 2-3 weeks  
**Status**: 🔄 In Progress  
**Prerequisites**: Phase 1 Complete  
**Last Updated**: January 31, 2026

---

## Overview

Phase 2 implements the core chore management system with advanced scheduling capabilities. This phase introduces the recurrence rule engine, chore assignment workflow, and parent approval system.

### Key Features

| Feature | Description | Priority |
|---------|-------------|----------|
| Chore CRUD | Create, read, update, delete chores | Critical |
| Recurrence Rules | RFC 5545-based scheduling with extensions | Critical |
| Chore Assignments | Instance generation from recurrence rules | Critical |
| Completion Workflow | Kids mark chores complete | Critical |
| Approval Workflow | Parents approve/reject completions | Critical |
| Reward/Penalty System | Configurable credits on approval, debits on rejection/expiration | Critical |
| Background Jobs | Scheduled instance generation | High |
| Overdue Tracking | Single instance with cumulative overdue days | High |

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Recurrence Format | RFC 5545 RRULE + extensions | Industry standard, extensible |
| Timing Modes | Absolute vs Relative | Flexibility for different chore types |
| Scheduler | APScheduler with PostgreSQL | Persistent job storage, reliability |
| Overdue Policy | Single instance, no backlog | Prevents overwhelming kids |
| Assignment Type | ASSIGNED only (Phase 2) | First-dibs deferred to Phase 3 |

---

## Prerequisites

### From Phase 1

- [x] Docker development environment running
- [x] PostgreSQL database with migrations applied
- [x] User authentication (JWT tokens)
- [x] Family membership and multi-tenant context
- [x] Account and Transaction models
- [x] Transaction service with balance updates

### New Dependencies

```yaml
# Add to requirements.txt
APScheduler>=3.10.0
python-dateutil>=2.8.2
```

---

## Repository Structure Changes

### chore-app-backend

```
app/
├── models/
│   ├── chore.py              # NEW: Chore, ChoreAssignment models
│   └── __init__.py           # UPDATE: Export new models
├── schemas/
│   ├── chore.py              # NEW: Chore schemas
│   └── __init__.py           # UPDATE: Export new schemas
├── routers/
│   ├── chores.py             # NEW: Chore endpoints
│   ├── chore_assignments.py  # NEW: Assignment endpoints
│   └── __init__.py           # UPDATE: Register new routers
├── services/
│   ├── chore_service.py      # NEW: Chore business logic
│   ├── recurrence_service.py # NEW: Recurrence rule engine
│   └── scheduler_service.py  # NEW: APScheduler setup
├── jobs/
│   └── __init__.py           # NEW: Background job definitions
│   └── chore_jobs.py         # NEW: Chore instance generation
alembic/
└── versions/
    └── 004_add_chores.py     # NEW: Chore tables migration
tests/
├── test_chores.py            # NEW: Chore endpoint tests
├── test_chore_assignments.py # NEW: Assignment tests
└── test_recurrence.py        # NEW: Recurrence algorithm tests
```

### chore-app-frontend

```
src/
├── pages/
│   ├── ChoresPage.jsx           # NEW: Chore list for parents
│   ├── ChoreCreatePage.jsx      # NEW: Create chore form
│   ├── ChoreDetailPage.jsx      # NEW: View/edit chore
│   ├── MyChoresPage.jsx         # NEW: Kid's chore dashboard
│   └── PendingApprovalsPage.jsx # NEW: Parent approval queue
├── components/
│   └── chores/
│       ├── ChoreCard.jsx         # NEW: Chore display card
│       ├── ChoreForm.jsx         # NEW: Create/edit form
│       ├── RecurrenceBuilder.jsx # NEW: Schedule UI
│       ├── ChoreAssignmentCard.jsx # NEW: Assignment display
│       └── ApprovalActions.jsx   # NEW: Approve/reject buttons
├── services/
│   └── choreService.js          # NEW: Chore API calls
e2e/
└── chores.spec.js               # NEW: E2E chore tests
```

---

## Implementation Checklist

### 1. Database Models & Migration

**Files to create:**

- [ ] `app/models/chore.py`
- [ ] `alembic/versions/004_add_chores.py`

**Files to update:**

- [ ] `app/models/transaction.py` — Add `CHORE_PENALTY` to `TransactionCategory` enum

#### TransactionCategory Enum Update

```python
class TransactionCategory(str, enum.Enum):
    """Category of transaction for reporting."""
    
    CHORE_REWARD = "CHORE_REWARD"   # Earned from completing chore
    CHORE_PENALTY = "CHORE_PENALTY" # NEW: Deducted for rejection/expiration/late completion
    ALLOWANCE = "ALLOWANCE"         # Regular allowance payment
    INTEREST = "INTEREST"           # Interest earned on savings
    TRANSFER_IN = "TRANSFER_IN"     # Received from another account
    TRANSFER_OUT = "TRANSFER_OUT"   # Sent to another account
    MANUAL_CREDIT = "MANUAL_CREDIT" # Manual deposit by parent
    MANUAL_DEBIT = "MANUAL_DEBIT"   # Manual withdrawal by parent
    PURCHASE = "PURCHASE"           # Money spent
    DONATION = "DONATION"           # Money donated (Give account)
    OTHER = "OTHER"                 # Miscellaneous
```

> **Migration Note**: The `004_add_chores.py` migration must also add the `CHORE_PENALTY` value to the existing `transactioncategory` PostgreSQL enum type.

#### Chore Model

| Column | Type | Constraints |
|--------|------|-------------|
| id | UUID | PK, default uuid4 |
| family_id | UUID | FK → families.id, not null |
| title | String(255) | Not null |
| description | Text | Nullable |
| reward_amount | Decimal(10,2) | Nullable (credit on approval) |
| reward_type | Enum | 'money', 'points', 'stars' |
| penalty_amount | Decimal(10,2) | Nullable (debit on rejection/expiration) |
| penalty_behavior | Enum | 'none', 'on_rejection', 'on_expiration', 'on_both' |
| credit_on_complete | Boolean | default false (credit immediately on completion vs. on approval) |
| assignment_type | Enum | 'ASSIGNED', 'FIRST_DIBS' |
| recurrence_rule | JSONB | RFC 5545 + extensions |
| timing_mode | Enum | 'absolute', 'relative' |
| deadline_time | Time | Nullable |
| estimated_duration | Integer | Minutes, nullable |
| require_photo | Boolean | default false |
| require_approval | Boolean | default true |
| auto_approve_after | Integer | Hours, nullable |
| allow_late_completion | Boolean | default true (can complete after due date) |
| expiration_days | Integer | Nullable (days after due date before expiration) |
| grace_period_days | Integer | Nullable (days after due date before "late") |
| grace_period_hours | Integer | Nullable (additional hours after grace days) |
| late_reward_percentage | Integer | Nullable (percentage of reward_amount for late completion, e.g., 50 = 50%) |
| late_penalty_amount | Decimal(10,2) | Nullable (penalty for late-but-valid completion) |
| early_bonus_amount | Decimal(10,2) | Nullable (extra reward for early completion) |
| early_threshold_hours | Integer | Nullable (hours before deadline to qualify for bonus) |
| penalty_per_overdue_day | Decimal(10,2) | Nullable (additional penalty per day overdue) |
| max_penalty_amount | Decimal(10,2) | Nullable (cap on total penalty) |
| is_active | Boolean | default true |
| created_by_id | UUID | FK → users.id |
| created_at | DateTime | default now |
| updated_at | DateTime | onupdate now |

#### Penalty Behavior Enum

```python
class PenaltyBehavior(str, enum.Enum):
    NONE = "NONE"                 # No penalty ever applied
    ON_REJECTION = "ON_REJECTION" # Debit when parent rejects completion
    ON_EXPIRATION = "ON_EXPIRATION" # Debit when chore expires without completion
    ON_BOTH = "ON_BOTH"           # Debit on either rejection or expiration

class RewardTier(str, enum.Enum):
    FULL = "FULL"       # Full reward_amount (on-time or early completion)
    REDUCED = "REDUCED" # late_reward_percentage of reward_amount (late completion)
    BONUS = "BONUS"     # reward_amount + early_bonus_amount (early completion)
    NONE = "NONE"       # No reward (neutral chore or failed completion)
```

#### ChoreAssignment Model

| Column | Type | Constraints |
|--------|------|-------------|
| id | UUID | PK, default uuid4 |
| chore_id | UUID | FK → chores.id, not null |
| family_id | UUID | FK → families.id, not null |
| assigned_to_id | UUID | FK → users.id, nullable |
| due_date | Date | Not null |
| due_time | Time | Nullable |
| status | Enum | See status enum below |
| claimed_by_id | UUID | FK → users.id, nullable |
| claimed_at | DateTime | Nullable |
| completed_at | DateTime | Nullable |
| completed_by_id | UUID | FK → users.id, nullable |
| approved_at | DateTime | Nullable |
| approved_by_id | UUID | FK → users.id, nullable |
| rejected_at | DateTime | Nullable |
| rejection_reason | Text | Nullable |
| expired_at | DateTime | Nullable |
| total_overdue_days | Integer | default 0 |
| photo_urls | ARRAY(Text) | Nullable |
| reward_amount | Decimal(10,2) | Snapshot from chore |
| penalty_amount | Decimal(10,2) | Snapshot from chore |
| penalty_behavior | String | Snapshot from chore |
| grace_period_days | Integer | Snapshot from chore |
| grace_period_hours | Integer | Snapshot from chore |
| late_reward_percentage | Integer | Snapshot from chore |
| late_penalty_amount | Decimal(10,2) | Snapshot from chore |
| early_bonus_amount | Decimal(10,2) | Snapshot from chore |
| early_threshold_hours | Integer | Snapshot from chore |
| penalty_per_overdue_day | Decimal(10,2) | Snapshot from chore |
| max_penalty_amount | Decimal(10,2) | Snapshot from chore |
| completed_on_time | Boolean | Nullable (calculated on completion) |
| reward_tier | Enum | 'FULL', 'REDUCED', 'BONUS', 'NONE' (determined on completion) |
| effective_reward | Decimal(10,2) | Actual amount credited |
| effective_penalty | Decimal(10,2) | Actual amount debited |
| reward_credited | Boolean | default false |
| penalty_applied | Boolean | default false |
| created_at | DateTime | default now |
| updated_at | DateTime | onupdate now |

#### Assignment Status Enum

```python
class AssignmentStatus(str, enum.Enum):
    PENDING = "PENDING"       # Assigned, not yet due
    AVAILABLE = "AVAILABLE"   # First-dibs, waiting to be claimed
    CLAIMED = "CLAIMED"       # First-dibs, claimed by kid
    COMPLETED = "COMPLETED"   # Kid marked complete, awaiting approval
    APPROVED = "APPROVED"     # Parent approved, reward credited
    REJECTED = "REJECTED"     # Parent rejected, needs redo
    EXPIRED = "EXPIRED"       # Past deadline, no longer completable
```

**Acceptance Criteria:**

- [ ] Migration creates both tables with all columns
- [ ] Foreign keys with proper ON DELETE behavior
- [ ] Indexes on family_id, chore_id, assigned_to_id, due_date, status
- [ ] JSONB column for recurrence_rule with GIN index
- [ ] Migration is reversible

---

### 2. Recurrence Rule Engine

**Files to create:**

- [ ] `app/services/recurrence_service.py`

#### Recurrence Rule Format

```python
@dataclass
class RecurrenceRule:
    """
    RFC 5545 RRULE with PicklesApp extensions.
    
    Stored as JSONB in chore.recurrence_rule column.
    """
    frequency: str          # DAILY, WEEKLY, MONTHLY, YEARLY
    interval: int = 1       # Every N frequency units
    by_weekday: list[str] | None = None   # ["MO", "WE", "FR"]
    by_monthday: list[int] | None = None  # [1, 15] or [-1] for last day
    by_month: list[int] | None = None     # [6, 7, 8] for summer
    timing_mode: str = "absolute"          # "absolute" or "relative"
    start_date: date = None
    end_date: date | None = None
    count: int | None = None              # Max occurrences
```

#### Supported Recurrence Patterns

| Pattern | Rule JSON |
|---------|-----------|
| Every day | `{"frequency": "DAILY", "interval": 1}` |
| Weekdays only | `{"frequency": "WEEKLY", "interval": 1, "by_weekday": ["MO","TU","WE","TH","FR"]}` |
| M-W-F | `{"frequency": "WEEKLY", "interval": 1, "by_weekday": ["MO","WE","FR"]}` |
| Every Tuesday | `{"frequency": "WEEKLY", "interval": 1, "by_weekday": ["TU"]}` |
| Bi-weekly Saturday | `{"frequency": "WEEKLY", "interval": 2, "by_weekday": ["SA"]}` |
| Every 3 days | `{"frequency": "DAILY", "interval": 3}` |
| 1st of month | `{"frequency": "MONTHLY", "interval": 1, "by_monthday": [1]}` |
| 1st and 15th | `{"frequency": "MONTHLY", "interval": 1, "by_monthday": [1, 15]}` |
| Last day of month | `{"frequency": "MONTHLY", "interval": 1, "by_monthday": [-1]}` |
| First Monday of month | `{"frequency": "MONTHLY", "interval": 1, "by_weekday": ["1MO"]}` |

#### Instance Generation Algorithm

```python
class RecurrenceService:
    """
    Service for generating chore instances from recurrence rules.
    """
    
    async def generate_instances(
        self,
        chore: Chore,
        from_date: date,
        limit: int = 10
    ) -> list[date]:
        """
        Generate upcoming instance dates for a chore.
        
        For absolute mode: Generate from chore's start_date
        For relative mode: Generate from last completion date
        """
        rule = chore.recurrence_rule
        
        if rule["timing_mode"] == "relative":
            # Find last completed assignment
            last_completed = await self._get_last_completed(chore.id)
            base_date = last_completed.completed_at.date() if last_completed else rule["start_date"]
        else:
            base_date = rule["start_date"]
        
        return self._calculate_occurrences(rule, base_date, from_date, limit)
    
    def _calculate_occurrences(
        self,
        rule: dict,
        base_date: date,
        after_date: date,
        limit: int
    ) -> list[date]:
        """
        Calculate occurrence dates using dateutil.rrule.
        """
        from dateutil.rrule import rrule, DAILY, WEEKLY, MONTHLY, YEARLY
        
        freq_map = {
            "DAILY": DAILY,
            "WEEKLY": WEEKLY,
            "MONTHLY": MONTHLY,
            "YEARLY": YEARLY
        }
        
        weekday_map = {
            "MO": 0, "TU": 1, "WE": 2, "TH": 3,
            "FR": 4, "SA": 5, "SU": 6
        }
        
        by_weekday = None
        if rule.get("by_weekday"):
            by_weekday = [weekday_map[d] for d in rule["by_weekday"]]
        
        rr = rrule(
            freq=freq_map[rule["frequency"]],
            interval=rule.get("interval", 1),
            byweekday=by_weekday,
            bymonthday=rule.get("by_monthday"),
            bymonth=rule.get("by_month"),
            dtstart=base_date,
            until=rule.get("end_date"),
            count=rule.get("count")
        )
        
        # Get occurrences after the specified date
        occurrences = []
        for dt in rr:
            if dt.date() >= after_date:
                occurrences.append(dt.date())
                if len(occurrences) >= limit:
                    break
        
        return occurrences
```

**Acceptance Criteria:**

- [ ] Parses all supported recurrence patterns
- [ ] Generates correct dates for daily, weekly, monthly patterns
- [ ] Handles custom weekday patterns (M-W-F)
- [ ] Handles bi-weekly and custom intervals
- [ ] Respects timing_mode (absolute vs relative)
- [ ] Handles edge cases (leap year, month boundaries, DST)

---

### 3. Background Job Scheduler

**Files to create:**

- [ ] `app/services/scheduler_service.py`
- [ ] `app/jobs/__init__.py`
- [ ] `app/jobs/chore_jobs.py`

#### APScheduler Configuration

```python
# app/services/scheduler_service.py
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.jobstores.sqlalchemy import SQLAlchemyJobStore

class SchedulerService:
    """
    APScheduler service for background job management.
    """
    
    def __init__(self, database_url: str):
        self.scheduler = AsyncIOScheduler(
            jobstores={
                'default': SQLAlchemyJobStore(url=database_url)
            },
            job_defaults={
                'coalesce': True,
                'max_instances': 1
            },
            timezone='UTC'
        )
    
    def start(self):
        """Start the scheduler with all registered jobs."""
        from app.jobs.chore_jobs import register_chore_jobs
        register_chore_jobs(self.scheduler)
        self.scheduler.start()
    
    def shutdown(self):
        """Gracefully shutdown scheduler."""
        self.scheduler.shutdown(wait=True)
```

#### Chore Jobs

```python
# app/jobs/chore_jobs.py

def register_chore_jobs(scheduler: AsyncIOScheduler):
    """Register all chore-related background jobs."""
    
    # Generate upcoming chore instances hourly
    scheduler.add_job(
        func=generate_chore_instances_job,
        trigger='cron',
        hour='*',  # Every hour
        minute=0,
        id='generate_chore_instances',
        replace_existing=True
    )
    
    # Update overdue tracking daily at midnight
    scheduler.add_job(
        func=update_overdue_chores_job,
        trigger='cron',
        hour=0,
        minute=5,
        id='update_overdue_chores',
        replace_existing=True
    )
    
    # Auto-approve expired approvals
    scheduler.add_job(
        func=auto_approve_chores_job,
        trigger='cron',
        hour='*',
        minute=30,
        id='auto_approve_chores',
        replace_existing=True
    )
    
    # Expire overdue chores and apply penalties
    scheduler.add_job(
        func=expire_chores_job,
        trigger='cron',
        hour=0,
        minute=15,
        id='expire_chores',
        replace_existing=True
    )

async def generate_chore_instances_job():
    """
    Generate upcoming chore assignment instances.
    
    Creates assignments for the next 7 days for all active chores.
    Only creates if assignment doesn't already exist for that date.
    """
    async with get_db_session() as db:
        service = ChoreService(db)
        await service.generate_upcoming_instances(days_ahead=7)

async def update_overdue_chores_job():
    """
    Update total_overdue_days for all pending/claimed assignments.
    """
    async with get_db_session() as db:
        service = ChoreService(db)
        await service.update_overdue_tracking()

async def auto_approve_chores_job():
    """
    Auto-approve completed chores past their auto_approve_after threshold.
    """
    async with get_db_session() as db:
        service = ChoreService(db)
        await service.process_auto_approvals()

async def expire_chores_job():
    """
    Expire overdue chores past their expiration_days threshold.
    Applies penalties according to penalty_behavior configuration.
    """
    async with get_db_session() as db:
        service = ChoreService(db)
        await service.process_expirations()
```

**Acceptance Criteria:**

- [ ] Scheduler starts with FastAPI app lifecycle
- [ ] Jobs persist across restarts (PostgreSQL job store)
- [ ] Instance generation runs hourly
- [ ] Overdue tracking updates daily
- [ ] Auto-approval processes hourly
- [ ] Jobs handle errors gracefully (don't crash scheduler)

---

### 4. Chore Service Layer

**Files to create:**

- [ ] `app/services/chore_service.py`

#### ChoreService Class

```python
class ChoreService:
    """
    Service for chore and assignment business logic.
    """
    
    def __init__(self, db: AsyncSession):
        self.db = db
        self.recurrence_service = RecurrenceService(db)
    
    # Chore CRUD
    async def create_chore(
        self,
        family_id: UUID,
        created_by_id: UUID,
        data: ChoreCreate
    ) -> Chore:
        """Create a new chore with initial assignment generation."""
        
    async def update_chore(
        self,
        chore_id: UUID,
        data: ChoreUpdate
    ) -> Chore:
        """Update chore settings. May regenerate future assignments."""
        
    async def delete_chore(
        self,
        chore_id: UUID
    ) -> None:
        """Soft delete chore and cancel future assignments."""
    
    # Assignment Management
    async def get_assignments_for_user(
        self,
        user_id: UUID,
        family_id: UUID,
        filters: AssignmentFilters
    ) -> list[ChoreAssignment]:
        """Get assignments for a specific user."""
        
    async def get_pending_approvals(
        self,
        family_id: UUID,
        parent_id: UUID
    ) -> list[ChoreAssignment]:
        """Get all completed assignments awaiting approval."""
    
    # Workflow Actions
    async def complete_assignment(
        self,
        assignment_id: UUID,
        user_id: UUID,
        photo_urls: list[str] | None = None
    ) -> tuple[ChoreAssignment, Transaction | None]:
        """
        Mark assignment as completed.
        - Calculates completed_on_time based on grace period
        - Determines reward_tier and effective_reward
        - May credit reward if credit_on_complete=True
        - May apply late_penalty_amount if completed late
        """
        
    async def approve_assignment(
        self,
        assignment_id: UUID,
        approver_id: UUID
    ) -> tuple[ChoreAssignment, Transaction | None]:
        """Approve assignment and credit effective_reward (if not already credited on complete)."""
        
    async def reject_assignment(
        self,
        assignment_id: UUID,
        rejector_id: UUID,
        reason: str,
        apply_penalty: bool = True
    ) -> tuple[ChoreAssignment, Transaction | None]:
        """Reject assignment with reason. Optionally apply penalty debit."""
    
    async def expire_assignment(
        self,
        assignment_id: UUID
    ) -> tuple[ChoreAssignment, Transaction | None]:
        """Mark assignment as expired. Apply escalating penalty if configured."""
    
    # Reward/Penalty Calculation Helpers
    def _calculate_completion_timing(
        self,
        assignment: ChoreAssignment,
        completed_at: datetime
    ) -> tuple[bool, RewardTier]:
        """
        Determine if completion is on-time and which reward tier applies.
        
        Returns (completed_on_time, reward_tier)
        """
        
    def _calculate_effective_reward(
        self,
        assignment: ChoreAssignment,
        reward_tier: RewardTier
    ) -> Decimal:
        """
        Calculate the actual reward amount based on tier.
        
        - BONUS: reward_amount + early_bonus_amount
        - FULL: reward_amount
        - REDUCED: reward_amount * (late_reward_percentage / 100)
        - NONE: 0
        """
        
    def _calculate_effective_penalty(
        self,
        assignment: ChoreAssignment,
        is_escalating: bool = False
    ) -> Decimal:
        """
        Calculate the actual penalty amount.
        
        For escalating penalties:
        penalty = min(penalty_amount + (penalty_per_overdue_day * total_overdue_days), max_penalty_amount)
        """
    
    # Background Job Support
    async def generate_upcoming_instances(
        self,
        days_ahead: int = 7
    ) -> int:
        """Generate assignment instances for all active chores."""
        
    async def update_overdue_tracking(self) -> int:
        """Update overdue days for all pending assignments."""
        
    async def process_auto_approvals(self) -> int:
        """Auto-approve eligible completed assignments."""
    
    async def process_expirations(self) -> int:
        """Expire overdue assignments past their expiration_days threshold."""
```

#### Completion Workflow with Tier Calculation

```python
async def complete_assignment(
    self,
    assignment_id: UUID,
    user_id: UUID,
    photo_urls: list[str] | None = None
) -> tuple[ChoreAssignment, Transaction | None]:
    """
    Mark assignment as completed with reward tier calculation.
    
    1. Validate assignment is in PENDING/CLAIMED status
    2. Check if late completion is allowed (if applicable)
    3. Calculate completed_on_time and reward_tier
    4. Calculate effective_reward based on tier
    5. Apply late_penalty_amount if completing late (mutually exclusive with expiration penalty)
    6. Credit reward if credit_on_complete=True
    """
    assignment = await self._get_assignment(assignment_id)
    now = datetime.now(timezone.utc)
    
    # Validate status
    if assignment.status not in [AssignmentStatus.PENDING, AssignmentStatus.CLAIMED]:
        raise HTTPException(400, "Assignment not available for completion")
    
    # Check late completion permission
    due_datetime = datetime.combine(assignment.due_date, assignment.due_time or time.max)
    if now > due_datetime and not assignment.allow_late_completion:
        raise HTTPException(400, "Late completion not allowed for this chore")
    
    # Calculate timing and tier
    completed_on_time, reward_tier = self._calculate_completion_timing(assignment, now)
    effective_reward = self._calculate_effective_reward(assignment, reward_tier)
    
    # Update assignment
    assignment.status = AssignmentStatus.COMPLETED
    assignment.completed_at = now
    assignment.completed_by_id = user_id
    assignment.photo_urls = photo_urls
    assignment.completed_on_time = completed_on_time
    assignment.reward_tier = reward_tier
    assignment.effective_reward = effective_reward
    
    transaction = None
    
    # Apply late penalty if completing late (mutually exclusive with expiration)
    if (not completed_on_time 
            and assignment.late_penalty_amount 
            and assignment.late_penalty_amount > 0
            and not assignment.penalty_applied):
        assignment.effective_penalty = assignment.late_penalty_amount
        transaction = await self._apply_penalty(
            assignment, 
            created_by_id=None, 
            amount=assignment.late_penalty_amount,
            description_prefix="Late completion penalty"
        )
        assignment.penalty_applied = True
    
    # Credit reward immediately if configured
    if assignment.credit_on_complete and effective_reward > 0:
        transaction = await self._credit_reward(assignment, user_id, amount=effective_reward)
        assignment.reward_credited = True
    
    await self.db.commit()
    return assignment, transaction

def _calculate_completion_timing(
    self,
    assignment: ChoreAssignment,
    completed_at: datetime
) -> tuple[bool, RewardTier]:
    """Determine if completion is on-time and which reward tier applies."""
    due_datetime = datetime.combine(
        assignment.due_date, 
        assignment.due_time or time.max,
        tzinfo=timezone.utc
    )
    
    # Check for early completion bonus
    if assignment.early_threshold_hours and assignment.early_bonus_amount:
        early_threshold = due_datetime - timedelta(hours=assignment.early_threshold_hours)
        if completed_at <= early_threshold:
            return True, RewardTier.BONUS
    
    # Check for on-time completion (before grace period)
    grace_delta = timedelta(
        days=assignment.grace_period_days or 0,
        hours=assignment.grace_period_hours or 0
    )
    grace_deadline = due_datetime + grace_delta
    
    # If no grace period configured, any completion before expiration is "on-time"
    if assignment.grace_period_days is None and assignment.grace_period_hours is None:
        return True, RewardTier.FULL
    
    if completed_at <= due_datetime:
        return True, RewardTier.FULL
    elif completed_at <= grace_deadline:
        # Late but within grace period
        if assignment.late_reward_percentage is not None:
            return False, RewardTier.REDUCED
        return False, RewardTier.FULL  # No reduced tier configured
    else:
        # Past grace period
        if assignment.late_reward_percentage is not None:
            return False, RewardTier.REDUCED
        return False, RewardTier.NONE

def _calculate_effective_reward(
    self,
    assignment: ChoreAssignment,
    reward_tier: RewardTier
) -> Decimal:
    """Calculate actual reward based on tier."""
    base_reward = assignment.reward_amount or Decimal("0")
    
    if reward_tier == RewardTier.BONUS:
        return base_reward + (assignment.early_bonus_amount or Decimal("0"))
    elif reward_tier == RewardTier.FULL:
        return base_reward
    elif reward_tier == RewardTier.REDUCED:
        percentage = assignment.late_reward_percentage or 100
        return base_reward * Decimal(percentage) / Decimal(100)
    else:  # NONE
        return Decimal("0")
```

#### Approval Workflow with Tiered Reward Crediting

```python
async def approve_assignment(
    self,
    assignment_id: UUID,
    approver_id: UUID
) -> tuple[ChoreAssignment, Transaction | None]:
    """
    Approve a completed chore assignment and credit the effective_reward.
    
    1. Validate assignment is in COMPLETED status
    2. Validate approver is a parent in the family
    3. Update assignment status to APPROVED
    4. Create transaction crediting effective_reward to child's account (if not already credited)
    5. Return both assignment and transaction
    """
    assignment = await self._get_assignment(assignment_id)
    
    if assignment.status != AssignmentStatus.COMPLETED:
        raise HTTPException(400, "Assignment not in completed status")
    
    await self._validate_parent_permission(approver_id, assignment.family_id)
    
    assignment.status = AssignmentStatus.APPROVED
    assignment.approved_at = datetime.now(timezone.utc)
    assignment.approved_by_id = approver_id
    
    # Credit effective_reward if not already credited on completion
    transaction = None
    if (assignment.effective_reward and assignment.effective_reward > 0 
            and not assignment.reward_credited):
        transaction = await self._credit_reward(
            assignment, 
            approver_id, 
            amount=assignment.effective_reward
        )
        assignment.reward_credited = True
    
    await self.db.commit()
    return assignment, transaction
```

#### Rejection Workflow with Penalty Debit

```python
async def reject_assignment(
    self,
    assignment_id: UUID,
    rejector_id: UUID,
    reason: str,
    apply_penalty: bool = True
) -> tuple[ChoreAssignment, Transaction | None]:
    """
    Reject a completed chore assignment and optionally apply penalty.
    
    1. Validate assignment is in COMPLETED status
    2. Validate rejector is a parent in the family
    3. Update assignment status to REJECTED
    4. If penalty configured and apply_penalty=True, debit child's account
    5. Late penalty and rejection penalty are mutually exclusive
    5. Allow re-completion (status can go back to COMPLETED)
    """
    assignment = await self._get_assignment(assignment_id)
    
    if assignment.status != AssignmentStatus.COMPLETED:
        raise HTTPException(400, "Assignment not in completed status")
    
    await self._validate_parent_permission(rejector_id, assignment.family_id)
    
    assignment.status = AssignmentStatus.REJECTED
    assignment.rejected_at = datetime.now(timezone.utc)
    assignment.rejection_reason = reason
    
    # Apply penalty if configured
    transaction = None
    penalty_behavior = assignment.penalty_behavior or "NONE"
    
    if (apply_penalty 
            and assignment.penalty_amount 
            and assignment.penalty_amount > 0
            and penalty_behavior in ["ON_REJECTION", "ON_BOTH"]
            and not assignment.penalty_applied):
        effective_penalty = self._calculate_effective_penalty(assignment, is_escalating=False)
        assignment.effective_penalty = effective_penalty
        transaction = await self._apply_penalty(
            assignment, 
            rejector_id, 
            amount=effective_penalty
        )
        assignment.penalty_applied = True
    
    await self.db.commit()
    return assignment, transaction
```

#### Expiration Workflow with Escalating Penalty

```python
async def expire_assignment(
    self,
    assignment_id: UUID
) -> tuple[ChoreAssignment, Transaction | None]:
    """
    Mark an overdue assignment as expired and apply escalating penalty if configured.
    
    Called by background job when assignment exceeds expiration_days.
    Escalating penalty formula:
    penalty = min(penalty_amount + (penalty_per_overdue_day * total_overdue_days), max_penalty_amount)
    """
    assignment = await self._get_assignment(assignment_id)
    
    if assignment.status not in [AssignmentStatus.PENDING, AssignmentStatus.CLAIMED]:
        return assignment, None  # Already processed
    
    assignment.status = AssignmentStatus.EXPIRED
    assignment.expired_at = datetime.now(timezone.utc)
    
    # Apply penalty if configured
    transaction = None
    penalty_behavior = assignment.penalty_behavior or "NONE"
    
    if (assignment.penalty_amount 
            and assignment.penalty_amount > 0
            and penalty_behavior in ["ON_EXPIRATION", "ON_BOTH"]
            and not assignment.penalty_applied):
        # Calculate escalating penalty
        effective_penalty = self._calculate_effective_penalty(assignment, is_escalating=True)
        assignment.effective_penalty = effective_penalty
        transaction = await self._apply_penalty(
            assignment, 
            created_by_id=None, 
            amount=effective_penalty
        )
        assignment.penalty_applied = True
    
    await self.db.commit()
    return assignment, transaction

def _calculate_effective_penalty(
    self,
    assignment: ChoreAssignment,
    is_escalating: bool = False
) -> Decimal:
    """
    Calculate the actual penalty amount.
    
    For escalating penalties (on expiration):
    penalty = min(base + (per_day * overdue_days), max)
    """
    base_penalty = assignment.penalty_amount or Decimal("0")
    
    if not is_escalating or not assignment.penalty_per_overdue_day:
        return base_penalty
    
    escalation = assignment.penalty_per_overdue_day * assignment.total_overdue_days
    total_penalty = base_penalty + escalation
    
    # Apply cap if configured
    if assignment.max_penalty_amount:
        total_penalty = min(total_penalty, assignment.max_penalty_amount)
    
    return total_penalty

async def _apply_penalty(
    self, 
    assignment: ChoreAssignment, 
    created_by_id: UUID | None,
    amount: Decimal | None = None,
    description_prefix: str = "Penalty"
) -> Transaction:
    """Debit penalty amount from child's account."""
    account = await self._get_default_account(
        assignment.assigned_to_id or assignment.completed_by_id,
        assignment.family_id
    )
    
    penalty_amount = amount if amount is not None else assignment.penalty_amount
    
    transaction_service = TransactionService(self.db)
    return await transaction_service.create_transaction(
        account_id=account.id,
        amount=penalty_amount,
        transaction_type=TransactionType.DEBIT,
        category=TransactionCategory.CHORE_PENALTY,
        description=f"{description_prefix} for: {assignment.chore.title}",
        created_by_id=created_by_id,
        reference_id=assignment.id,
        reference_type="chore_assignment"
    )

async def _credit_reward(
    self, 
    assignment: ChoreAssignment, 
    created_by_id: UUID,
    amount: Decimal | None = None
) -> Transaction:
    """Credit reward amount to child's account."""
    account = await self._get_default_account(
        assignment.assigned_to_id or assignment.completed_by_id,
        assignment.family_id
    )
    
    reward_amount = amount if amount is not None else assignment.reward_amount
    tier_suffix = f" ({assignment.reward_tier})" if assignment.reward_tier else ""
    
    transaction_service = TransactionService(self.db)
    return await transaction_service.create_transaction(
        account_id=account.id,
        amount=reward_amount,
        transaction_type=TransactionType.CREDIT,
        category=TransactionCategory.CHORE_REWARD,
        description=f"Reward for: {assignment.chore.title}{tier_suffix}",
        created_by_id=created_by_id,
        reference_id=assignment.id,
        reference_type="chore_assignment"
    )
```

**Acceptance Criteria:**

- [ ] CRUD operations respect family isolation
- [ ] Assignments generated correctly from recurrence rules
- [ ] Completion workflow updates status appropriately
- [ ] Completion calculates `completed_on_time` based on grace period
- [ ] Completion determines `reward_tier` (FULL, REDUCED, BONUS, NONE)
- [ ] Completion calculates `effective_reward` based on tier
- [ ] Late penalty applied if completing past grace period (mutually exclusive with expiration)
- [ ] Early bonus added to reward if completing before threshold
- [ ] Completion can optionally credit reward immediately (credit_on_complete)
- [ ] Approval workflow credits `effective_reward` atomically (if not already credited)
- [ ] Rejection workflow optionally applies penalty debit
- [ ] Expiration workflow applies escalating penalty if configured
- [ ] Escalating penalty formula: `min(base + (per_day × overdue_days), max)`
- [ ] Penalties respect penalty_behavior configuration
- [ ] Rewards and penalties are only applied once (tracked via flags)
- [ ] Rejection allows re-completion
- [ ] Overdue tracking calculated correctly

---

### Reward/Penalty Configuration Examples

The following examples show common parent setups using Simple vs Advanced mode:

| Scenario | Mode | Field Configuration |
|----------|------|---------------------|
| **Basic reward only** | Simple | `reward_amount=5.00`, `penalty_behavior=NONE` |
| **Reward + penalty on skip** | Simple | `reward_amount=5.00`, `penalty_amount=2.00`, `penalty_behavior=ON_EXPIRATION` |
| **Reward + penalty on rejection** | Simple | `reward_amount=5.00`, `penalty_amount=2.00`, `penalty_behavior=ON_REJECTION` |
| **Neutral chore (tracking only)** | Simple | `reward_amount=null`, `penalty_behavior=NONE` |
| **On-time full, late half** | Advanced | `reward_amount=5.00`, `grace_period_days=1`, `late_reward_percentage=50` |
| **Strict deadline + penalty** | Advanced | `reward_amount=5.00`, `grace_period_days=0`, `grace_period_hours=0`, `penalty_amount=2.00`, `penalty_behavior=ON_EXPIRATION` |
| **Early completion bonus** | Advanced | `reward_amount=5.00`, `early_bonus_amount=2.00`, `early_threshold_hours=24` |
| **Escalating penalty** | Advanced | `penalty_amount=1.00`, `penalty_per_overdue_day=0.50`, `max_penalty_amount=5.00`, `penalty_behavior=ON_EXPIRATION` |
| **Multi-tier complete** | Advanced | `reward_amount=5.00`, `grace_period_days=2`, `late_reward_percentage=50`, `penalty_amount=2.00`, `penalty_per_overdue_day=0.25`, `max_penalty_amount=5.00`, `penalty_behavior=ON_EXPIRATION` |
| **No late completion allowed** | Advanced | `allow_late_completion=false`, `expiration_days=1`, `penalty_amount=2.00`, `penalty_behavior=ON_EXPIRATION` |

#### Decision Tree: Which Tier Applies?

```
Completion received
├── Is completion before (due_datetime - early_threshold_hours)?
│   └── YES → BONUS tier (reward + early_bonus)
│   └── NO ↓
├── Is completion before due_datetime?
│   └── YES → FULL tier (full reward)
│   └── NO ↓
├── Is grace_period configured?
│   └── NO → FULL tier (any completion is "on-time")
│   └── YES ↓
├── Is completion within grace_period?
│   └── YES → REDUCED tier (late_reward_percentage of reward)
│   └── NO → REDUCED tier + late_penalty_amount (if configured)
```

---

### 5. Pydantic Schemas

**Files to create:**

- [ ] `app/schemas/chore.py`

#### Schema Definitions

```python
# Base schemas
class RecurrenceRuleBase(BaseModel):
    """Recurrence rule configuration."""
    frequency: Literal["DAILY", "WEEKLY", "MONTHLY", "YEARLY"]
    interval: int = Field(default=1, ge=1, le=365)
    by_weekday: list[Literal["MO", "TU", "WE", "TH", "FR", "SA", "SU"]] | None = None
    by_monthday: list[int] | None = Field(default=None, description="Day(s) of month (1-31, -1 for last)")
    by_month: list[int] | None = Field(default=None, description="Month(s) of year (1-12)")
    timing_mode: Literal["absolute", "relative"] = "absolute"
    start_date: date
    end_date: date | None = None
    count: int | None = Field(default=None, ge=1)

class ChoreBase(BaseModel):
    """Base chore fields."""
    title: str = Field(..., min_length=1, max_length=255)
    description: str | None = None
    
    # Reward configuration
    reward_amount: Decimal | None = Field(default=None, ge=0, description="Full reward on approval")
    reward_type: Literal["money", "points", "stars"] = "money"
    credit_on_complete: bool = Field(default=False, description="Credit immediately on completion vs. on approval")
    
    # Tiered reward configuration (Advanced mode)
    grace_period_days: int | None = Field(default=None, ge=0, description="Days after due date before 'late'")
    grace_period_hours: int | None = Field(default=None, ge=0, description="Additional hours after grace days")
    late_reward_percentage: int | None = Field(default=None, ge=0, le=100, description="Percentage of reward for late completion")
    early_bonus_amount: Decimal | None = Field(default=None, ge=0, description="Extra reward for early completion")
    early_threshold_hours: int | None = Field(default=None, ge=1, description="Hours before deadline for early bonus")
    
    # Penalty configuration
    penalty_amount: Decimal | None = Field(default=None, ge=0, description="Base penalty on rejection/expiration")
    penalty_behavior: Literal["NONE", "ON_REJECTION", "ON_EXPIRATION", "ON_BOTH"] = "NONE"
    late_penalty_amount: Decimal | None = Field(default=None, ge=0, description="Penalty for late-but-valid completion")
    
    # Escalating penalty configuration (Advanced mode)
    penalty_per_overdue_day: Decimal | None = Field(default=None, ge=0, description="Additional penalty per day overdue")
    max_penalty_amount: Decimal | None = Field(default=None, ge=0, description="Cap on total penalty")
    
    # Assignment configuration
    assignment_type: Literal["ASSIGNED", "FIRST_DIBS"] = "ASSIGNED"
    deadline_time: time | None = None
    estimated_duration: int | None = Field(default=None, ge=1, description="Duration in minutes")
    require_photo: bool = False
    require_approval: bool = True
    auto_approve_after: int | None = Field(default=None, ge=1, description="Hours until auto-approve")
    allow_late_completion: bool = Field(default=True, description="Can complete after due date")
    expiration_days: int | None = Field(default=None, ge=1, description="Days after due date before expiration")

class ChoreCreate(ChoreBase):
    """Create chore request."""
    recurrence_rule: RecurrenceRuleBase
    assigned_to_user_ids: list[UUID] = Field(default_factory=list)

class ChoreUpdate(BaseModel):
    """Update chore request. All reward/penalty fields updatable."""
    title: str | None = None
    description: str | None = None
    reward_amount: Decimal | None = None
    reward_type: Literal["money", "points", "stars"] | None = None
    credit_on_complete: bool | None = None
    grace_period_days: int | None = None
    grace_period_hours: int | None = None
    late_reward_percentage: int | None = None
    early_bonus_amount: Decimal | None = None
    early_threshold_hours: int | None = None
    penalty_amount: Decimal | None = None
    penalty_behavior: Literal["NONE", "ON_REJECTION", "ON_EXPIRATION", "ON_BOTH"] | None = None
    late_penalty_amount: Decimal | None = None
    penalty_per_overdue_day: Decimal | None = None
    max_penalty_amount: Decimal | None = None
    deadline_time: time | None = None
    estimated_duration: int | None = None
    require_photo: bool | None = None
    require_approval: bool | None = None
    auto_approve_after: int | None = None
    allow_late_completion: bool | None = None
    expiration_days: int | None = None
    is_active: bool | None = None

class ChoreRead(ChoreBase):
    """Chore response."""
    id: UUID
    family_id: UUID
    recurrence_rule: dict
    is_active: bool
    created_by_id: UUID
    created_at: datetime
    updated_at: datetime
    next_instance_date: date | None = None  # Computed
    
    model_config = ConfigDict(from_attributes=True)

# Assignment schemas
class ChoreAssignmentRead(BaseModel):
    """Chore assignment response with tiered reward/penalty tracking."""
    id: UUID
    chore_id: UUID
    chore_title: str
    family_id: UUID
    assigned_to_id: UUID | None
    assigned_to_name: str | None
    due_date: date
    due_time: time | None
    status: str
    total_overdue_days: int
    
    # Reward tracking
    reward_amount: Decimal | None
    completed_on_time: bool | None
    reward_tier: Literal["FULL", "REDUCED", "BONUS", "NONE"] | None
    effective_reward: Decimal | None
    reward_credited: bool
    
    # Penalty tracking
    penalty_amount: Decimal | None
    penalty_behavior: str
    effective_penalty: Decimal | None
    penalty_applied: bool
    
    # Timestamps
    completed_at: datetime | None
    approved_at: datetime | None
    rejected_at: datetime | None
    expired_at: datetime | None
    rejection_reason: str | None
    
    model_config = ConfigDict(from_attributes=True)

class AssignmentComplete(BaseModel):
    """Complete assignment request."""
    photo_urls: list[str] = Field(default_factory=list)

class AssignmentReject(BaseModel):
    """Reject assignment request."""
    reason: str = Field(..., min_length=1, max_length=500)
    apply_penalty: bool = Field(default=True, description="Apply configured penalty on rejection")
```

**Acceptance Criteria:**

- [ ] All schemas use Pydantic v2 syntax
- [ ] Proper validation with Field() constraints
- [ ] ConfigDict for ORM integration
- [ ] Clear documentation with descriptions

---

### 6. API Endpoints

**Files to create:**

- [ ] `app/routers/chores.py`
- [ ] `app/routers/chore_assignments.py`

#### Chore Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/chores` | List chores for family |
| POST | `/chores` | Create new chore |
| GET | `/chores/{chore_id}` | Get chore details |
| PATCH | `/chores/{chore_id}` | Update chore |
| DELETE | `/chores/{chore_id}` | Soft delete chore |
| GET | `/chores/{chore_id}/preview` | Preview next N occurrences |

#### Assignment Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/chore-assignments` | List assignments (filterable) |
| GET | `/chore-assignments/my` | Get current user's assignments |
| GET | `/chore-assignments/pending-approval` | Parent's approval queue |
| GET | `/chore-assignments/{id}` | Get assignment details |
| POST | `/chore-assignments/{id}/complete` | Mark as complete |
| POST | `/chore-assignments/{id}/approve` | Approve completion |
| POST | `/chore-assignments/{id}/reject` | Reject completion |
| POST | `/chore-assignments/batch-approve` | Approve multiple |

**Acceptance Criteria:**

- [ ] All endpoints require authentication
- [ ] All endpoints respect family context
- [ ] Parents can view all family chores/assignments
- [ ] Children can only view their own assignments
- [ ] Only parents can approve/reject
- [ ] Proper error handling and status codes

---

### 7. Frontend Chore Pages

**Files to create:**

- [ ] `src/pages/ChoresPage.jsx`
- [ ] `src/pages/ChoreCreatePage.jsx`
- [ ] `src/pages/ChoreDetailPage.jsx`
- [ ] `src/pages/MyChoresPage.jsx`
- [ ] `src/pages/PendingApprovalsPage.jsx`

#### ChoresPage (Parent View)

```jsx
// List of all family chores with management options
<ChoresPage>
  <PageHeader>
    <Title>Chores</Title>
    <Button onClick={navigateToCreate}>+ New Chore</Button>
  </PageHeader>
  
  <ChoreFilters>
    <Select label="Status">Active / Inactive / All</Select>
    <Select label="Assigned To">Filter by child</Select>
  </ChoreFilters>
  
  <ChoreList>
    {chores.map(chore => (
      <ChoreCard key={chore.id} chore={chore} />
    ))}
  </ChoreList>
</ChoresPage>
```

#### ChoreCreatePage

```jsx
// Create chore with Simple/Advanced mode toggle
<ChoreCreatePage>
  <PageHeader>
    <BackButton />
    <Title>Create Chore</Title>
  </PageHeader>
  
  <ChoreForm onSubmit={handleCreate}>
    {/* Basic Info */}
    <Section title="Chore Details">
      <Input name="title" label="Title" required />
      <Textarea name="description" label="Description" />
      <Select name="assignedTo" label="Assign To" multiple>
        {familyChildren.map(child => ...)}
      </Select>
    </Section>
    
    {/* Reward/Penalty Section with Simple/Advanced Toggle */}
    <Section title="Reward & Penalty">
      <ModeToggle 
        value={advancedMode} 
        onChange={setAdvancedMode}
        labels={["Simple", "Advanced"]}
      />
      
      {/* Simple Mode - Always visible */}
      <Input name="rewardAmount" type="number" label="Reward Amount" prefix="$" />
      <Input name="penaltyAmount" type="number" label="Penalty Amount" prefix="$" />
      <Select name="penaltyBehavior" label="Apply Penalty When">
        <Option value="NONE">Never</Option>
        <Option value="ON_REJECTION">Chore is rejected</Option>
        <Option value="ON_EXPIRATION">Chore expires incomplete</Option>
        <Option value="ON_BOTH">Either rejection or expiration</Option>
      </Select>
      
      {/* Advanced Mode - Expanded fields */}
      {advancedMode && (
        <>
          <Divider label="Grace Period" />
          <div className="grid grid-cols-2 gap-4">
            <Input name="gracePeriodDays" type="number" label="Days" min={0} />
            <Input name="gracePeriodHours" type="number" label="Hours" min={0} />
          </div>
          <Help>Time after due date before completion is considered "late"</Help>
          
          <Divider label="Late Completion" />
          <Input 
            name="lateRewardPercentage" 
            type="number" 
            label="Late Reward %" 
            min={0} max={100}
            placeholder="e.g., 50 for half reward"
          />
          <Input 
            name="latePenaltyAmount" 
            type="number" 
            label="Late Penalty" 
            prefix="$"
          />
          <Help>Penalty applied when completing after grace period</Help>
          
          <Divider label="Early Completion Bonus" />
          <div className="grid grid-cols-2 gap-4">
            <Input name="earlyBonusAmount" type="number" label="Bonus Amount" prefix="$" />
            <Input name="earlyThresholdHours" type="number" label="Hours Early" />
          </div>
          <Help>Extra reward if completed this many hours before deadline</Help>
          
          <Divider label="Escalating Penalties" />
          <div className="grid grid-cols-2 gap-4">
            <Input name="penaltyPerOverdueDay" type="number" label="Per Day Late" prefix="$" />
            <Input name="maxPenaltyAmount" type="number" label="Maximum Penalty" prefix="$" />
          </div>
          <Help>Additional penalty per day overdue, capped at maximum</Help>
        </>
      )}
    </Section>
    
    {/* Schedule */}
    <Section title="Schedule">
      <RecurrenceBuilder
        value={recurrenceRule}
        onChange={setRecurrenceRule}
      />
      <RecurrencePreview rule={recurrenceRule} count={5} />
    </Section>
    
    {/* Options */}
    <Section title="Options">
      <TimeInput name="deadlineTime" label="Due By Time" />
      <Input name="estimatedDuration" type="number" label="Estimated Minutes" />
      <Checkbox name="requirePhoto" label="Require Photo" />
      <Checkbox name="requireApproval" label="Require Approval" />
      <Input name="autoApproveAfter" type="number" label="Auto-approve After (hours)" />
    </Section>
    
    <SubmitButton>Create Chore</SubmitButton>
  </ChoreForm>
</ChoreCreatePage>
```

#### RecurrenceBuilder Component

```jsx
// Advanced recurrence rule builder
<RecurrenceBuilder value={rule} onChange={setRule}>
  {/* Frequency Selection */}
  <Select name="frequency" label="Repeats">
    <Option value="DAILY">Daily</Option>
    <Option value="WEEKLY">Weekly</Option>
    <Option value="MONTHLY">Monthly</Option>
  </Select>
  
  {/* Interval */}
  <Input name="interval" type="number" label="Every" min={1} />
  <Label>{frequencyLabel}</Label> {/* "days" / "weeks" / "months" */}
  
  {/* Weekly Options */}
  {frequency === "WEEKLY" && (
    <WeekdayPicker
      value={byWeekday}
      onChange={setByWeekday}
    />
  )}
  
  {/* Monthly Options */}
  {frequency === "MONTHLY" && (
    <MonthDayPicker
      value={byMonthday}
      onChange={setByMonthday}
    />
  )}
  
  {/* Timing Mode */}
  <RadioGroup name="timingMode" label="Schedule Type">
    <Radio value="absolute">
      Fixed Schedule
      <Help>Next chore is always on the scheduled day, regardless of when completed</Help>
    </Radio>
    <Radio value="relative">
      Flexible Schedule
      <Help>Next chore is scheduled relative to when this one is completed</Help>
    </Radio>
  </RadioGroup>
  
  {/* Start/End Dates */}
  <DateInput name="startDate" label="Start Date" required />
  <DateInput name="endDate" label="End Date (Optional)" />
</RecurrenceBuilder>
```

#### MyChoresPage (Kid View)

```jsx
// Kid's chore dashboard
<MyChoresPage>
  <PageHeader>
    <Title>My Chores</Title>
  </PageHeader>
  
  {/* Today's Chores */}
  <Section title="Today">
    {todayChores.map(assignment => (
      <ChoreAssignmentCard
        key={assignment.id}
        assignment={assignment}
        onComplete={handleComplete}
      />
    ))}
  </Section>
  
  {/* Overdue */}
  {overdueChores.length > 0 && (
    <Section title="Overdue" variant="warning">
      {overdueChores.map(assignment => (
        <ChoreAssignmentCard
          key={assignment.id}
          assignment={assignment}
          onComplete={handleComplete}
          showOverdueBadge
        />
      ))}
    </Section>
  )}
  
  {/* Upcoming */}
  <Section title="Coming Up">
    {upcomingChores.map(assignment => (
      <ChoreAssignmentCard
        key={assignment.id}
        assignment={assignment}
        disabled
      />
    ))}
  </Section>
</MyChoresPage>
```

#### PendingApprovalsPage (Parent View)

```jsx
// Parent approval queue
<PendingApprovalsPage>
  <PageHeader>
    <Title>Pending Approvals</Title>
    {pendingCount > 1 && (
      <Button onClick={approveAll}>Approve All ({pendingCount})</Button>
    )}
  </PageHeader>
  
  <ApprovalList>
    {pendingApprovals.map(assignment => (
      <ApprovalCard key={assignment.id}>
        <ChoreInfo>
          <Title>{assignment.chore_title}</Title>
          <CompletedBy>{assignment.assigned_to_name}</CompletedBy>
          <CompletedAt>{formatDate(assignment.completed_at)}</CompletedAt>
          {assignment.photo_urls?.length > 0 && (
            <PhotoGallery photos={assignment.photo_urls} />
          )}
        </ChoreInfo>
        
        <Actions>
          <Button variant="success" onClick={() => approve(assignment.id)}>
            ✓ Approve (${assignment.reward_amount})
          </Button>
          <Button variant="danger" onClick={() => openRejectModal(assignment)}>
            ✗ Reject
          </Button>
        </Actions>
      </ApprovalCard>
    ))}
  </ApprovalList>
  
  {/* Reject Modal */}
  <RejectModal
    isOpen={rejectModalOpen}
    assignment={selectedAssignment}
    onReject={handleReject}
    onClose={closeRejectModal}
  />
</PendingApprovalsPage>
```

**Acceptance Criteria:**

- [ ] ChoresPage lists all family chores
- [ ] ChoreCreatePage with full recurrence builder
- [ ] RecurrencePreview shows next 5 occurrences
- [ ] MyChoresPage shows kid's assignments by status
- [ ] Overdue chores highlighted with badge
- [ ] PendingApprovalsPage shows approval queue
- [ ] Approve/reject actions work correctly
- [ ] Proper loading and error states

---

### 8. Chore Service (Frontend)

**Files to create:**

- [ ] `src/services/choreService.js`

```javascript
import api from './api'

const choreService = {
  // Chores
  async getChores(filters = {}) {
    const params = new URLSearchParams(filters)
    const response = await api.get(`/chores?${params}`)
    return response.data
  },
  
  async getChore(choreId) {
    const response = await api.get(`/chores/${choreId}`)
    return response.data
  },
  
  async createChore(data) {
    const response = await api.post('/chores', data)
    return response.data
  },
  
  async updateChore(choreId, data) {
    const response = await api.patch(`/chores/${choreId}`, data)
    return response.data
  },
  
  async deleteChore(choreId) {
    await api.delete(`/chores/${choreId}`)
  },
  
  async previewOccurrences(choreId, count = 5) {
    const response = await api.get(`/chores/${choreId}/preview?count=${count}`)
    return response.data
  },
  
  // Assignments
  async getAssignments(filters = {}) {
    const params = new URLSearchParams(filters)
    const response = await api.get(`/chore-assignments?${params}`)
    return response.data
  },
  
  async getMyAssignments(filters = {}) {
    const params = new URLSearchParams(filters)
    const response = await api.get(`/chore-assignments/my?${params}`)
    return response.data
  },
  
  async getPendingApprovals() {
    const response = await api.get('/chore-assignments/pending-approval')
    return response.data
  },
  
  async completeAssignment(assignmentId, photoUrls = []) {
    const response = await api.post(`/chore-assignments/${assignmentId}/complete`, {
      photo_urls: photoUrls
    })
    return response.data
  },
  
  async approveAssignment(assignmentId) {
    const response = await api.post(`/chore-assignments/${assignmentId}/approve`)
    return response.data
  },
  
  async rejectAssignment(assignmentId, reason) {
    const response = await api.post(`/chore-assignments/${assignmentId}/reject`, {
      reason
    })
    return response.data
  },
  
  async batchApprove(assignmentIds) {
    const response = await api.post('/chore-assignments/batch-approve', {
      assignment_ids: assignmentIds
    })
    return response.data
  }
}

export default choreService
```

---

### 9. Navigation Updates

**Files to update:**

- [ ] `src/App.jsx` - Add chore routes
- [ ] `src/components/layout/Sidebar.jsx` - Add chore navigation

#### New Routes

```javascript
// Add to App.jsx routes
<Route path="/chores" element={<ProtectedRoute><ChoresPage /></ProtectedRoute>} />
<Route path="/chores/create" element={<ProtectedRoute><ChoreCreatePage /></ProtectedRoute>} />
<Route path="/chores/:choreId" element={<ProtectedRoute><ChoreDetailPage /></ProtectedRoute>} />
<Route path="/my-chores" element={<ProtectedRoute><MyChoresPage /></ProtectedRoute>} />
<Route path="/approvals" element={<ProtectedRoute><PendingApprovalsPage /></ProtectedRoute>} />
```

#### Sidebar Navigation

```javascript
// Parent navigation
{
  name: 'Chores',
  path: '/chores',
  icon: ClipboardListIcon
},
{
  name: 'Pending Approvals',
  path: '/approvals',
  icon: CheckCircleIcon,
  badge: pendingApprovalCount
}

// Child navigation
{
  name: 'My Chores',
  path: '/my-chores',
  icon: ClipboardListIcon
}
```

---

## Testing Requirements

See [PHASE_2_TESTING.md](./PHASE_2_TESTING.md) for complete test scenarios.

### Coverage Targets

| Component | Target | Priority |
|-----------|--------|----------|
| Recurrence algorithm | 100% | Critical |
| Assignment workflow | 100% | Critical |
| Approval/rejection | 100% | Critical |
| Reward crediting | 100% | Critical |
| Chore CRUD | 80% | High |
| Background jobs | 80% | High |
| Frontend components | 60% | Medium |

---

## Documentation Updates

### Files to Update

- [ ] `chore-app-docs/README.md` - Update phase status
- [ ] `chore-app-docs/docs/shared/DEVELOPMENT_PLAN.md` - Mark Phase 2 complete
- [ ] `chore-app-backend/README.md` - Add chore API documentation
- [ ] `chore-app-frontend/README.md` - Add chore pages documentation

### Files to Create

- [ ] `chore-app-docs/docs/backend/CHORE_SCHEDULING.md` - Algorithm documentation
- [ ] `chore-app-docs/docs/phases/PHASE_2_TESTING.md` - Test scenarios

---

## Deployment Checklist

### Pre-deployment

- [ ] All tests passing (backend and frontend)
- [ ] Migration runs successfully on test database
- [ ] Background scheduler starts without errors
- [ ] API documentation updated

### Post-deployment

- [ ] Verify chore creation works
- [ ] Verify assignment generation works
- [ ] Verify completion/approval workflow
- [ ] Verify reward crediting
- [ ] Monitor background job execution

---

## Phase 2 Completion Criteria

- [ ] Chore and ChoreAssignment models implemented
- [ ] Recurrence rule engine supports all patterns
- [ ] APScheduler configured with PostgreSQL backend
- [ ] Instance generation job running hourly
- [ ] Chore CRUD endpoints functional
- [ ] Assignment workflow complete (complete → approve/reject)
- [ ] Reward crediting on approval working
- [ ] Parent dashboard shows pending approvals
- [ ] Kid dashboard shows today's chores
- [ ] Overdue tracking with badge display
- [ ] All backend tests passing (80%+ coverage)
- [ ] All frontend tests passing
- [ ] E2E tests for chore workflows passing
- [ ] Documentation updated

---

## Phase 1 Amendment: Bidirectional Family Building

**Added**: During Phase 2 development  
**Status**: ✅ Complete

### Overview

This amendment adds enhanced registration flow and bidirectional family building, which was identified as a missing Phase 1 component. Users can now:

1. **Self-register with age-based role determination** - Birth year determines PARENT (18+) or CHILD (<18) role
2. **Children can request to join families** - Instead of only parents inviting children
3. **Adults can create OR join families** - Option to join existing family via parent email
4. **Email verification required** - Must verify email before join requests can be approved

### Database Changes

#### Migration 005: Extend Invitations Table

```sql
-- New enums
CREATE TYPE invitationtype AS ENUM ('INVITE', 'JOIN_REQUEST');
CREATE TYPE invitationstatus AS ENUM ('PENDING', 'ACCEPTED', 'REJECTED', 'EXPIRED');

-- New columns on invitations table
ALTER TABLE invitations ADD COLUMN invitation_type invitationtype NOT NULL DEFAULT 'INVITE';
ALTER TABLE invitations ADD COLUMN status invitationstatus NOT NULL DEFAULT 'PENDING';
ALTER TABLE invitations ADD COLUMN message TEXT;
ALTER TABLE invitations ADD COLUMN reviewed_by_id UUID REFERENCES users(id);
ALTER TABLE invitations ADD COLUMN reviewed_at TIMESTAMP WITH TIME ZONE;
ALTER TABLE invitations ADD COLUMN target_email VARCHAR(255);
ALTER TABLE invitations ALTER COLUMN family_id DROP NOT NULL;  -- Allow null for pending join requests
```

#### Migration 006: User Verification Fields

```sql
ALTER TABLE users ADD COLUMN email_verified BOOLEAN NOT NULL DEFAULT FALSE;
ALTER TABLE users ADD COLUMN email_verification_token VARCHAR(64) UNIQUE;
ALTER TABLE users ADD COLUMN birth_year INTEGER;
```

### API Changes

#### Updated Endpoints

| Endpoint | Change |
|----------|--------|
| `POST /auth/register` | Now requires `birth_year`, returns `RegistrationResponse` instead of tokens |

#### New Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/auth/verify-email` | POST | Verify email with token, returns tokens on success |
| `/auth/resend-verification` | POST | Resend verification email |
| `/join-requests` | POST | Create join request to a family |
| `/me/join-requests` | GET | Get current user's join requests |
| `/families/{id}/join-requests` | GET | List pending join requests (parents only) |
| `/join-requests/{id}/review` | POST | Approve or reject a join request |
| `/join-requests/{id}` | DELETE | Cancel a pending join request |

### Registration Flow

```
User enters birth year
         │
         ▼
    ┌────────────┐
    │ Age >= 18? │
    └────────────┘
         │
    ┌────┴────┐
    ▼         ▼
 PARENT     CHILD
    │         │
    ▼         ▼
Optional   Required
parent     parent
email      email
    │         │
    ▼         ▼
┌─────────────────────┐
│  Create account     │
│  (email unverified) │
└─────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
Has parent  No parent
  email       email
    │         │
    ▼         ▼
Create     Create
join       family
request    (adult only)
    │         │
    ▼         ▼
┌─────────────────────┐
│ Send verification   │
│ email to user       │
└─────────────────────┘
```

### Join Request Flow

```
User creates join request
         │
         ▼
    ┌─────────────────┐
    │ Target email    │
    │ registered?     │
    └─────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
   Yes        No
    │         │
    ▼         ▼
Target has  Send
 family?   registration
    │      invitation
    ▼         │
   Yes        │
    │         │
    ▼         │
Create join   │
request for   │
that family   │
    │         │
    ▼         │
Notify all    │
parents       │
    │         │
    ▼         ▼
┌─────────────────────┐
│ Wait for approval   │
│ (30-day expiration) │
└─────────────────────┘
         │
         ▼
    ┌─────────────────┐
    │ Parent reviews  │
    │ request         │
    └─────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
Approve    Reject
    │         │
    ▼         ▼
Check if   Notify
requester  requester
email      (can retry)
verified
    │
    ▼
Add to family
& notify
```

### Email Templates (Stubbed in Dev Mode)

| Template | Trigger | Recipients |
|----------|---------|------------|
| `send_verification_email` | User registers | New user |
| `send_join_request_notification` | User creates join request | Parent(s) in target family |
| `send_join_request_approved` | Parent approves request | Requester |
| `send_join_request_rejected` | Parent rejects request | Requester |
| `send_registration_invitation` | User requests join to unregistered email | Target email |

### Frontend Changes

| Component | Change |
|-----------|--------|
| `RegisterForm.jsx` | Multi-step: (1) birth year, (2) basic info, (3) family connection |
| `RegisterPage.jsx` | Handles invitation token from URL |
| `DashboardPage.jsx` | Shows pending join requests badge for parents |
| `EmailVerificationPage.jsx` | New page for email verification flow |
| `JoinRequestsPage.jsx` | New page for parents to review requests |
| `App.jsx` | Added routes for `/verify-email/:token` and `/family/join-requests` |

### Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `AGE_THRESHOLD` | 18 | Age below which user is a CHILD |
| Join request expiration | 30 days | Time before join request expires |
| Invitation expiration | 7 days | Time before invitation expires |

### Testing Notes

- All email functions are stubbed in development mode (logged to console)
- Birth year validation: 1900 to current year
- Children must provide parent email during registration
- Adults can optionally provide parent email to join existing family
- Email verification is required before join request approval completes
