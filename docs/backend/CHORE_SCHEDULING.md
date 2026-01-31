# Chore Scheduling System

> Technical documentation for the PicklesApp chore scheduling engine

---

## Overview

The chore scheduling system is the core engine that powers recurring chore assignments in PicklesApp. It uses a modified RFC 5545 iCalendar RRULE format with custom extensions to support both absolute and relative timing modes.

---

## Recurrence Rule Format

### Base Fields (RFC 5545)

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| frequency | String | DAILY, WEEKLY, MONTHLY, YEARLY | Yes |
| interval | Integer | Every N frequency units (default: 1) | No |
| by_weekday | Array[String] | Weekday codes: MO, TU, WE, TH, FR, SA, SU | No |
| by_monthday | Array[Integer] | Day(s) of month (1-31, -1 for last day) | No |
| by_month | Array[Integer] | Month(s) of year (1-12) | No |
| start_date | Date | First possible occurrence | Yes |
| end_date | Date | Last possible occurrence (null = infinite) | No |
| count | Integer | Maximum number of occurrences | No |

### PicklesApp Extensions

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| timing_mode | String | "absolute" or "relative" | "absolute" |

---

## Timing Modes

### Absolute Mode

In absolute mode, chore instances are generated at fixed intervals from the start date, regardless of when the chore is completed.

**Use Cases:**
- Time-sensitive chores (trash day is always Thursday)
- Calendar-tied routines (clean before guests arrive Saturday)
- Chores that must happen on schedule regardless of completion

**Behavior:**
- Schedule: Every Monday starting Feb 1
- Timeline: Feb 1, Feb 8, Feb 15, Feb 22, Mar 1...
- Scenario: Kid completes Feb 1 chore on Feb 5 (late)
- Result: Next chore still due Feb 8

### Relative Mode

In relative mode, the next chore instance is generated relative to when the previous instance was **completed**, not when it was due.

**Use Cases:**
- Maintenance tasks (vacuum 2 weeks after last vacuuming)
- Flexible cleaning schedules
- Chores where consistent intervals between completions matter

**Behavior:**
- Schedule: Every 2 weeks starting Feb 1
- Timeline initially: Feb 1, Feb 15, Mar 1...
- Scenario: Kid completes Feb 1 chore on Feb 5 (late)
- Result: Next chore due Feb 19 (2 weeks from Feb 5)

---

## Common Patterns

### Daily Patterns

| Pattern | Rule |
|---------|------|
| Every day | `{"frequency": "DAILY", "interval": 1}` |
| Every 3 days | `{"frequency": "DAILY", "interval": 3}` |
| Weekdays only | `{"frequency": "WEEKLY", "by_weekday": ["MO","TU","WE","TH","FR"]}` |

### Weekly Patterns

| Pattern | Rule |
|---------|------|
| Every Monday | `{"frequency": "WEEKLY", "by_weekday": ["MO"]}` |
| M-W-F | `{"frequency": "WEEKLY", "by_weekday": ["MO","WE","FR"]}` |
| Bi-weekly Saturday | `{"frequency": "WEEKLY", "interval": 2, "by_weekday": ["SA"]}` |
| Weekends | `{"frequency": "WEEKLY", "by_weekday": ["SA","SU"]}` |

### Monthly Patterns

| Pattern | Rule |
|---------|------|
| 1st of month | `{"frequency": "MONTHLY", "by_monthday": [1]}` |
| 1st and 15th | `{"frequency": "MONTHLY", "by_monthday": [1, 15]}` |
| Last day | `{"frequency": "MONTHLY", "by_monthday": [-1]}` |
| Every quarter | `{"frequency": "MONTHLY", "interval": 3, "by_monthday": [1]}` |

---

## Assignment Lifecycle

### ASSIGNED Chore Flow

```
[Generated] -> PENDING -> COMPLETED -> APPROVED (Reward Credited)
                  |            |
                  |            +-> REJECTED -> COMPLETED (retry)
                  |
                  +-> EXPIRED (if expiration enabled)
```

### FIRST_DIBS Chore Flow (Phase 3)

```
[Generated] -> AVAILABLE -> CLAIMED -> COMPLETED -> APPROVED
                  |
                  +-> EXPIRED (if no one claims)
```

---

## Overdue Handling

### Single Instance Policy

- Only ONE active (non-completed) assignment exists at a time per chore per user
- Overdue chores don't accumulate (no backlog of "vacuum stairs x3")
- The single overdue instance tracks cumulative overdue days via `total_overdue_days`

### Display Example

```
Today's Chores:
+-------------------------------------+
| Clean Room                          |
| Due: Today at 8 PM                  |
| [Mark Complete]                     |
+-------------------------------------+

Overdue:
+-------------------------------------+
| Take Out Trash          3 days late |
| Originally due: Monday              |
| [Mark Complete]                     |
+-------------------------------------+
```

---

## Background Jobs

| Job | Schedule | Purpose |
|-----|----------|---------|
| generate_chore_instances | Hourly at :00 | Create upcoming assignments for next 7 days |
| update_overdue_chores | Daily at 00:05 | Update total_overdue_days for pending assignments |
| auto_approve_chores | Hourly at :30 | Auto-approve chores past their threshold |
| expire_chores | Daily at 00:15 | Mark expired assignments |

---

## Edge Cases

### Leap Year
- Monthly chores on the 29th skip February in non-leap years
- Or fall back to Feb 28 depending on dateutil configuration

### Month Boundaries
- Chores on the 31st skip 30-day months
- Last day of month (`by_monthday: [-1]`) works correctly

### Daylight Saving Time
- Deadline times are stored without timezone
- Applied using family's configured timezone
- 8:00 AM deadline stays at 8:00 AM local time

### Timezone Changes
- If family changes timezone, deadline times shift to new timezone
- Due dates are date-only (no timezone)

---

## Database Indexes

For optimal query performance:

- `idx_chore_assignments_chore` - Lookups by chore
- `idx_chore_assignments_family` - Family scoping
- `idx_chore_assignments_assigned_to` - User's assignments
- `idx_chore_assignments_status` - Status filtering
- `idx_chore_assignments_due_date` - Date-based queries
- `idx_chore_assignments_user_status_date` - Compound index for common queries

---

For complete implementation details, see [PHASE_2_DETAILED.md](../phases/PHASE_2_DETAILED.md).
