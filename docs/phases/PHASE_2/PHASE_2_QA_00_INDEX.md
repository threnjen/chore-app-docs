# Phase 2 QA Checklist Index

> Manual testing checklist for Phase 2: Chores MVP

**Last Updated**: February 3, 2026

---

## Overview

This QA checklist has been divided into focused sections for easier testing. Complete each section and mark sign-off below.

---

## Checklist Sections

| # | Section | File | Description |
|---|---------|------|-------------|
| 1 | **Setup & Reference** | [PHASE_2_QA_SETUP.md](PHASE_2_QA_SETUP.md) | Prerequisites, test accounts, database access queries |
| 2 | **Chore Management** | [PHASE_2_QA_CHORES.md](PHASE_2_QA_CHORES.md) | Parent chore CRUD, recurrence patterns, assignment types |
| 3 | **Child Workflow** | [PHASE_2_QA_CHILD_WORKFLOW.md](PHASE_2_QA_CHILD_WORKFLOW.md) | Child chore view, completion flow, age-appropriate UI |
| 4 | **Approvals** | [PHASE_2_QA_APPROVALS.md](PHASE_2_QA_APPROVALS.md) | Parent approval workflow, all assignment statuses |
| 5 | **Rewards & Penalties** | [PHASE_2_QA_REWARDS.md](PHASE_2_QA_REWARDS.md) | Tiered rewards, reward types, penalties, reward integration |
| 6 | **Permissions** | [PHASE_2_QA_PERMISSIONS.md](PHASE_2_QA_PERMISSIONS.md) | Permission testing, multi-tenant, multi-household |
| 7 | **Edge Cases** | [PHASE_2_QA_EDGE_CASES.md](PHASE_2_QA_EDGE_CASES.md) | Edge cases and unusual scenarios |
| 8 | **API & Automation** | [PHASE_2_QA_API_TESTS.md](PHASE_2_QA_API_TESTS.md) | cURL commands, automated test execution |

### Related Phase 1 Testing

| Section | File | Description |
|---------|------|-------------|
| **Family Building** | [PHASE_1_QA_FAMILY_BUILDING.md](PHASE_1_QA_FAMILY_BUILDING.md) | Bidirectional family building flows (registration, join requests) |

---

## ✅ Sign-Off

| Area | Tester | Date | Status |
|------|--------|------|--------|
| Parent Chore Management | API/Automated | 2026-02-03 | ✅ |
| Child Chore Workflow | API/Automated | 2026-02-03 | ✅ |
| Parent Approval Workflow | API/Automated | 2026-02-03 | ✅ |
| Tiered Rewards | pytest | 2026-02-03 | ✅ |
| Reward Types (MONEY/POINTS/STARS) | API/Automated | 2026-02-03 | ✅ |
| Penalty System | pytest | 2026-02-03 | ✅ |
| Recurrence Patterns | pytest/API | 2026-02-03 | ✅ |
| Timing Modes (ABSOLUTE/RELATIVE) | pytest | 2026-02-03 | ✅ |
| Auto-Approve | pytest | 2026-02-03 | ✅ |
| Photo Required | | | ⬜ |
| Permissions | API/Automated | 2026-02-03 | ✅ |
| Multi-Tenant Isolation | | | ⬜ |
| Multi-Household Child | | | ⬜ |
| Assignment Statuses (all 8) | API/Automated | 2026-02-03 | ✅ |
| Reward Integration | DB Verified | 2026-02-03 | ✅ |
| Age-Appropriate UI | | | ⬜ |
| Edge Cases | | | ⬜ |
| Bidirectional Family Building | | | ⬜ |
| API Testing | cURL/Automated | 2026-02-03 | ✅ |
| Automated Tests Pass | pytest | 2026-02-03 | ✅ |

---

## 🤖 Automated QA Summary (February 3, 2026)

### Backend Tests (pytest)
- **Result**: ✅ **73 passed, 1 skipped**
- Files tested:
  - `test_chores.py` - Chore CRUD, validation
  - `test_chore_assignments.py` - Assignment workflow, approval/rejection
  - `test_recurrence.py` - RFC 5545 recurrence rules
  - `test_tiered_rewards.py` - Reward tiers, penalties, grace periods

### API Testing (cURL)
- ✅ Authentication (parent/child tokens)
- ✅ List chores (26 chores in system)
- ✅ Create chore with recurrence rule
- ✅ Child views assignments (18 assignments)
- ✅ Child completes assignment → PENDING_APPROVAL
- ✅ Parent views pending approvals (8 pending)
- ✅ Parent approves → APPROVED, reward credited
- ✅ Parent rejects → REJECTED with reason
- ✅ Recurrence preview (M-W-F pattern verified)
- ✅ Permission check (child cannot create chores)

### Database Verification
| Table | Count |
|-------|-------|
| Chores | 33 |
| Assignments | 152 |
| Transactions | 26 |
| APPROVED | 5 |
| PENDING | 131 |
| REJECTED | 2 |
| PENDING_APPROVAL | 6 |

### E2E Tests (Playwright)
- **Status**: ⚠️ Needs test fixes (not app issues)
- **Issues**: Password field selector ambiguity, localStorage access timing
- **Recommendation**: Update selectors in `e2e/chores.spec.js`

---

## Notes

Use this space to document any issues found during QA:

| Issue | Severity | Status | Notes |
|-------|----------|--------|-------|
| E2E Password selector | Low | Open | `getByLabel('Password')` matches 2 elements - use `getByPlaceholder` or input locator |
| E2E localStorage access | Low | Open | Move `localStorage.clear()` after `page.goto()` |
| Chore.chore_title null in nested response | Info | Noted | Some assignment responses show null chore_title in nested object |
