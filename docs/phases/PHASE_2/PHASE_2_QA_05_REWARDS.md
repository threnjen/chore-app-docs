# Phase 2 QA: Rewards & Penalties

> Tiered rewards, reward types, penalties, and reward integration

**Last Updated**: February 3, 2026

---

## 💰 Tiered Rewards Testing

### Full Reward (On-Time)

- [ ] Create chore with $10 reward
- [ ] Complete before due date
- [ ] Approve
- [ ] Verify $10 credited

### Bonus Reward (Early)

- [ ] Create chore with $10 reward + $2 early bonus + 24h threshold
- [ ] Complete 48+ hours early
- [ ] Approve
- [ ] Verify $12 credited (effective_reward)

### Reduced Reward (Late)

- [ ] Create chore with $10 reward + 1 day grace + 50% late percentage
- [ ] Complete 2 days after due
- [ ] Approve
- [ ] Verify $5 credited (50% of reward)

**Backend Verification for Tiered Rewards**:
```bash
# Check assignment reward tier after completion
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT ca.id, c.title, ca.status, ca.reward_tier, ca.effective_reward, c.reward_amount
   FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id
   ORDER BY ca.completed_at DESC NULLS LAST LIMIT 5;"
# Expected: reward_tier=BONUS/FULL/REDUCED, effective_reward calculated

# Check transaction amount matches effective_reward
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT t.amount, t.description FROM transactions t 
   WHERE t.category='CHORE_REWARD' ORDER BY t.created_at DESC LIMIT 3;"
```

---

## 🎯 Reward Types Testing

### MONEY Reward Type

- [ ] Create chore with reward_type=MONEY, reward_amount=5.00
- [ ] Complete and approve
- [ ] Verify transaction shows dollar amount
- [ ] Verify child's SPEND account balance increases by $5.00

### POINTS Reward Type

- [ ] Create chore with reward_type=POINTS, reward_amount=100
- [ ] Complete and approve
- [ ] Verify transaction shows points (not dollars)
- [ ] Verify UI displays "100 points" not "$100.00"
- [ ] Verify points tracked correctly

**Backend Verification for POINTS**:
```bash
# Check chore reward type
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT id, title, reward_type, reward_amount FROM chores WHERE reward_type='POINTS';"

# Check assignment effective_reward for points
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT ca.effective_reward, c.reward_type FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id WHERE c.reward_type='POINTS';"
```

### STARS Reward Type

- [ ] Create chore with reward_type=STARS, reward_amount=5
- [ ] Complete and approve
- [ ] Verify UI displays "5 stars" with star icon
- [ ] Verify stars tracked correctly

### Mixed Reward Types in Approval Queue

- [ ] Have pending approvals with MONEY, POINTS, and STARS
- [ ] Verify approval queue shows correct icons/labels for each type
- [ ] Verify batch approve handles different types correctly

---

## ⚠️ Penalty Testing

### Penalty on Rejection

- [ ] Create chore with penalty_behavior=ON_REJECTION, penalty=$2
- [ ] Complete the chore
- [ ] Reject with reason
- [ ] Verify $2 debited from child account
- [ ] Verify transaction category is CHORE_PENALTY

### Penalty on Expiration

- [ ] Create chore with penalty_behavior=ON_EXPIRATION, expiration_days=1
- [ ] Let chore expire (or trigger expire job manually)
- [ ] Verify assignment status is EXPIRED
- [ ] Verify penalty applied

### Escalating Penalty

- [ ] Create chore with base penalty $1 + $0.50/day + max $5
- [ ] Let expire 10 days overdue
- [ ] Verify effective_penalty = $5 (capped)

**Backend Verification for Penalties**:
```bash
# Check chore penalty configuration
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT id, title, penalty_behavior, penalty_amount, penalty_per_day, max_penalty
   FROM chores WHERE penalty_amount > 0;"

# Check assignment status and penalty
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT ca.id, c.title, ca.status, ca.effective_penalty, ca.rejection_reason
   FROM chore_assignments ca
   JOIN chores c ON ca.chore_id = c.id
   WHERE ca.status IN ('REJECTED', 'EXPIRED')
   ORDER BY ca.updated_at DESC LIMIT 5;"

# Check penalty transaction was created
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT t.amount, t.transaction_type, t.category, t.description 
   FROM transactions t WHERE t.category='CHORE_PENALTY' 
   ORDER BY t.created_at DESC LIMIT 3;"
# Expected: transaction_type=DEBIT, category=CHORE_PENALTY

# Manually trigger expire job (if needed)
docker-compose exec api python -c "
import asyncio
from app.jobs.chore_jobs import expire_overdue_assignments
asyncio.run(expire_overdue_assignments())
"
```

---

## 💳 Reward Integration

- [ ] Approval creates transaction
- [ ] Transaction appears in child's account
- [ ] Balance increased by reward amount
- [ ] Transaction shows "Chore Reward" category (CHORE_REWARD)
- [ ] Transaction description includes chore title

**Backend Verification**:
```bash
# Check transaction was created after approving a chore
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT t.id, t.amount, t.transaction_type, t.category, t.description, a.account_type
   FROM transactions t
   JOIN accounts a ON t.account_id = a.id
   ORDER BY t.created_at DESC LIMIT 5;"
# Expected: transaction_type=CREDIT, category=CHORE_REWARD

# Check account balance updated
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT u.first_name, a.account_type, a.balance 
   FROM accounts a JOIN users u ON a.owner_id = u.id
   WHERE u.email='CHILD_EMAIL';"
```
