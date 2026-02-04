# Phase 2 QA: Setup & Reference

> Prerequisites, test accounts, and database verification queries

**Last Updated**: February 3, 2026

---

## Prerequisites

- Docker Desktop running
- Backend: `docker-compose up` or `uvicorn app.main:app --reload`
- Database migrated: `docker-compose exec api alembic upgrade head`
- Seed data loaded: `docker-compose exec api python scripts/seed_dev_data.py`
- Frontend: `npm run dev` (in chore-app-frontend)
- DEV_MODE enabled (emails logged to console, not sent)
- Run QA on page http://localhost:5173/

### Resetting all data/containers

- `docker-compose down -v`
- `docker-compose up`
- `docker-compose exec api alembic upgrade head`
- `docker-compose exec api python scripts/seed_dev_data.py`

---

## 🔑 Test Accounts (Seed Data)

All accounts use password: `TestPass123`

### Parents

| Email | Name | Family | Role |
|-------|------|--------|------|
| parent1@test.com | Alice Parent | Parent Family | OWNER |
| parent2@test.com | Bob Parent | Parent Family | ADULT |
| parent3@test.com | Carol Parent | Third Family | OWNER |

### Children

| Email | Name | Age | Family | Account Types |
|-------|------|-----|--------|---------------|
| child1@test.com | Charlie Child | 10 | Parent Family | SPEND, SAVE, GIVE |
| child2@test.com | Diana Child | 8 | Parent Family | SPEND, SAVE |
| child3@test.com | Eddie Multi-Family | 13 | Parent Family + Third Family | SPEND only |
| child4@test.com | Fiona Youngest | 6 | Parent Family | SPEND only |
| child5@test.com | George Teen | 16 | Third Family | SPEND, SAVE, GIVE |

### Test Scenarios by User

- **child1**: Standard middle-age child, all account types
- **child2**: Younger child, limited accounts
- **child3**: Multi-household child (belongs to 2 families)
- **child4**: Youngest child (6yo), simplest setup
- **child5**: Teenager, Third Family only

---

## 🗃️ Database Access (Backend Verification)

### Quick Access Commands

```bash
# Connect to PostgreSQL CLI (interactive mode)
docker-compose exec db psql -U picklesapp -d picklesapp

# Run a single query and exit
docker-compose exec db psql -U picklesapp -d picklesapp -c "SELECT * FROM users LIMIT 5;"

# Pretty-print with expanded display
docker-compose exec db psql -U picklesapp -d picklesapp -c "\x" -c "SELECT * FROM users WHERE email='test@test.com';"
```

### Useful psql Commands (inside psql shell)

```sql
\dt                    -- List all tables
\d users               -- Describe users table (show columns)
\d+ users              -- Describe with more detail
\x                     -- Toggle expanded display (easier to read)
\q                     -- Quit psql
```

### Common Verification Queries

```sql
-- View all users
SELECT id, email, first_name, last_name, role, email_verified, created_at 
FROM users ORDER BY created_at DESC;

-- View a specific user by email
SELECT * FROM users WHERE email = 'test@example.com';

-- View all families
SELECT id, name, timezone, created_at FROM families;

-- View family memberships (who belongs to which family)
SELECT fm.id, u.email, u.first_name, f.name as family_name, fm.role 
FROM family_memberships fm
JOIN users u ON fm.user_id = u.id
JOIN families f ON fm.family_id = f.id;

-- View join requests / invitations
SELECT i.id, i.invitation_type, i.status, 
       requester.email as requester_email,
       target.email as target_email,
       f.name as family_name,
       i.message, i.created_at, i.expires_at
FROM invitations i
LEFT JOIN users requester ON i.requester_id = requester.id
LEFT JOIN users target ON i.target_email = target.email
LEFT JOIN families f ON i.family_id = f.id
ORDER BY i.created_at DESC;

-- View chores
SELECT id, title, reward_amount, is_active, created_at 
FROM chores ORDER BY created_at DESC;

-- View chore assignments
SELECT ca.id, c.title, u.first_name as assigned_to, ca.status, 
       ca.due_date, ca.reward_tier, ca.effective_reward
FROM chore_assignments ca
JOIN chores c ON ca.chore_id = c.id
LEFT JOIN users u ON ca.assigned_to_id = u.id
ORDER BY ca.due_date DESC;

-- View accounts
SELECT a.id, u.first_name as owner, a.account_type, a.balance 
FROM accounts a
JOIN users u ON a.owner_id = u.id;

-- View transactions
SELECT t.id, t.amount, t.transaction_type, t.category, t.description, t.created_at
FROM transactions t
ORDER BY t.created_at DESC LIMIT 10;
```
