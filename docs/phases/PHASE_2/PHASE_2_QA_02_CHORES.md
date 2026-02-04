# Phase 2 QA: Chore Management

> Parent chore CRUD, recurrence patterns, and assignment types

**Last Updated**: February 3, 2026

---

## 🏠 Parent Chore Management

### Create Chore

- [x] Navigate to `/chores` from sidebar
- [x] Click "New Chore" button
- [x] **Form Fields**:
  - [x] Enter title "Test Daily Chore"
  - [x] Enter description "Test description"
  - [x] Set reward amount $5.00
  - [x] Select child to assign to
  - [x] Select frequency "Daily"
  - [x] Verify weekday picker hidden for Daily
  - [x] Select frequency "Weekly"
  - [x] Verify weekday picker appears
  - [x] Select M-W-F
  - [x] Can set interval (every 2 weeks)
  - [x] Can toggle between Absolute and Relative timing mode
  - [x] Verify preview shows next 5 occurrences correctly
  - [x] Set deadline time (optional)
  - [x] Set estimated duration (optional)
  - [x] Toggle "Require Photo"
  - [x] Toggle "Require Approval"
  - [x] Set auto-approve after hours (optional)
  - [x] Toggle "Show Advanced" mode
  - [x] Set grace period (1 day)
  - [x] Set late reward percentage (50%)
  - [x] Set early bonus ($1.00)
  - [x] Set penalty behavior (On Expiration)
  - [x] Set penalty amount ($1.00)
- [x] Submit form
- [x] Verify redirect to `/chores`
- [x] Verify chore appears in list
- [x] Verify initial assignments generated (check via API or database)

### List Chores

- [x] Verify all chores display correctly
- [x] Verify reward badges show correctly
- [x] Verify penalty badges (if configured)
- [x] Verify recurrence pattern descriptions
- [x] Verify assignment count displays
- [x] Can filter by active/inactive
- [x] Can click to view chore details

### View/Edit Chore

- [x] Click on a chore to view details
- [x] Verify `/chores/{id}` page loads
- [x] Click "Edit" button
- [x] Modify title
- [x] Save changes
- [x] Verify update reflected

### Delete Chore

- [x] Click "Delete" on a chore
- [x] Confirm deletion (with confirmation dialog)
- [x] Verify chore removed (soft-delete)
- [x] Verify with `include_inactive=true` query

---

## 🔄 Recurrence Testing

### One-Time Chore (No Recurrence)

- [ ] Create chore without recurrence rule
- [ ] Verify only single assignment created
- [ ] Preview shows just the one date
- [ ] Complete and approve
- [ ] Verify no future assignments generated

### Daily Pattern

- [ ] Create daily chore
- [ ] Verify assignments created for next 7 days
- [ ] Preview shows consecutive days

**Backend Verification for Recurrence**:
```bash
# Check chore recurrence rule
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT id, title, recurrence_rule FROM chores ORDER BY created_at DESC LIMIT 3;"

# Check generated assignments
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT ca.due_date, ca.status, c.title
   FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id
   WHERE c.title LIKE '%YOUR_CHORE_TITLE%'
   ORDER BY ca.due_date LIMIT 10;"
```

### Weekly M-W-F Pattern

- [ ] Create weekly M-W-F chore
- [ ] Preview shows only Mon/Wed/Fri dates

### Bi-Weekly Pattern

- [ ] Create chore with interval=2, weekly
- [ ] Preview shows every other week

### Monthly Pattern

- [ ] Create monthly on 15th
- [ ] Preview shows 15th of each month

### Last Day of Month

- [ ] Create monthly on by_monthday=[-1]
- [ ] Verify handles Feb correctly

### Timing Mode: ABSOLUTE vs RELATIVE

**ABSOLUTE timing** (default):
- [ ] Create chore with timing_mode=ABSOLUTE
- [ ] Complete chore on Day 1
- [ ] Next occurrence is still Day 2 (fixed schedule)

**RELATIVE timing**:
- [ ] Create chore with timing_mode=RELATIVE
- [ ] Complete chore on Day 1 (due originally Day 1)
- [ ] Verify next occurrence calculated from completion date
- [ ] Example: Complete 2 days late → next due shifts by 2 days

---

## 📋 Assignment Types Testing

### ASSIGNED Type (Direct Assignment)

- [ ] Create chore with assignment_type=ASSIGNED
- [ ] Select specific child(ren) to assign
- [ ] Verify assignments created with status=PENDING
- [ ] Verify assigned children see chore in their list
- [ ] Verify non-assigned children do NOT see chore
