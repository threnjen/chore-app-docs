# Phase 3: Enhanced Chores
**Duration: 2 weeks**

---

## Overview

Phase 3 expands the chore system with first-dibs claiming, template libraries, and improved mobile UX. This phase builds on the MVP chore system from Phase 2.

---

## Goals

- First-dibs chore system
- Chore templates library
- Improved mobile UX

---

## Deliverables

### 1. Backend

- First-dibs claiming logic with race condition handling
- Chore templates library (predefined chores)
- Bulk chore creation endpoint
- Notification system (basic push notifications)

### 2. Frontend

- First-dibs UI (available chores, claim button)
- Chore templates browser
- Bulk chore creation wizard
- Mobile swipe actions (swipe to approve/reject)
- Pull-to-refresh
- Overdue badges with cumulative days

### 3. Features

- First-dibs assignment type
- Chore claiming with timeout
- Template library (10-20 common chores)
- Clone existing chores
- Batch approval
- Push notifications (new chore, completion, approval)

---

## Technical Details

### First-Dibs Implementation

```python
# Race condition handling for first-dibs claims
async def claim_chore(
    chore_assignment_id: UUID,
    user_id: UUID,
    db: Session
) -> ChoreAssignment:
    """
    Claim a first-dibs chore with optimistic locking.
    """
    # Lock the row to prevent concurrent claims
    assignment = db.query(ChoreAssignment).filter(
        ChoreAssignment.id == chore_assignment_id,
        ChoreAssignment.status == "AVAILABLE"
    ).with_for_update().first()
    
    if not assignment:
        raise HTTPException(409, "Chore already claimed by another family member")
    
    assignment.assigned_to_id = user_id
    assignment.status = "CLAIMED"
    assignment.claimed_at = datetime.utcnow()
    
    db.commit()
    return assignment
```

### Chore Templates Schema

```python
class ChoreTemplate(Base):
    __tablename__ = "chore_templates"
    
    id = Column(UUID, primary_key=True, default=uuid4)
    name = Column(String(100), nullable=False)
    description = Column(Text)
    category = Column(String(50))  # e.g., "bedroom", "kitchen", "outdoor"
    suggested_reward = Column(Numeric(10, 2))
    suggested_age_bracket = Column(String(20))  # young, tween, teen
    suggested_recurrence = Column(JSON)  # Default recurrence pattern
    is_system = Column(Boolean, default=True)  # System vs family-created
    family_id = Column(UUID, ForeignKey("families.id"), nullable=True)  # NULL for system templates
    created_at = Column(DateTime(timezone=True), server_default=func.now())
```

### Template Categories

| Category | Examples |
|----------|----------|
| Bedroom | Make bed, Clean room, Organize closet |
| Kitchen | Set table, Clear dishes, Load dishwasher, Wipe counters |
| Bathroom | Clean bathroom, Wipe mirror, Take out trash |
| Outdoor | Take out trash, Mow lawn, Water plants, Walk dog |
| Pet Care | Feed pet, Clean litter box, Walk dog |
| Laundry | Put away clothes, Sort laundry, Fold towels |
| General | Vacuum, Dust, Take out recycling |

### Bulk Chore Creation API

```python
@router.post("/chores/bulk")
async def create_bulk_chores(
    request: BulkChoreCreateRequest,
    context: FamilyContext = Depends(get_current_family_context),
    db: Session = Depends(get_db)
):
    """
    Create multiple chores from templates or custom definitions.
    """
    created_chores = []
    
    for chore_def in request.chores:
        if chore_def.template_id:
            # Clone from template
            template = get_template(chore_def.template_id, db)
            chore = create_chore_from_template(template, chore_def.overrides, context.family_id, db)
        else:
            # Create custom chore
            chore = create_chore(chore_def, context.family_id, db)
        
        created_chores.append(chore)
    
    db.commit()
    return {"created": len(created_chores), "chores": created_chores}
```

---

## Mobile UX Enhancements

### Swipe Actions

```jsx
// Swipe to approve/reject for parents
<SwipeableCard
  onSwipeLeft={() => rejectChore(assignment.id)}
  onSwipeRight={() => approveChore(assignment.id)}
  leftLabel="Reject"
  rightLabel="Approve"
  leftColor="red"
  rightColor="green"
>
  <ChoreCard assignment={assignment} />
</SwipeableCard>
```

### Pull-to-Refresh

```jsx
// Pull-to-refresh implementation
<PullToRefresh
  onRefresh={async () => {
    await refetchChores();
  }}
>
  <ChoreList chores={chores} />
</PullToRefresh>
```

### Overdue Badge

```jsx
<OverdueBadge days={assignment.total_overdue_days}>
  {assignment.total_overdue_days === 1 
    ? "1 day overdue" 
    : `${assignment.total_overdue_days} days overdue`}
</OverdueBadge>
```

---

## Database Migrations

### New Tables

```sql
-- Chore templates table
CREATE TABLE chore_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    category VARCHAR(50),
    suggested_reward DECIMAL(10, 2),
    suggested_age_bracket VARCHAR(20),
    suggested_recurrence JSONB,
    is_system BOOLEAN DEFAULT true,
    family_id UUID REFERENCES families(id),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_chore_templates_family ON chore_templates(family_id);
CREATE INDEX idx_chore_templates_category ON chore_templates(category);
```

### Seed System Templates

```sql
-- System chore templates
INSERT INTO chore_templates (name, description, category, suggested_reward, suggested_age_bracket, suggested_recurrence, is_system) VALUES
('Make Bed', 'Make your bed neatly every morning', 'bedroom', 0.50, 'young', '{"frequency": "DAILY", "interval": 1}', true),
('Clean Room', 'Pick up toys, organize belongings', 'bedroom', 2.00, 'young', '{"frequency": "WEEKLY", "interval": 1}', true),
('Set Table', 'Set the table for dinner', 'kitchen', 1.00, 'young', '{"frequency": "DAILY", "interval": 1}', true),
('Load Dishwasher', 'Load dishes into dishwasher after meals', 'kitchen', 2.00, 'tween', '{"frequency": "DAILY", "interval": 1}', true),
('Take Out Trash', 'Take trash bins to curb', 'outdoor', 3.00, 'tween', '{"frequency": "WEEKLY", "interval": 1}', true),
('Mow Lawn', 'Mow front and back lawn', 'outdoor', 10.00, 'teen', '{"frequency": "WEEKLY", "interval": 1}', true),
('Walk Dog', 'Walk the dog for 20 minutes', 'pet_care', 2.00, 'tween', '{"frequency": "DAILY", "interval": 1}', true),
('Vacuum House', 'Vacuum all carpeted areas', 'general', 5.00, 'teen', '{"frequency": "WEEKLY", "interval": 1}', true);
```

---

## API Endpoints

### Templates

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/chores/templates` | GET | List all available templates (system + family) |
| `/api/v1/chores/templates/{id}` | GET | Get template details |
| `/api/v1/chores/templates` | POST | Create family custom template |
| `/api/v1/chores/templates/{id}` | DELETE | Delete family custom template |

### Bulk Operations

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/chores/bulk` | POST | Create multiple chores at once |
| `/api/v1/chores/assignments/bulk/approve` | POST | Batch approve assignments |
| `/api/v1/chores/assignments/bulk/reject` | POST | Batch reject assignments |

### Claiming

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/chores/assignments/{id}/claim` | POST | Claim a first-dibs chore |
| `/api/v1/chores/assignments/{id}/unclaim` | POST | Release a claimed chore |

---

## Testing Requirements

### Backend Tests

```python
# Race condition test
def test_concurrent_chore_claim():
    """Two kids trying to claim same chore simultaneously."""
    # Create first-dibs chore
    chore = create_first_dibs_chore(family_id)
    assignment = get_pending_assignment(chore.id)
    
    # Simulate concurrent claims
    import threading
    results = []
    
    def claim_for_kid(kid_id):
        try:
            result = client.post(f"/chores/assignments/{assignment.id}/claim", headers=kid_headers[kid_id])
            results.append((kid_id, result.status_code))
        except Exception as e:
            results.append((kid_id, str(e)))
    
    t1 = threading.Thread(target=claim_for_kid, args=(kid1.id,))
    t2 = threading.Thread(target=claim_for_kid, args=(kid2.id,))
    
    t1.start()
    t2.start()
    t1.join()
    t2.join()
    
    # One should succeed, one should fail
    success_count = sum(1 for _, status in results if status == 200)
    assert success_count == 1
```

### Coverage Requirements

- First-dibs claiming: 100%
- Template CRUD: 80%
- Bulk operations: 80%
- Notification delivery: 80%

### E2E Tests

- Template browser workflow
- Bulk chore creation wizard
- First-dibs claim flow
- Swipe gestures (mobile viewport)
- Pull-to-refresh

---

## UI Mockups

### Templates Browser

```
┌─────────────────────────────────────┐
│ 📋 Chore Templates                  │
├─────────────────────────────────────┤
│ [🔍 Search templates...]            │
├─────────────────────────────────────┤
│ 🛏️ Bedroom                          │
│   • Make Bed ($0.50)                │
│   • Clean Room ($2.00)              │
│   • Organize Closet ($3.00)         │
├─────────────────────────────────────┤
│ 🍽️ Kitchen                          │
│   • Set Table ($1.00)               │
│   • Clear Dishes ($1.50)            │
│   • Load Dishwasher ($2.00)         │
├─────────────────────────────────────┤
│ 🌳 Outdoor                          │
│   • Take Out Trash ($3.00)          │
│   • Mow Lawn ($10.00)               │
└─────────────────────────────────────┘
```

### First-Dibs Available Chores

```
┌─────────────────────────────────────┐
│ 🎯 Available Chores                 │
│ First come, first served!           │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │
│ │ 🧹 Vacuum Living Room           │ │
│ │ Reward: $3.00                   │ │
│ │ Due: Today 5:00 PM              │ │
│ │        [ Claim This Chore ]     │ │
│ └─────────────────────────────────┘ │
│ ┌─────────────────────────────────┐ │
│ │ 🚽 Clean Bathroom               │ │
│ │ Reward: $5.00                   │ │
│ │ Due: Tomorrow                   │ │
│ │        [ Claim This Chore ]     │ │
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

---

## Definition of Done

### Must Have
- [ ] First-dibs claiming works with proper race condition handling
- [ ] Chore template library with 10+ system templates
- [ ] Bulk chore creation from templates
- [ ] Swipe to approve/reject on mobile
- [ ] Pull-to-refresh on chore lists
- [ ] Overdue badges display correctly
- [ ] All tests passing with 80%+ coverage

### Nice to Have
- [ ] Custom template creation by family
- [ ] Template suggestions based on kid age
- [ ] Haptic feedback on swipe
- [ ] Animation on chore claim

---

## Dependencies

- **Requires**: Phase 2 completion (chores MVP)
- **Enables**: Phase 4 (can run in parallel with Phase 4)

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Race conditions on claims | PostgreSQL row locking with FOR UPDATE |
| Swipe gestures inconsistent | Use proven library (react-swipeable) |
| Template overload | Categorize and search, limit initial templates |

---

*Phase 3 Document - Last Updated: February 2026*
