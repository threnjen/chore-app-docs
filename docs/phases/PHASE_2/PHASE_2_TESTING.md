# Phase 2 Testing Guide

> Complete testing documentation for Phase 2: Chores MVP

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
| Recurrence algorithm | 100% | Critical |
| Assignment workflow | 100% | Critical |
| Approval/rejection flow | 100% | Critical |
| Reward crediting | 100% | Critical |
| Penalty system | 100% | Critical |
| Expiration workflow | 100% | Critical |
| Chore CRUD operations | 80% | High |
| Background jobs | 80% | High |
| Multi-tenant isolation | 100% | Critical |
| Frontend chore flows | 80% | High |
| Frontend components | 60% | Medium |

### Test Types

- **Unit Tests**: Recurrence algorithm, service methods
- **Integration Tests**: API endpoint testing with database
- **Component Tests**: React component testing with mocks
- **E2E Tests**: Full chore workflow testing with Playwright

---

## Backend Test Scenarios

### Recurrence Algorithm (100% Coverage Required)

#### Daily Patterns

| ID | Scenario | Input Rule | Expected Occurrences |
|----|----------|------------|---------------------|
| REC-01 | Every day | `{"frequency": "DAILY", "interval": 1, "start_date": "2026-02-01"}` | Feb 1, 2, 3, 4, 5... |
| REC-02 | Every 3 days | `{"frequency": "DAILY", "interval": 3, "start_date": "2026-02-01"}` | Feb 1, 4, 7, 10, 13... |
| REC-03 | Weekdays only | `{"frequency": "WEEKLY", "by_weekday": ["MO","TU","WE","TH","FR"], "start_date": "2026-02-02"}` | Feb 2(Mon), 3, 4, 5, 6, 9, 10... |

#### Weekly Patterns

| ID | Scenario | Input Rule | Expected Occurrences |
|----|----------|------------|---------------------|
| REC-04 | Every Monday | `{"frequency": "WEEKLY", "interval": 1, "by_weekday": ["MO"], "start_date": "2026-02-02"}` | Feb 2, 9, 16, 23... |
| REC-05 | M-W-F pattern | `{"frequency": "WEEKLY", "interval": 1, "by_weekday": ["MO","WE","FR"], "start_date": "2026-02-02"}` | Feb 2, 4, 6, 9, 11, 13... |
| REC-06 | Bi-weekly Sunday | `{"frequency": "WEEKLY", "interval": 2, "by_weekday": ["SU"], "start_date": "2026-02-01"}` | Feb 1, 15, Mar 1... |
| REC-07 | Every 3 weeks Tuesday | `{"frequency": "WEEKLY", "interval": 3, "by_weekday": ["TU"], "start_date": "2026-02-03"}` | Feb 3, 24, Mar 17... |
| REC-08 | Weekend only | `{"frequency": "WEEKLY", "interval": 1, "by_weekday": ["SA","SU"], "start_date": "2026-02-01"}` | Feb 1, 7, 8, 14, 15... |

#### Monthly Patterns

| ID | Scenario | Input Rule | Expected Occurrences |
|----|----------|------------|---------------------|
| REC-09 | 1st of month | `{"frequency": "MONTHLY", "interval": 1, "by_monthday": [1], "start_date": "2026-02-01"}` | Feb 1, Mar 1, Apr 1... |
| REC-10 | 1st and 15th | `{"frequency": "MONTHLY", "interval": 1, "by_monthday": [1, 15], "start_date": "2026-02-01"}` | Feb 1, 15, Mar 1, 15... |
| REC-11 | Last day of month | `{"frequency": "MONTHLY", "interval": 1, "by_monthday": [-1], "start_date": "2026-02-28"}` | Feb 28, Mar 31, Apr 30... |
| REC-12 | 31st (short months) | `{"frequency": "MONTHLY", "interval": 1, "by_monthday": [31], "start_date": "2026-01-31"}` | Jan 31, Mar 31, May 31... (skips short months) |
| REC-13 | Every other month 15th | `{"frequency": "MONTHLY", "interval": 2, "by_monthday": [15], "start_date": "2026-02-15"}` | Feb 15, Apr 15, Jun 15... |

#### Edge Cases

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| REC-14 | Leap year Feb 29 | Start Feb 29, 2024 (leap year), check 2025 | Skips to Feb 28, 2025 or Mar 1 |
| REC-15 | DST transition (spring) | Daily chore crossing Mar DST | Times remain consistent |
| REC-16 | DST transition (fall) | Daily chore crossing Nov DST | Times remain consistent |
| REC-17 | End date respected | Rule with end_date | No occurrences after end_date |
| REC-18 | Count limit respected | Rule with count: 5 | Exactly 5 occurrences |
| REC-19 | Start date in past | Start date before today | First occurrence >= today |

#### Timing Modes

| ID | Scenario | Mode | Expected Behavior |
|----|----------|------|-------------------|
| REC-20 | Absolute mode - on time | absolute | Next occurrence per schedule |
| REC-21 | Absolute mode - completed late | absolute | Next occurrence still per original schedule |
| REC-22 | Relative mode - on time | relative | Next occurrence = completion + interval |
| REC-23 | Relative mode - completed late | relative | Next occurrence = late completion + interval |
| REC-24 | Relative mode - no completion | relative | Uses start_date as base |

---

### Chore CRUD (80% Coverage Required)

#### Create Chore

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| CHR-01 | Create valid chore | Valid title, reward, recurrence | 201, chore created |
| CHR-02 | Create without title | Missing title | 422, validation error |
| CHR-03 | Create with empty title | `{title: ""}` | 422, validation error |
| CHR-04 | Create with negative reward | `{reward_amount: -5.00}` | 422, validation error |
| CHR-05 | Create with invalid frequency | `{recurrence_rule: {frequency: "HOURLY"}}` | 422, validation error |
| CHR-06 | Create with invalid weekdays | `{by_weekday: ["XX"]}` | 422, validation error |
| CHR-07 | Create assigned chore | `{assignment_type: "ASSIGNED", assigned_to_user_ids: [...]}` | 201, assignments created |
| CHR-08 | Create with all options | Full payload | 201, all fields saved |
| CHR-09 | Create generates initial instances | Valid chore | Assignments created for next 7 days |

#### Read Chore

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| CHR-10 | List family chores | Authenticated + X-Family-Id | 200, array of chores |
| CHR-11 | List with active filter | `?is_active=true` | Only active chores |
| CHR-12 | Get chore by ID | Valid chore ID | 200, chore details |
| CHR-13 | Get non-existent chore | Random UUID | 404 |
| CHR-14 | Get other family's chore | Chore from different family | 404 |
| CHR-15 | Preview occurrences | `/chores/{id}/preview?count=5` | 200, next 5 dates |

#### Update Chore

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| CHR-16 | Update title | `{title: "New Title"}` | 200, title updated |
| CHR-17 | Update reward | `{reward_amount: 10.00}` | 200, future assignments updated |
| CHR-18 | Deactivate chore | `{is_active: false}` | 200, future assignments cancelled |
| CHR-19 | Update as child | Child token | 403, forbidden |
| CHR-20 | Update other family's chore | Wrong family | 404 |

#### Delete Chore

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| CHR-21 | Delete chore (soft) | Valid chore ID | 204, is_active = false |
| CHR-22 | Delete cancels future assignments | Valid chore ID | Future assignments marked cancelled |
| CHR-23 | Delete preserves history | Valid chore ID | Past assignments remain |
| CHR-24 | Delete as child | Child token | 403, forbidden |

---

### Chore Assignment Workflow (100% Coverage Required)

#### Assignment Generation

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| ASN-01 | Generate for daily chore | Daily recurrence | 7 assignments for next week |
| ASN-02 | Generate for weekly chore | Weekly recurrence | 1-2 assignments |
| ASN-03 | No duplicate generation | Run generator twice | Same number of assignments |
| ASN-04 | Respects timing mode | relative mode, no completion | Uses start_date |
| ASN-05 | Relative mode with completion | relative mode, has completion | Uses completion date |

#### Complete Assignment

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| ASN-06 | Complete pending assignment | PENDING assignment | 200, status = COMPLETED |
| ASN-07 | Complete with photos | `{photo_urls: ["url1", "url2"]}` | 200, photos saved |
| ASN-08 | Complete already completed | COMPLETED assignment | 400, already completed |
| ASN-09 | Complete approved assignment | APPROVED assignment | 400, already processed |
| ASN-10 | Complete rejected assignment | REJECTED assignment | 200, status = COMPLETED (retry) |
| ASN-11 | Complete as wrong user | Other child's assignment | 403, forbidden |
| ASN-12 | Complete sets completed_at | Any valid | completed_at timestamp set |
| ASN-13 | Complete sets completed_by | Any valid | completed_by_id set to current user |

#### Approve Assignment

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| ASN-14 | Approve completed assignment | COMPLETED assignment, parent token | 200, status = APPROVED |
| ASN-15 | Approve credits reward | Chore with $5 reward | Transaction created, balance +$5 |
| ASN-16 | Approve zero reward chore | No reward | 200, no transaction created |
| ASN-17 | Approve non-monetary reward | points/stars reward | 200, no transaction (Phase 3 feature) |
| ASN-18 | Approve as child | Child token | 403, forbidden |
| ASN-19 | Approve pending assignment | PENDING (not completed) | 400, not completed |
| ASN-20 | Approve already approved | APPROVED | 400, already approved |
| ASN-21 | Approve sets approved_at | Any valid | approved_at timestamp set |
| ASN-22 | Approve sets approved_by | Any valid | approved_by_id set |
| ASN-23 | Approval transaction has reference | Any valid | reference_id = assignment.id |

#### Reject Assignment

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| ASN-24 | Reject completed assignment | COMPLETED + reason | 200, status = REJECTED |
| ASN-25 | Reject without reason | No reason | 422, reason required |
| ASN-26 | Reject empty reason | `{reason: ""}` | 422, reason required |
| ASN-27 | Reject as child | Child token | 403, forbidden |
| ASN-28 | Reject pending assignment | PENDING | 400, not completed |
| ASN-29 | Reject sets rejection_reason | Valid reason | rejection_reason saved |
| ASN-30 | Rejected can be re-completed | REJECTED assignment | Complete works again |

#### Batch Operations

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| ASN-31 | Batch approve multiple | 3 valid assignment IDs | 200, all approved |
| ASN-32 | Batch approve partial failure | 2 valid, 1 invalid | 207, partial success |
| ASN-33 | Batch approve empty list | `[]` | 400, at least one required |
| ASN-34 | Batch approve as child | Child token | 403, forbidden |

---

### Reward Crediting (100% Coverage Required)

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| RWD-01 | Credit to default spending account | Approval with reward | Balance increases |
| RWD-02 | Transaction category is CHORE_REWARD | Any approval | category = CHORE_REWARD |
| RWD-03 | Transaction description includes title | Chore titled "Clean room" | "Reward for: Clean room" |
| RWD-04 | Transaction reference links assignment | Any approval | reference_id = assignment.id |
| RWD-05 | Decimal precision maintained | Reward = $5.75 | Balance +$5.75 exactly |
| RWD-06 | Atomicity on failure | DB error during credit | Both assignment and transaction rolled back |
| RWD-07 | No account exists | Child has no account | 400, no default account |
| RWD-08 | Credit on complete (instant) | credit_on_complete = True | Balance credited on completion, not approval |
| RWD-09 | Credit on approval (default) | credit_on_complete = False | Balance credited only after approval |
| RWD-10 | No duplicate credit | Approve with credit_on_complete=True | Reward only credited once |

---

### Tiered Rewards (100% Coverage Required)

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| TRD-01 | FULL tier on-time completion | Complete before due_datetime | reward_tier = FULL, effective_reward = reward_amount |
| TRD-02 | FULL tier no grace configured | grace_period_days = null, complete late | reward_tier = FULL (any completion is on-time) |
| TRD-03 | REDUCED tier within grace | Complete within grace_period | reward_tier = REDUCED |
| TRD-04 | REDUCED tier calculation | reward=10, late_reward_percentage=50 | effective_reward = 5.00 |
| TRD-05 | BONUS tier early completion | Complete 24h before deadline | reward_tier = BONUS |
| TRD-06 | BONUS tier calculation | reward=5, early_bonus=2 | effective_reward = 7.00 |
| TRD-07 | Early threshold not met | Complete 12h early, threshold=24h | reward_tier = FULL (not early enough) |
| TRD-08 | completed_on_time flag set | On-time completion | completed_on_time = True |
| TRD-09 | completed_on_time false | Late completion | completed_on_time = False |
| TRD-10 | effective_reward stored | Any completion | assignment.effective_reward populated |
| TRD-11 | Transaction uses effective_reward | Approval | Transaction amount = effective_reward |

### Grace Period Calculation (100% Coverage Required)

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| GRC-01 | Grace period days only | grace_period_days=2, complete 1 day late | Within grace, REDUCED tier |
| GRC-02 | Grace period hours only | grace_period_hours=12, complete 6h late | Within grace, REDUCED tier |
| GRC-03 | Grace period combined | days=1, hours=12, complete 36h late | Within grace (1.5 days < 1d+12h) |
| GRC-04 | Beyond grace period | days=1, hours=0, complete 2 days late | Past grace, REDUCED tier + late_penalty |
| GRC-05 | Zero grace strict deadline | days=0, hours=0, complete 1min late | Past grace immediately |
| GRC-06 | Grace with due_time | due_time=17:00, grace=2h, complete 18:30 | Within grace |
| GRC-07 | Grace respects timezone | Family timezone = PST | Calculation uses family timezone |

### Late Penalty (100% Coverage Required)

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| LTP-01 | Late penalty on completion | late_penalty_amount=2, complete late | Balance debited $2 |
| LTP-02 | No late penalty on-time | late_penalty_amount=2, complete on-time | No penalty |
| LTP-03 | Late penalty description | Complete late | "Late completion penalty for: {title}" |
| LTP-04 | Late penalty mutually exclusive | Complete late (penalty applied) | Expiration penalty NOT applied |
| LTP-05 | Late penalty + reduced reward | Both configured | Reward reduced AND penalty applied |
| LTP-06 | Late penalty flag set | Late penalty applied | penalty_applied = True |

### Escalating Penalties (100% Coverage Required)

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| ESC-01 | Base penalty only | penalty_per_overdue_day=null, expire | effective_penalty = penalty_amount |
| ESC-02 | Escalation 1 day | base=1, per_day=0.50, overdue=1 | effective_penalty = 1.50 |
| ESC-03 | Escalation 5 days | base=1, per_day=0.50, overdue=5 | effective_penalty = 3.50 |
| ESC-04 | Escalation with cap | base=1, per_day=0.50, max=2.00, overdue=10 | effective_penalty = 2.00 (capped) |
| ESC-05 | No cap configured | max_penalty_amount=null, overdue=30 | Penalty grows unbounded |
| ESC-06 | Escalation formula correct | base + (per_day × overdue_days) | Formula applied correctly |
| ESC-07 | effective_penalty stored | Expiration with escalation | assignment.effective_penalty populated |
| ESC-08 | Transaction uses effective_penalty | Expiration | Transaction amount = effective_penalty |

---

### Penalty System (100% Coverage Required)

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| PEN-01 | Penalty on rejection | penalty_behavior = ON_REJECTION, reject | Balance decreases by penalty_amount |
| PEN-02 | No penalty on rejection | penalty_behavior = ON_EXPIRATION, reject | No balance change |
| PEN-03 | Penalty on expiration | penalty_behavior = ON_EXPIRATION, expire | Balance decreases |
| PEN-04 | No penalty on expiration | penalty_behavior = ON_REJECTION, expire | No balance change |
| PEN-05 | Penalty on both | penalty_behavior = ON_BOTH, reject | Balance decreases |
| PEN-06 | Penalty on both (expiration) | penalty_behavior = ON_BOTH, expire | Balance decreases |
| PEN-07 | No penalty configured | penalty_behavior = NONE, reject | No balance change |
| PEN-08 | Penalty transaction category | Any penalty | category = CHORE_PENALTY |
| PEN-09 | Penalty description includes title | Chore titled "Clean room" | "Penalty for: Clean room" |
| PEN-10 | Penalty reference links assignment | Any penalty | reference_id = assignment.id |
| PEN-11 | Decimal precision maintained | Penalty = $2.50 | Balance -$2.50 exactly |
| PEN-12 | Penalty atomicity on failure | DB error during penalty | Both assignment and transaction rolled back |
| PEN-13 | No account exists | Child has no account | 400, no default account |
| PEN-14 | penalty_applied flag set | Penalty applied | assignment.penalty_applied = True |
| PEN-15 | Skip penalty if already applied | penalty_applied = True | No duplicate penalty |
| PEN-16 | Override penalty on reject | apply_penalty = False in request | No penalty despite config |
| PEN-17 | Balance can go negative | Penalty > balance | Balance becomes negative |

---

### Expiration Workflow

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| EXP-01 | Auto-expire after expiration_days | expiration_days = 3, overdue 4 days | status = EXPIRED |
| EXP-02 | No auto-expire without config | expiration_days = None, overdue 30 days | status = PENDING |
| EXP-03 | Expire job runs | Scheduled trigger | Eligible assignments expired |
| EXP-04 | expired_at timestamp set | Assignment expires | expired_at = now() |
| EXP-05 | Late completion allowed | allow_late_completion = True | Can complete overdue |
| EXP-06 | Late completion blocked | allow_late_completion = False, overdue | 400, cannot complete |
| EXP-07 | Cannot complete expired | status = EXPIRED | 400, assignment expired |
| EXP-08 | Penalty applied on expiration | penalty_behavior = ON_EXPIRATION | Balance debited |

---

### Overdue Tracking

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| OVD-01 | Assignment due today | due_date = today | total_overdue_days = 0 |
| OVD-02 | Assignment 1 day overdue | due_date = yesterday | total_overdue_days = 1 |
| OVD-03 | Assignment 5 days overdue | due_date = 5 days ago | total_overdue_days = 5 |
| OVD-04 | Overdue tracking job updates | Job runs | All pending assignments updated |
| OVD-05 | Completed assignments not updated | COMPLETED status | total_overdue_days frozen |
| OVD-06 | Approved assignments not updated | APPROVED status | total_overdue_days frozen |

---

### Multi-Tenant Isolation (100% Coverage Required)

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| MT-01 | Create chore without X-Family-Id | No header | 400, header required |
| MT-02 | Create chore for non-member family | Valid UUID, not member | 403 |
| MT-03 | List chores only shows family's | Authenticated | Only current family chores |
| MT-04 | Cannot access other family's chore | Other family's chore ID | 404 |
| MT-05 | Cannot approve other family's assignment | Other family's assignment | 404 |
| MT-06 | Parent sees all family assignments | Parent token | All family assignments visible |
| MT-07 | Child sees only own assignments | Child token | Only assigned-to-self visible |

---

### Background Jobs

| ID | Scenario | Input | Expected Result |
|----|----------|-------|-----------------|
| JOB-01 | Instance generation job runs | Hourly trigger | New assignments created |
| JOB-02 | Overdue tracking job runs | Daily trigger | Overdue days updated |
| JOB-03 | Auto-approve job runs | Hourly trigger | Eligible chores approved |
| JOB-04 | Auto-approve respects threshold | auto_approve_after = 24 | Only 24+ hour old completions |
| JOB-05 | Job handles errors gracefully | Exception in job | Scheduler continues |
| JOB-06 | Jobs persist across restart | Restart app | Jobs still scheduled |
| JOB-07 | Expire assignments job runs | Hourly trigger | Overdue assignments expired |
| JOB-08 | Expire job applies penalties | penalty_behavior configured | Penalties applied on expiration |

---

## Frontend Test Scenarios

### Unit Tests (Vitest)

#### ChoreService

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| FS-01 | getChores success | Call getChores() | Returns chore array |
| FS-02 | getChores with filters | Call getChores({is_active: true}) | Passes query params |
| FS-03 | createChore success | Call createChore(data) | Returns created chore |
| FS-04 | completeAssignment success | Call completeAssignment(id) | Returns updated assignment |
| FS-05 | approveAssignment success | Call approveAssignment(id) | Returns approved assignment |
| FS-06 | rejectAssignment success | Call rejectAssignment(id, reason) | Returns rejected assignment |

### Component Tests

#### ChoreForm

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| CP-01 | Empty form validation | Submit empty | Title error displayed |
| CP-02 | Valid form submission | Fill and submit | onSubmit called with data |
| CP-03 | Recurrence builder renders | Open form | Frequency options visible |
| CP-04 | Weekday picker shows for weekly | Select WEEKLY | Weekday checkboxes appear |
| CP-05 | Preview updates on rule change | Change recurrence | Preview dates update |

#### ChoreAssignmentCard

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| CP-06 | Displays chore title | Render card | Title visible |
| CP-07 | Shows overdue badge | overdue_days > 0 | Badge with "X days overdue" |
| CP-08 | Complete button works | Click complete | onComplete callback called |
| CP-09 | Disabled when not due | Future due_date | Button disabled |

#### RecurrenceBuilder

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| CP-10 | Default to weekly | Initial render | WEEKLY selected |
| CP-11 | Weekday picker interaction | Click "MO" | MO toggled in by_weekday |
| CP-12 | Interval validation | Enter 0 | Error shown |
| CP-13 | Start date required | Clear start date | Error shown |
| CP-14 | Timing mode toggle | Select relative | timing_mode = "relative" |

#### ChoreForm Simple/Advanced Mode

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| CP-SA-01 | Default to Simple mode | Open form | Only reward/penalty/behavior visible |
| CP-SA-02 | Toggle to Advanced | Click Advanced toggle | Grace period fields appear |
| CP-SA-03 | Advanced shows all fields | In Advanced mode | Early bonus, escalation visible |
| CP-SA-04 | Toggle back to Simple | Click Simple toggle | Advanced fields hidden |
| CP-SA-05 | Mode persists in localStorage | Toggle Advanced, refresh | Still in Advanced mode |
| CP-SA-06 | Simple mode nulls advanced fields | Switch to Simple | grace_period_days = null in submission |
| CP-SA-07 | Grace period validation | Enter negative days | Validation error |
| CP-SA-08 | Late percentage 0-100 | Enter 150 | Validation error |
| CP-SA-09 | Early threshold validation | Enter 0 | Validation error (must be >= 1) |
| CP-SA-10 | Max penalty >= base penalty | max < base penalty | Warning displayed |

#### ChoreAssignmentCard Tier Display

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| CP-TD-01 | Show BONUS tier badge | reward_tier = BONUS | "Early Bonus" badge displayed |
| CP-TD-02 | Show REDUCED tier badge | reward_tier = REDUCED | "Late" badge displayed |
| CP-TD-03 | Show effective reward | effective_reward set | Amount shows effective, not base |
| CP-TD-04 | Show penalty applied indicator | penalty_applied = True | Penalty indicator visible |
| CP-TD-05 | Show effective penalty amount | effective_penalty set | Shows escalated amount |

#### PendingApprovalsPage

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| CP-15 | Lists pending approvals | Render page | Approval cards displayed |
| CP-16 | Empty state | No pending | "No pending approvals" message |
| CP-17 | Approve button works | Click approve | API called, card removed |
| CP-18 | Reject opens modal | Click reject | Modal opens |
| CP-19 | Reject requires reason | Submit empty reason | Validation error |
| CP-20 | Batch approve button | Click approve all | All approved |

#### PendingApprovalsPage Tier Info

| ID | Scenario | Action | Expected Result |
|----|----------|--------|-----------------|
| CP-PA-01 | Show tier in approval card | REDUCED tier | "Late completion (50%)" displayed |
| CP-PA-02 | Show effective reward in button | effective_reward=2.50 | "Approve ($2.50)" button text |
| CP-PA-03 | Show penalty warning on reject | penalty configured | "This will debit $X" warning |
| CP-PA-04 | Override penalty checkbox | In reject modal | "Apply penalty" checkbox (default checked) |

### E2E Tests (Playwright)

#### Chore Creation Flow

| ID | Scenario | Steps | Expected Result |
|----|----------|-------|-----------------|
| E2E-01 | Create simple daily chore | Login → Chores → Create → Fill → Submit | Chore appears in list |
| E2E-02 | Create M-W-F chore | Create → Select M-W-F → Submit | Preview shows correct dates |
| E2E-03 | Create with reward | Create → Enter $5 → Submit | Reward displayed on chore |
| E2E-04 | Create with deadline time | Create → Set 8 PM → Submit | Time shown on assignments |

#### Chore Completion Flow

| ID | Scenario | Steps | Expected Result |
|----|----------|-------|-----------------|
| E2E-05 | Kid completes chore | Login as kid → My Chores → Complete | Status changes to "Waiting for approval" |
| E2E-06 | Kid sees overdue chore | Login as kid → My Chores | Overdue badge visible |
| E2E-07 | Kid can retry rejected | Login → See rejected → Complete again | Status back to completed |

#### Approval Flow

| ID | Scenario | Steps | Expected Result |
|----|----------|-------|-----------------|
| E2E-08 | Parent approves chore | Login as parent → Approvals → Approve | Toast: "Approved! $5 credited" |
| E2E-09 | Parent rejects chore | Approvals → Reject → Enter reason | Status shows rejected |
| E2E-10 | Approval credits account | Approve → Check kid's account | Balance increased by reward |
| E2E-11 | Batch approve multiple | Select all → Approve all | All approved |

#### Dashboard Integration

| ID | Scenario | Steps | Expected Result |
|----|----------|-------|-----------------|
| E2E-12 | Pending count in nav | Complete chore | Parent sees badge on Approvals |
| E2E-13 | Today's chores on dashboard | Login as kid | Today's chores section visible |

---

## Manual Testing Checklist

### Prerequisites

- [ ] Docker Desktop running
- [ ] Backend started (`docker-compose up`)
- [ ] Database migrated with Phase 2 migration
- [ ] Frontend started (`npm run dev`)
- [ ] Seed data loaded with test families

### Chore Creation (Parent)

- [ ] Can navigate to Chores page from sidebar
- [ ] Can click "New Chore" button
- [ ] Form shows all required fields
- [ ] Can enter chore title and description
- [ ] Can set reward amount
- [ ] Can select child to assign to
- [ ] **Recurrence Builder**:
  - [ ] Can select Daily frequency
  - [ ] Can select Weekly frequency
  - [ ] Can select Monthly frequency
  - [ ] Weekday picker appears for Weekly
  - [ ] Can select multiple weekdays (M-W-F)
  - [ ] Can set interval (every 2 weeks)
  - [ ] Can toggle between Absolute and Relative timing
  - [ ] Preview shows next 5 occurrences correctly
- [ ] Can set deadline time (optional)
- [ ] Can set estimated duration (optional)
- [ ] Can toggle "Require Photo"
- [ ] Can toggle "Require Approval"
- [ ] Can set auto-approve after hours
- [ ] Submit creates chore successfully
- [ ] Chore appears in list
- [ ] Initial assignments generated

### Chore List (Parent)

- [ ] Shows all family chores
- [ ] Shows chore title and reward
- [ ] Shows recurrence pattern description
- [ ] Shows assignment count
- [ ] Can filter by active/inactive
- [ ] Can click to view chore details
- [ ] Can edit chore
- [ ] Can delete chore (with confirmation)

### My Chores (Child)

- [ ] Can navigate to My Chores page
- [ ] Shows "Today" section with due chores
- [ ] Shows "Overdue" section if any
- [ ] Overdue badge shows days count
- [ ] Shows "Upcoming" section
- [ ] Can click "Complete" on due chore
- [ ] Completion confirmation shown
- [ ] Status changes to "Waiting for approval"
- [ ] Completed chores move to different section

### Pending Approvals (Parent)

- [ ] Can navigate to Pending Approvals page
- [ ] Shows list of completed assignments
- [ ] Shows child name and completion time
- [ ] Shows chore title and reward amount
- [ ] Can click Approve button
- [ ] Approval success message shown
- [ ] Transaction created (check transactions)
- [ ] Can click Reject button
- [ ] Reject modal requires reason
- [ ] Rejection updates status
- [ ] Child can see rejection reason
- [ ] "Approve All" button works (if multiple)

### Reward Integration

- [ ] Approval creates transaction
- [ ] Transaction appears in child's account
- [ ] Balance increased by reward amount
- [ ] Transaction shows "Chore Reward" category
- [ ] Transaction description includes chore title

### Edge Cases

- [ ] Child cannot access Chores page (parent only)
- [ ] Child cannot approve/reject
- [ ] Cannot complete future chores
- [ ] Cannot complete already approved chores
- [ ] Rejected chores can be re-completed
- [ ] Deactivating chore cancels future assignments

---

## API Testing with cURL

### Setup

```bash
# Get auth token
TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "parent@test.com", "password": "password123"}' \
  | jq -r '.access_token')

FAMILY_ID="<your-family-uuid>"
```

### Chore Endpoints

```bash
# Create chore
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
      "start_date": "2026-02-01"
    },
    "deadline_time": "20:00:00",
    "require_approval": true
  }'

# List chores
curl -X GET http://localhost:8000/chores \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"

# Get chore details
curl -X GET http://localhost:8000/chores/<chore-id> \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"

# Preview occurrences
curl -X GET "http://localhost:8000/chores/<chore-id>/preview?count=5" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"

# Update chore
curl -X PATCH http://localhost:8000/chores/<chore-id> \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{"reward_amount": "7.50"}'

# Delete chore
curl -X DELETE http://localhost:8000/chores/<chore-id> \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"
```

### Assignment Endpoints

```bash
# Get assignments for child
CHILD_TOKEN=$(curl -s -X POST http://localhost:8000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "child@test.com", "password": "TestPass123"}' \
  | jq -r '.access_token')

# List my assignments
curl -X GET http://localhost:8000/chore-assignments/my \
  -H "Authorization: Bearer $CHILD_TOKEN" \
  -H "X-Family-Id: $FAMILY_ID"

# Complete assignment
curl -X POST http://localhost:8000/chore-assignments/<assignment-id>/complete \
  -H "Authorization: Bearer $CHILD_TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{"photo_urls": []}'

# Get pending approvals (parent)
curl -X GET http://localhost:8000/chore-assignments/pending-approval \
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
  -d '{"reason": "Room still looks messy, please try again"}'

# Batch approve
curl -X POST http://localhost:8000/chore-assignments/batch-approve \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Family-Id: $FAMILY_ID" \
  -H "Content-Type: application/json" \
  -d '{"assignment_ids": ["<id1>", "<id2>", "<id3>"]}'
```

---

## Running Tests

### Backend Tests

```bash
# Enter backend container
docker-compose exec api bash

# Run all tests
pytest

# Run with coverage
pytest --cov=app --cov-report=html

# Run specific test file
pytest tests/test_chores.py -v

# Run specific test
pytest tests/test_chores.py::TestChoreCreate::test_create_valid_chore -v

# Run recurrence tests
pytest tests/test_recurrence.py -v

# Run assignment workflow tests
pytest tests/test_chore_assignments.py -v
```

### Frontend Tests

```bash
# Run unit tests
npm run test

# Run with coverage
npm run test:coverage

# Run specific test file
npm run test -- --testPathPattern=ChoreForm

# Run E2E tests
npm run test:e2e

# Run E2E tests headed (see browser)
npm run test:e2e -- --headed

# Run specific E2E test
npm run test:e2e -- --grep "Create simple daily chore"
```

### Test Database Reset

```bash
# Reset test database
docker-compose exec db psql -U postgres -c "DROP DATABASE IF EXISTS picklesapp_test;"
docker-compose exec db psql -U postgres -c "CREATE DATABASE picklesapp_test;"
docker-compose exec api alembic upgrade head
```

---

## Test Data Setup

### Seed Script Addition

Add to `scripts/seed_dev_data.py`:

```python
async def seed_chores(db: AsyncSession, family: Family, parent: User, children: list[User]):
    """Seed sample chores for testing."""
    
    # Daily chore
    daily_chore = Chore(
        family_id=family.id,
        title="Make bed",
        description="Make your bed before breakfast",
        reward_amount=Decimal("1.00"),
        reward_type="money",
        assignment_type="ASSIGNED",
        recurrence_rule={
            "frequency": "DAILY",
            "interval": 1,
            "timing_mode": "absolute",
            "start_date": date.today().isoformat()
        },
        deadline_time=time(8, 0),
        require_approval=True,
        created_by_id=parent.id
    )
    
    # M-W-F chore
    mwf_chore = Chore(
        family_id=family.id,
        title="Clean room",
        description="Pick up toys, vacuum, dust",
        reward_amount=Decimal("5.00"),
        reward_type="money",
        assignment_type="ASSIGNED",
        recurrence_rule={
            "frequency": "WEEKLY",
            "interval": 1,
            "by_weekday": ["MO", "WE", "FR"],
            "timing_mode": "absolute",
            "start_date": date.today().isoformat()
        },
        deadline_time=time(20, 0),
        require_approval=True,
        created_by_id=parent.id
    )
    
    # Weekly chore
    weekly_chore = Chore(
        family_id=family.id,
        title="Take out trash",
        description="Empty all trash cans into outdoor bin",
        reward_amount=Decimal("3.00"),
        reward_type="money",
        assignment_type="ASSIGNED",
        recurrence_rule={
            "frequency": "WEEKLY",
            "interval": 1,
            "by_weekday": ["TH"],
            "timing_mode": "absolute",
            "start_date": date.today().isoformat()
        },
        require_approval=True,
        created_by_id=parent.id
    )
    
    db.add_all([daily_chore, mwf_chore, weekly_chore])
    await db.commit()
    
    # Generate initial assignments
    service = ChoreService(db)
    await service.generate_upcoming_instances(days_ahead=7)
```

---

## Phase 1 Amendment: Bidirectional Family Building Testing

This section covers testing for the enhanced registration and join request functionality added during Phase 2.

### Manual Testing Flow

#### Prerequisites

1. Backend server running: `uvicorn app.main:app --reload`
2. Frontend dev server running: `npm run dev`
3. Database migrated: `alembic upgrade head`
4. Check DEV_MODE is enabled (emails will be logged, not sent)

---

### Test Scenario 1: Adult Registration (Create New Family)

**Goal**: Verify an adult (18+) can register and automatically get a family created.

**Steps**:

1. Navigate to `http://localhost:5173/register`
2. **Step 1 - Birth Year**:
   - Select birth year: `2000` (adult, 26 years old)
   - Click "Continue"
3. **Step 2 - Basic Info**:
   - Verify badge shows "👨‍👩‍👧 Parent Account"
   - Enter:
     - First name: `Test`
     - Last name: `Parent`
     - Email: `testparent@example.com`
     - Password: `TestPass123`
     - Confirm Password: `TestPass123`
   - Click "Continue"
4. **Step 3 - Family Connection**:
   - Verify "Create a new family" is selected by default
   - Click "Create Account"
5. **Success Screen**:
   - Verify success message appears
   - Verify "Your family has been created!" message
   - Verify "Please check your email to verify" message

**Expected Results**:
- User created with `role=PARENT`, `email_verified=false`
- Family created with name "{LastName} Family"
- User added as family OWNER
- Verification email logged to console
- User redirected to login page

**Backend Verification**:
```bash
# Check user was created
curl -s http://localhost:8000/api/v1/auth/me -H "Authorization: Bearer <token>" | jq

# Check logs for verification email
# Look for: [DEV MODE] Email would be sent: ... Subject: Verify your email address
```

---

### Test Scenario 2: Adult Registration (Join Existing Family)

**Goal**: Verify an adult can request to join an existing family.

**Prerequisites**: Complete Test Scenario 1 first (need existing family)

**Steps**:

1. Navigate to `http://localhost:5173/register`
2. **Step 1 - Birth Year**:
   - Select birth year: `1995` (adult, 31 years old)
   - Click "Continue"
3. **Step 2 - Basic Info**:
   - Enter:
     - First name: `Other`
     - Last name: `Parent`
     - Email: `otherparent@example.com`
     - Password: `TestPass123`
     - Confirm Password: `TestPass123`
   - Click "Continue"
4. **Step 3 - Family Connection**:
   - Select "Join an existing family" radio button
   - Enter parent email: `testparent@example.com`
   - Enter message: `Hi, I'm the other parent!`
   - Click "Send Request"
5. **Success Screen**:
   - Verify "Your request to join has been sent" message

**Expected Results**:
- User created with `role=PARENT`, `email_verified=false`
- Join request created with `status=PENDING`, `invitation_type=JOIN_REQUEST`
- Notification email to target parent logged to console
- User NOT added to family yet

---

### Test Scenario 3: Child Registration (Request to Join Parent)

**Goal**: Verify a child (<18) must provide parent email and creates a join request.

**Prerequisites**: Complete Test Scenario 1 first

**Steps**:

1. Navigate to `http://localhost:5173/register`
2. **Step 1 - Birth Year**:
   - Select birth year: `2015` (child, 11 years old)
   - Click "Continue"
3. **Step 2 - Basic Info**:
   - Verify badge shows "🧒 Child Account"
   - Enter:
     - First name: `Test`
     - Last name: `Child`
     - Email: `testchild@example.com`
     - Password: `TestPass123`
     - Confirm Password: `TestPass123`
   - Click "Continue"
4. **Step 3 - Family Connection**:
   - Verify NO option to create family (only parent email input)
   - Enter parent email: `testparent@example.com`
   - Enter message: `Hi Mom, it's me!`
   - Click "Send Request"
5. **Success Screen**:
   - Verify "We've sent a request to your parent" message

**Expected Results**:
- User created with `role=CHILD`, `email_verified=false`
- Join request created linking to parent's family
- Notification email to parent logged
- Child NOT added to family yet

---

### Test Scenario 4: Email Verification Flow

**Goal**: Verify email verification works and returns tokens.

**Prerequisites**: Complete any registration scenario

**Steps**:

1. Find verification token from console logs:
   ```
   [DEV MODE] Email would be sent:
     To: testparent@example.com
     Subject: Verify your email address
     Body: ...verify-email/ABC123TOKEN...
   ```
2. Extract the token from the URL (e.g., `ABC123TOKEN`)
3. Navigate to `http://localhost:5173/verify-email/ABC123TOKEN`
4. **Verification Page**:
   - Verify "Verifying your email..." spinner
   - Verify "Email Verified!" success message
   - Verify automatic redirect to dashboard

**Expected Results**:
- User's `email_verified` set to `true`
- `email_verification_token` set to `null`
- Access and refresh tokens returned
- User redirected to dashboard

**API Test**:
```bash
# Direct API test
curl -X POST http://localhost:8000/api/v1/auth/verify-email \
  -H "Content-Type: application/json" \
  -d '{"token": "ABC123TOKEN"}'
```

---

### Test Scenario 5: Parent Reviews Join Requests

**Goal**: Verify parent can see and approve/reject join requests.

**Prerequisites**: 
- Complete Test Scenario 1 (parent registered)
- Complete Test Scenario 3 or 4 (child registered with join request)
- Verify parent's email (Test Scenario 4)

**Steps**:

1. Login as parent: `testparent@example.com`
2. Navigate to dashboard
3. **Dashboard Notification**:
   - Verify yellow notification card: "1 pending join request"
   - Click the notification
4. **Join Requests Page** (`/family/join-requests`):
   - Verify pending request shows:
     - Requester name and email
     - Their message
     - Created date
     - Expiration (30 days from creation)
5. **Approve Request**:
   - Click "Approve" button
   - **If child email not verified**: Verify error message about email verification
   - **If child email verified**: Verify success message
6. **Verify Child Added**:
   - Navigate to Family Members page
   - Verify child appears in member list

**Expected Results (Approval)**:
- Join request status changed to `ACCEPTED`
- Child added to family as MEMBER
- Approval email sent to child (logged)

---

### Test Scenario 6: Parent Rejects Join Request

**Goal**: Verify rejection flow.

**Prerequisites**: Create a new join request

**Steps**:

1. Register a new child (different email)
2. Login as parent
3. Go to Join Requests page
4. Click "Reject" on the request
5. Verify success message
6. Verify request removed from list

**Expected Results**:
- Join request status changed to `REJECTED`
- Rejection email sent to requester (logged)
- Child can submit a new request if desired

---

### Test Scenario 7: Child with Unregistered Parent Email

**Goal**: Verify registration invitation sent when parent email doesn't exist.

**Steps**:

1. Navigate to register page
2. Complete registration as child (birth year 2014)
3. Enter parent email that doesn't exist: `newparent@example.com`
4. Submit registration

**Expected Results**:
- Child account created
- Registration invitation email sent to `newparent@example.com` (logged)
- Join request created with `family_id=null` (no family yet)
- Message indicates parent will be invited

**Console Log Check**:
```
[DEV MODE] Email would be sent:
  To: newparent@example.com
  Subject: Test Child wants you to join their family app!
```

---

### Test Scenario 8: Resend Verification Email

**Goal**: Verify resend verification works.

**Steps**:

1. Register a new user but don't verify
2. Navigate to login page
3. Try to use resend verification (or hit API directly)

**API Test**:
```bash
curl -X POST http://localhost:8000/api/v1/auth/resend-verification \
  -H "Content-Type: application/json" \
  -d '{"email": "unverified@example.com"}'
```

**Expected Results**:
- New verification token generated
- New verification email sent (logged)
- Response always says "If email registered..." (security)

---

### Test Scenario 9: Cancel Join Request

**Goal**: Verify user can cancel their own pending request.

**Prerequisites**: User with pending join request

**Steps**:

1. Login as user with pending join request
2. Hit API to cancel:
   ```bash
   curl -X DELETE http://localhost:8000/api/v1/join-requests/{request_id} \
     -H "Authorization: Bearer <token>"
   ```

**Expected Results**:
- Join request deleted
- 204 No Content response
- Request no longer appears in parent's list

---

### Test Scenario 10: Edge Cases

#### 10a: Child tries to register without parent email
**Steps**: Skip parent email field during child registration
**Expected**: 400 error "Children must provide a parent's email"

#### 10b: Register with existing email
**Steps**: Try to register with email that already exists
**Expected**: 400 error "Email already registered"

#### 10c: Verify with invalid token
**Steps**: Go to `/verify-email/invalid-token-123`
**Expected**: Error page "Invalid or expired verification token"

#### 10d: Approve unverified user
**Steps**: Parent tries to approve child who hasn't verified email
**Expected**: 400 error explaining child must verify first

#### 10e: Review already-reviewed request
**Steps**: Try to approve/reject a request that's already been handled
**Expected**: 400 error "This request has already been approved/rejected"

#### 10f: Cancel non-pending request
**Steps**: Try to cancel an approved or rejected request
**Expected**: 400 error "Cannot cancel a request that has been approved/rejected"

---

### API Quick Reference for Testing

```bash
# Base URL
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

# Login
curl -X POST $BASE/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "test@example.com", "password": "TestPass123"}'

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

## Coverage Reports

### Backend Coverage Goals

```
app/models/chore.py           100%
app/services/chore_service.py  95%
app/services/recurrence_service.py  100%
app/routers/chores.py          90%
app/routers/chore_assignments.py  95%
app/jobs/chore_jobs.py         80%
----------------------------------------
TOTAL                          92%
```

### Frontend Coverage Goals

```
src/services/choreService.js    90%
src/pages/ChoresPage.jsx        80%
src/pages/ChoreCreatePage.jsx   80%
src/pages/MyChoresPage.jsx      85%
src/pages/PendingApprovalsPage.jsx  85%
src/components/chores/*         75%
----------------------------------------
TOTAL                           82%
```
