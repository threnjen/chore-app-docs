# Phase 2 QA: Permissions

> Permission testing, multi-tenant isolation, and multi-household scenarios

**Last Updated**: February 3, 2026

---

## 🔐 Permission Testing

### Child Restrictions

- [ ] Login as child
- [ ] Verify cannot access `/chores/create`
- [ ] Verify cannot see "New Chore" button
- [ ] Verify cannot approve/reject chores
- [ ] Verify can only see own assignments

### Parent Access

- [ ] Login as parent
- [ ] Verify can see all family assignments
- [ ] Verify can create/edit/delete chores
- [ ] Verify can approve/reject

---

## 🌐 Multi-Tenant Isolation

- [ ] Login as user in Family A
- [ ] Create a chore
- [ ] Login as user in Family B
- [ ] Verify cannot see Family A's chore
- [ ] Verify `/chores/{id}` returns 404 for other family's chore

---

## 👨‍👩‍👧‍👦 Multi-Household Testing

### Child in Multiple Families

**Setup**: child3 (Eddie) belongs to both "Parent Family" and "Third Family"

- [ ] Login as child3@test.com
- [ ] Verify family switcher shows both families
- [ ] Switch to "Parent Family"
- [ ] Verify sees chores from Parent Family only
- [ ] Switch to "Third Family"
- [ ] Verify sees chores from Third Family only
- [ ] Complete chore in each family
- [ ] Verify rewards go to correct family's account

### Parent Managing Multi-Household Child

- [ ] Login as parent1@test.com (Parent Family)
- [ ] Create chore assigned to child3
- [ ] Login as parent3@test.com (Third Family)
- [ ] Create chore assigned to child3
- [ ] Verify each parent only sees their own family's assignments

**Backend Verification for Multi-Household**:
```bash
# Check child3's family memberships
docker-compose exec db psql -U picklesapp -d picklesapp -c \
  "SELECT fm.id, u.email, f.name as family_name, fm.role
   FROM family_memberships fm
   JOIN users u ON fm.user_id = u.id
   JOIN families f ON fm.family_id = f.id
   WHERE u.email='child3@test.com';"
# Expected: 2 rows (two family memberships)
```
