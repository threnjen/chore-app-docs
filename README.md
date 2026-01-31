# PicklesApp Documentation















































































































































































































































































































































































































































































































































































































| Missed instances | Job didn't run | Check APScheduler logs || Wrong dates | Timezone mismatch | Ensure family timezone is applied || Duplicate assignments | Race condition | Use database unique constraint || No assignments generated | Chore is inactive | Check `is_active` flag ||-------|-------|----------|| Issue | Cause | Solution |### Common Issues- `approval_queue_size` - Gauge of pending approvals- `overdue_assignments_count` - Gauge of current overdue assignments- `chore_generation_duration` - Time to run generation job- `chore_instances_generated` - Counter of assignments createdTrack the following metrics for monitoring:### Metrics```logger.info(f"Chore instance generation job completed: {stats}")# Log job executionlogger.error(f"Failed to generate instances for chore {chore.id}: {error}")# Log errorslogger.info(f"Generated {count} assignments for chore {chore.id}")# Log instance generationlogger = logging.getLogger("chore_scheduler")import logging```python### Logging## Monitoring & Debugging---```        ]            date(2026, 3, 19)   # 2 weeks from Mar 5            date(2026, 3, 5),   # 2 weeks from Feb 19            date(2026, 2, 19),  # 2 weeks from Feb 5        assert occurrences == [                )            limit=3            after_date=date(2026, 2, 6),            base_date=date(2026, 2, 5),  # Completion date            rule=rule,        occurrences = service.calculate_occurrences(        # Simulate completion on Feb 5 (4 days late)                }            "start_date": "2026-02-01"            "timing_mode": "relative",            "interval": 2,            "frequency": "WEEKLY",        rule = {    def test_relative_mode_after_completion(self, service):            ]            date(2026, 2, 13)   # Fri            date(2026, 2, 11),  # Wed            date(2026, 2, 9),   # Mon            date(2026, 2, 6),   # Fri            date(2026, 2, 4),   # Wed            date(2026, 2, 2),   # Mon        assert occurrences == [                )            limit=6            after_date=date(2026, 2, 2),            rule=rule,        occurrences = service.calculate_occurrences(                }            "start_date": "2026-02-02"  # Monday            "by_weekday": ["MO", "WE", "FR"],            "interval": 1,            "frequency": "WEEKLY",        rule = {    def test_weekly_mwf(self, service):            ]            date(2026, 2, 5)            date(2026, 2, 4),            date(2026, 2, 3),            date(2026, 2, 2),            date(2026, 2, 1),        assert occurrences == [                )            limit=5            after_date=date(2026, 2, 1),            rule=rule,        occurrences = service.calculate_occurrences(                }            "start_date": "2026-02-01"            "interval": 1,            "frequency": "DAILY",        rule = {    def test_daily_every_day(self, service):            return RecurrenceService()    def service(self):    @pytest.fixture    class TestRecurrenceService:from app.services.recurrence_service import RecurrenceServicefrom datetime import dateimport pytest```python### Test Fixtures## Testing Recurrence Rules---```)    timezone='UTC'    },        'misfire_grace_time': 3600  # 1 hour grace period        'max_instances': 1,    # Prevent concurrent runs        'coalesce': True,      # Combine missed runs    job_defaults={    },        )            tablename='apscheduler_jobs'            url=settings.database_url,        'default': SQLAlchemyJobStore(    jobstores={scheduler = AsyncIOScheduler(from apscheduler.jobstores.sqlalchemy import SQLAlchemyJobStorefrom apscheduler.schedulers.asyncio import AsyncIOScheduler```python### APScheduler Configuration| `expire_chores` | Daily at 00:15 | Mark expired assignments || `auto_approve_chores` | Every hour at :30 | Auto-approve eligible chores || `update_overdue_chores` | Daily at 00:05 | Update overdue tracking || `generate_chore_instances` | Every hour at :00 | Create upcoming assignments ||-----|----------|---------|| Job | Schedule | Purpose |### Job Schedule## Background Jobs Configuration---```WHERE status = 'COMPLETED';ON chore_assignments(family_id, status) CREATE INDEX idx_chore_assignments_pending_approval -- For pending approval queriesON chore_assignments(assigned_to_id, status, due_date);CREATE INDEX idx_chore_assignments_user_status_date -- Compound index for common query patternCREATE INDEX idx_chore_assignments_due_date ON chore_assignments(due_date);-- Date-based queries (most common)CREATE INDEX idx_chore_assignments_status ON chore_assignments(status);-- Status-based queriesCREATE INDEX idx_chore_assignments_assigned_to ON chore_assignments(assigned_to_id);CREATE INDEX idx_chore_assignments_family ON chore_assignments(family_id);CREATE INDEX idx_chore_assignments_chore ON chore_assignments(chore_id);-- Primary lookups```sqlFor optimal query performance, the following indexes are created:## Database Indexes---```└──────────────────────────────────────────────────────────────────┘│                                                                  ││                      └──► EXPIRED (if no one claims)             ││                      │                                           ││                      │                   completion)             ││                      │             └──► (Can unclaim before      ││                      │             │                             ││   [Generated] ──► AVAILABLE ──► CLAIMED ──► COMPLETED ──► ...   ││                                                                  ││   ────────────────────────────────                               ││   FIRST_DIBS Chore Flow (Phase 3):                              ││                                                                  │┌──────────────────────────────────────────────────────────────────┐└──────────────────────────────────────────────────────────────────┘│                                                                  ││                      └──► EXPIRED (if expiration enabled)        ││                      │                                           ││                      │                     (Kid can retry)       ││                      │                     └────────────┘        ││                      │                     │            │        ││                      │             └──► REJECTED ──► COMPLETED   ││                      │             │                             ││                      │             │                 Credited)   ││                      │             │            └──► (Reward     ││                      │             │            │                ││   [Generated] ──► PENDING ──► COMPLETED ──► APPROVED            ││                                                                  ││   ─────────────────────                                         ││   ASSIGNED Chore Flow:                                          ││                                                                  │┌──────────────────────────────────────────────────────────────────┐```## Assignment Lifecycle---- Deadline times apply in the new family timezone- Due dates are stored as date only (no timezone)- All deadline times shift to the new timezoneIf a family changes their timezone:### Timezone Changes- During DST transition, the wall clock time is preserved- 8:00 AM deadline stays at 8:00 AM local timeDeadline times are stored without timezone and applied using the family's timezone:### Daylight Saving Time- February → skips or occurs on 28th/29th- 30-day months → skips or occurs on 30th (configurable)- Months with 31 days → occurs on 31stFor monthly chores on the 31st:### Month Boundary Handling- Feb in non-leap years → skips to March (or uses 28th per dateutil behavior)- Feb 29th in leap years → occursFor monthly chores on the 29th:### Leap Year Handling## Edge Cases---```└─────────────────────────────────────┘│ [Mark Complete]                     ││ Originally due: Monday              ││ 🗑️ Take Out Trash      🔴 3 days   │┌─────────────────────────────────────┐Overdue:└─────────────────────────────────────┘│ [Mark Complete]                     ││ Due: Today at 8 PM                  ││ 🧹 Clean Room                       │┌─────────────────────────────────────┐Today's Chores:```### Display```    await db.commit()            assignment.total_overdue_days = days_overdue        days_overdue = (today - assignment.due_date).days    for assignment in pending_assignments:        ).all()        ChoreAssignment.due_date < today        ChoreAssignment.status.in_(["PENDING", "CLAIMED"]),    pending_assignments = await db.query(ChoreAssignment).filter(        today = date.today()    """    Called daily at midnight.    Update overdue days for all pending assignments.    """async def update_overdue_tracking():```pythonThe `total_overdue_days` field on `ChoreAssignment` tracks how many days the assignment has been overdue:### Overdue Tracking- The single overdue instance tracks cumulative overdue days- Overdue chores don't accumulate (no backlog of "vacuum stairs x3")- Only ONE active (non-completed) assignment exists at a time per chore per userPicklesApp maintains a **single instance policy** for overdue chores:### Single Instance Policy## Overdue Handling---```                )                    reward_amount=chore.reward_amount                    status="AVAILABLE",                    due_date=due_date,                    assigned_to_id=None,                    chore_id=chore.id,                await create_assignment(            else:  # FIRST_DIBS                    )                        reward_amount=chore.reward_amount                        status="PENDING",                        due_date=due_date,                        assigned_to_id=user_id,                        chore_id=chore.id,                    await create_assignment(                for user_id in chore.assigned_to_user_ids:            if chore.assignment_type == "ASSIGNED":            # Create assignment(s)                            continue            if existing:            existing = await get_assignment(chore.id, due_date)            # Check for existing assignment        for due_date in occurrences:                )            before_date=end_date            after_date=today,            base_date=base_date,            rule=chore.recurrence_rule,        occurrences = calculate_occurrences(        # Calculate occurrences                    base_date = parse_date(chore.recurrence_rule["start_date"])        else:                base_date = parse_date(chore.recurrence_rule["start_date"])            else:                base_date = last_completion.completed_at.date()            if last_completion:            last_completion = await get_last_completed_assignment(chore.id)        if chore.recurrence_rule["timing_mode"] == "relative":        # Determine base date based on timing mode    for chore in chores:        chores = await get_active_chores()        end_date = today + timedelta(days=days_ahead)    today = date.today()    """    Called hourly by background job.    Generate chore assignments for all active chores.    """async def generate_upcoming_instances(days_ahead: int = 7):```python### Pseudocode```   - Create single assignment with status AVAILABLE4. Handle FIRST_DIBS chores (Phase 3):   - Create one assignment per assigned_to_user_id3. Handle assignment creation for ASSIGNED chores:      - If not, create new ChoreAssignment      - Check if assignment already exists for that date   c. For each occurrence date:   b. Calculate occurrences from base_date for next 7 days      - Relative mode: Use last completion date (or start_date if none)      - Absolute mode: Use chore.start_date   a. Determine base date:2. For each chore:1. Query all active chores in the family```### Algorithm StepsThe instance generation job runs **hourly** and creates `ChoreAssignment` records for the upcoming 7 days. This look-ahead window ensures kids always see their upcoming chores while limiting database growth.### Overview## Instance Generation Algorithm---```}  "start_date": "2026-06-01"  "by_month": [6, 7, 8],  "by_weekday": ["SA"],  "interval": 1,  "frequency": "MONTHLY",{// Summer months only (June-August), every Saturday```json### Seasonal Patterns```}  "start_date": "2026-01-01"  "by_monthday": [1],  "interval": 3,  "frequency": "MONTHLY",{// Every quarter (3 months), 1st day}  "start_date": "2026-01-31"  "by_monthday": [-1],  "interval": 1,  "frequency": "MONTHLY",{// Last day of month}  "start_date": "2026-02-01"  "by_monthday": [1, 15],  "interval": 1,  "frequency": "MONTHLY",{// 1st and 15th (semi-monthly)}  "start_date": "2026-02-01"  "by_monthday": [1],  "interval": 1,  "frequency": "MONTHLY",{// 1st of every month```json### Monthly Patterns```}  "start_date": "2026-02-01"  "by_weekday": ["SA", "SU"],  "interval": 1,  "frequency": "WEEKLY",{// Weekend only}  "start_date": "2026-02-07"  "by_weekday": ["SA"],  "interval": 2,  "frequency": "WEEKLY",{// Every other week on Saturday}  "start_date": "2026-02-02"  "by_weekday": ["MO", "WE", "FR"],  "interval": 1,  "frequency": "WEEKLY",{// Monday, Wednesday, Friday}  "start_date": "2026-02-02"  "by_weekday": ["MO"],  "interval": 1,  "frequency": "WEEKLY",{// Every Monday```json### Weekly Patterns```}  "start_date": "2026-02-02"  "by_weekday": ["MO", "TU", "WE", "TH", "FR"],  "interval": 1,  "frequency": "WEEKLY",{// Weekdays only (Mon-Fri){"frequency": "DAILY", "interval": 3, "start_date": "2026-02-01"}// Every 3 days{"frequency": "DAILY", "interval": 1, "start_date": "2026-02-01"}// Every day```json### Daily Patterns## Common Patterns---```}  "start_date": "2026-02-01"  "timing_mode": "relative",  "interval": 2,  "frequency": "WEEKLY",{```json**Example:**```Result: Next chore due Feb 19 (2 weeks from Feb 5)Scenario: Kid completes Feb 1 chore on Feb 5 (late)Timeline initially: Feb 1, Feb 15, Mar 1...Schedule: Every 2 weeks starting Feb 1```**Behavior:**- Chores where consistent intervals between completions matter- Flexible cleaning schedules- Maintenance tasks (vacuum 2 weeks after last vacuuming)**Use Cases:**In relative mode, the next chore instance is generated relative to when the previous instance was **completed**, not when it was due.### Relative Mode```}  "start_date": "2026-02-01"  "timing_mode": "absolute",  "by_weekday": ["MO"],  "interval": 1,  "frequency": "WEEKLY",{```json**Example:**```Result: Next chore still due Feb 8Scenario: Kid completes Feb 1 chore on Feb 5 (late)Timeline: Feb 1, Feb 8, Feb 15, Feb 22, Mar 1...Schedule: Every Monday starting Feb 1```**Behavior:**- Chores that must happen on schedule regardless of completion- Calendar-tied routines (clean before guests arrive Saturday)- Time-sensitive chores (trash day is always Thursday)**Use Cases:**In absolute mode, chore instances are generated at fixed intervals from the start date, regardless of when the chore is completed.### Absolute Mode## Timing Modes---| `timing_mode` | String | "absolute" or "relative" | "absolute" ||-------|------|-------------|---------|| Field | Type | Description | Default |### PicklesApp Extensions| `count` | Integer | Maximum number of occurrences | No || `end_date` | Date | Last possible occurrence (null = infinite) | No || `start_date` | Date | First possible occurrence | Yes || `by_month` | Array[Integer] | Month(s) of year (1-12) | No || `by_monthday` | Array[Integer] | Day(s) of month (1-31, -1 for last day) | No || `by_weekday` | Array[String] | Weekday codes: MO, TU, WE, TH, FR, SA, SU | No || `interval` | Integer | Every N frequency units (default: 1) | No || `frequency` | String | DAILY, WEEKLY, MONTHLY, YEARLY | Yes ||-------|------|-------------|----------|| Field | Type | Description | Required |### Base Fields (RFC 5545)## Recurrence Rule Format---The chore scheduling system is the core engine that powers recurring chore assignments in PicklesApp. It uses a modified RFC 5545 iCalendar RRULE format with custom extensions to support both absolute and relative timing modes.## Overview---> Technical documentation for the PicklesApp chore scheduling engine**Family Chore & Allowance Management Platform**

*"Your parenting, your way"*

---

## Overview

This repository contains comprehensive documentation for PicklesApp, a family financial management platform that combines chore management, allowance automation, and financial education for children.

## Quick Links

- **[Backend Repository](https://github.com/threnjen/chore-app-backend)** - FastAPI backend service
- **[Frontend Repository](https://github.com/threnjen/chore-app-frontend)** - React PWA frontend

## Documentation Structure

### 📘 Backend Documentation

Located in `docs/backend/`:

| Document | Description |
|----------|-------------|
| **[Database Schema](docs/backend/DATABASE_SCHEMA.md)** | Complete database structure with DDL, indexes, and Row-Level Security policies |
| **[Local Development Setup](docs/backend/LOCAL_DEVELOPMENT_SETUP.md)** | PostgreSQL installation, Python environment, and backend development setup |
| **[Testing Strategy](docs/backend/TESTING.md)** | Backend testing with pytest, factories, integration tests, and load testing |

### 🎨 Frontend Documentation

Located in `docs/frontend/`:

| Document | Description |
|----------|-------------|
| **[Component Architecture](docs/frontend/COMPONENT_ARCHITECTURE.md)** | Component organization, patterns, composition strategies, and code splitting |
| **[State Management](docs/frontend/STATE_MANAGEMENT.md)** | React Context architecture, custom hooks, and API integration patterns |
| **[Age-Adaptive UI Guide](docs/frontend/AGE_ADAPTIVE_UI_GUIDE.md)** | Implementing age-adaptive interfaces for three age brackets (5-9, 10-13, 14-17) |
| **[PWA Implementation](docs/frontend/PWA_IMPLEMENTATION.md)** | Service worker setup, offline support, background sync, and push notifications |
| **[Styling Guide](docs/frontend/STYLING_GUIDE.md)** | Tailwind CSS conventions, design system, and responsive patterns |
| **[Local Development Setup](docs/frontend/LOCAL_DEVELOPMENT_SETUP.md)** | Node/npm installation, Vite configuration, and frontend development setup |
| **[Testing Strategy](docs/frontend/TESTING.md)** | Component testing with Vitest/RTL and E2E testing with Playwright |

### 🔗 Shared Documentation

Located in `docs/shared/`:

| Document | Description |
|----------|-------------|
| **[API Specifications](docs/shared/API_SPECIFICATIONS.md)** | Complete REST API reference with all endpoints, request/response examples |
| **[Development Plan](docs/shared/DEVELOPMENT_PLAN.md)** | Comprehensive project plan with architecture, phases, and deployment strategy |

### 📋 Phase Documentation

Located in `docs/phases/`:

| Phase | Document | Status |
|-------|----------|--------|
| Phase 1 | **[Detailed Plan](docs/phases/PHASE_1_DETAILED.md)** ・ **[Testing](docs/phases/PHASE_1_TESTING.md)** | ✅ Complete |
| Phase 2 | **[Detailed Plan](docs/phases/PHASE_2_DETAILED.md)** ・ **[Testing](docs/phases/PHASE_2_TESTING.md)** | 🔄 In Progress |

## Project Features

### Core Capabilities

- **Advanced Chore Scheduling**: RFC 5545 RRULE-based recurrence patterns with custom intervals
- **Multi-Household Support**: Complete data isolation for children in divorced/separated families
- **Age-Adaptive UI**: Interface automatically adapts based on child's age (Young 5-9, Tween 10-13, Teen 14-17)
- **Financial Transaction System**: ACID-compliant transactions with balance tracking and audit trails
- **Automated Allowances**: Scheduled payments with split distribution across multiple accounts
- **Progressive Web App**: Installable, offline-capable mobile experience
- **OAuth 2.0 Authentication**: Google, Apple, and Facebook social login support

### Technology Stack

**Backend**
- Python 3.11+ with FastAPI
- PostgreSQL 15+ with SQLAlchemy 2.0+
- APScheduler for background jobs
- JWT authentication
- Docker containerization

**Frontend**
- React 18+ with Vite 5+
- Tailwind CSS 3+ with Headless UI
- React Router v6
- React Context + Hooks for state
- PWA with offline support

## Getting Started

### For Backend Developers

1. Review [Database Schema](docs/backend/DATABASE_SCHEMA.md) to understand data models
2. Follow [Backend Setup Guide](docs/backend/LOCAL_DEVELOPMENT_SETUP.md) to install PostgreSQL and Python dependencies
3. Check [API Specifications](docs/shared/API_SPECIFICATIONS.md) for endpoint details
4. Read [Testing Strategy](docs/backend/TESTING.md) for writing tests
5. Clone [chore-app-backend](https://github.com/threnjen/chore-app-backend) and start coding

### For Frontend Developers

1. Review [Component Architecture](docs/frontend/COMPONENT_ARCHITECTURE.md) for component organization
2. Follow [Frontend Setup Guide](docs/frontend/LOCAL_DEVELOPMENT_SETUP.md) to install Node and dependencies
3. Read [Age-Adaptive UI Guide](docs/frontend/AGE_ADAPTIVE_UI_GUIDE.md) for UI implementation patterns
4. Check [State Management](docs/frontend/STATE_MANAGEMENT.md) for Context and hooks usage
5. Review [API Specifications](docs/shared/API_SPECIFICATIONS.md) for backend integration
6. Clone [chore-app-frontend](https://github.com/threnjen/chore-app-frontend) and start building

### For Project Managers / Product

1. Start with [Development Plan](docs/shared/DEVELOPMENT_PLAN.md) for full project overview
2. Review architecture diagrams and phase breakdown
3. Understand multi-household architecture and age-adaptive UI system
4. Check deployment strategy and cost management sections

## Architecture Highlights

### Multi-Tenant Architecture
- Family-based multi-tenancy with complete data isolation
- Middleware-based family scoping for API requests
- PostgreSQL Row-Level Security as safety net
- Support for children belonging to multiple families

### Chore Scheduling Engine
- RFC 5545 iCalendar RRULE-based recurrence patterns
- Custom patterns: M-W-F, bi-weekly, monthly, etc.
- Absolute vs relative timing modes
- Single-instance overdue handling (no backlog accumulation)

### Age-Adaptive UI System
- **Young (5-9)**: Large icons, emoji navigation, simplified interface
- **Tween (10-13)**: Balanced text/icons, educational focus, moderate features
- **Teen (14-17)**: Full feature set, compact layout, advanced financial tools

### Financial Transaction System
- ACID-compliant transactions with automatic balance updates
- Multiple account types: SPENDING, SAVING, GIVING, INVESTMENT, LOAN
- Automated allowance processing with split distributions
- Complete audit trail with balance snapshots

## Development Phases

| Phase | Description | Duration | Status |
|-------|-------------|----------|--------|
| **Phase 1** | Foundation + Email/Password Auth | 2-3 weeks | ✅ Complete |
| **Phase 2** | Chores MVP | 2-3 weeks | 🔄 In Progress |
| **Phase 3** | Enhanced Chores (First-dibs, Templates) | 2 weeks | ⏳ Planned |
| **Phase 4** | Allowances & Automation | 2 weeks | ⏳ Planned |
| **Phase 5** | Infrastructure & Deployment | 2-3 weeks | ⏳ Planned |
| **Phase 6** | Goals, OAuth & Advanced Features | 2-3 weeks | ⏳ Planned |
| **Phase 7** | Age-Adaptive UI & Responsive Design | 2 weeks | ⏳ Planned |
| **Phase 8** | Polish & Performance | 1-2 weeks | ⏳ Planned |
| **Phase 9** | Production Launch | 1-2 weeks | ⏳ Planned |

**Total Timeline**: ~18-22 weeks to production-ready

## Deployment Architecture

### POC (Proof of Concept)
- AWS Lightsail Containers ($10/mo micro instance)
- Lightsail Managed Database (PostgreSQL)
- Lightsail CDN + S3 for static assets
- **Cost**: $25-30/month

### Production Scale
- AWS ECS Fargate with auto-scaling
- AWS RDS Multi-AZ PostgreSQL
- CloudFront CDN + S3
- SES for email, CloudWatch monitoring
- **Cost**: $300-800/month (scales with usage)

## Business Model

**Freemium Approach**:
- **Free Tier**: Single family, basic chores, manual allowances, 2 children
- **Premium ($5/month)**: Multi-household, advanced scheduling, automated allowances, unlimited children, savings goals, budgets, priority support

## Contributing

This project follows the coding guidelines in [chore-app-backend/AGENTS.md](https://github.com/threnjen/chore-app-backend/blob/main/AGENTS.md):
- Use public and trusted libraries
- Prefer caching strategies with `lru_cache`
- Prioritize object-oriented programming
- Always use type hints and docstrings
- Use Pydantic for data modeling

## Repository Links

| Repository | Description | Link |
|------------|-------------|------|
| **chore-app-backend** | FastAPI backend service | [View Repo](https://github.com/threnjen/chore-app-backend) |
| **chore-app-frontend** | React PWA frontend | [View Repo](https://github.com/threnjen/chore-app-frontend) |
| **chore-app-docs** | Documentation (this repo) | [View Repo](https://github.com/threnjen/chore-app-docs) |

## License

TBD

## Contact

For questions or contributions, please open an issue in the relevant repository.

---

**Project Status**: Phase 1 complete, Phase 2 (Chores MVP) in progress.

**Last Updated**: January 31, 2026