# Phase 8: GraphQL Migration [OPTIONAL]
**Duration: 2 weeks**

> **Note**: This phase is optional. Proceed with REST API if GraphQL adds unnecessary complexity for the current feature set.

---

## Overview

Phase 8 introduces a GraphQL API layer alongside the existing REST API. This enables more efficient data fetching, real-time subscriptions, and a better developer experience for complex queries.

---

## Goals

- GraphQL API layer
- Frontend migration to GraphQL
- Maintain REST backwards compatibility

---

## Deliverables

### 1. Backend

- Strawberry GraphQL setup
- Schema definition
- Resolvers for all entities
- Subscriptions (real-time)
- REST endpoints marked deprecated

### 2. Frontend

- Apollo Client setup
- GraphQL queries/mutations
- Subscription handling
- Optimistic updates

### 3. Features

- GraphQL playground
- Efficient nested data fetching
- Real-time subscriptions
- Reduced over-fetching

---

## When to Use GraphQL vs REST

### Use GraphQL When:
- Complex nested data requirements (e.g., dashboard with accounts + chores + goals)
- Real-time updates are critical
- Mobile clients need minimal data transfer
- Frontend needs flexibility in data shape

### Keep REST When:
- Simple CRUD operations
- File uploads
- Third-party integrations
- WebHooks

---

## GraphQL Schema Design

### Type Definitions

```python
# app/graphql/types.py

import strawberry
from typing import List, Optional
from datetime import datetime, date
from decimal import Decimal

@strawberry.type
class User:
    id: strawberry.ID
    email: str
    first_name: str
    last_name: str
    birthdate: Optional[date]
    role: str
    age_bracket: str
    
    @strawberry.field
    def full_name(self) -> str:
        return f"{self.first_name} {self.last_name}"
    
    @strawberry.field
    async def accounts(self, info) -> List["Account"]:
        return await info.context.loaders.accounts_by_user.load(self.id)
    
    @strawberry.field
    async def chore_assignments(self, info, status: Optional[str] = None) -> List["ChoreAssignment"]:
        return await info.context.loaders.assignments_by_user.load((self.id, status))

@strawberry.type
class Family:
    id: strawberry.ID
    name: str
    timezone: str
    currency: str
    created_at: datetime
    
    @strawberry.field
    async def members(self, info) -> List[User]:
        return await info.context.loaders.members_by_family.load(self.id)
    
    @strawberry.field
    async def chores(self, info) -> List["Chore"]:
        return await info.context.loaders.chores_by_family.load(self.id)

@strawberry.type
class Account:
    id: strawberry.ID
    name: str
    account_type: str
    balance: Decimal
    created_at: datetime
    
    @strawberry.field
    async def transactions(self, info, limit: int = 10) -> List["Transaction"]:
        return await info.context.loaders.transactions_by_account.load((self.id, limit))
    
    @strawberry.field
    async def interest_config(self, info) -> Optional["InterestConfig"]:
        return await info.context.loaders.interest_by_account.load(self.id)

@strawberry.type
class Chore:
    id: strawberry.ID
    title: str
    description: Optional[str]
    reward_amount: Optional[Decimal]
    reward_type: str
    recurrence_rule: Optional[str]  # JSON string
    assignment_type: str
    is_active: bool
    
    @strawberry.field
    async def assignments(self, info, status: Optional[str] = None) -> List["ChoreAssignment"]:
        return await info.context.loaders.assignments_by_chore.load((self.id, status))

@strawberry.type
class ChoreAssignment:
    id: strawberry.ID
    status: str
    due_date: date
    due_time: Optional[str]
    completed_at: Optional[datetime]
    approved_at: Optional[datetime]
    total_overdue_days: int
    
    @strawberry.field
    async def chore(self, info) -> Chore:
        return await info.context.loaders.chore.load(self.chore_id)
    
    @strawberry.field
    async def assigned_to(self, info) -> Optional[User]:
        if self.assigned_to_id:
            return await info.context.loaders.user.load(self.assigned_to_id)
        return None

@strawberry.type
class Transaction:
    id: strawberry.ID
    amount: Decimal
    transaction_type: str
    description: Optional[str]
    balance_after: Decimal
    created_at: datetime

@strawberry.type
class SavingsGoal:
    id: strawberry.ID
    name: str
    target_amount: Decimal
    current_amount: Decimal
    target_date: Optional[date]
    icon: Optional[str]
    color: Optional[str]
    is_active: bool
    
    @strawberry.field
    def progress_percent(self) -> float:
        if self.target_amount == 0:
            return 100.0
        return float(self.current_amount / self.target_amount * 100)
    
    @strawberry.field
    def remaining_amount(self) -> Decimal:
        return max(Decimal('0'), self.target_amount - self.current_amount)
```

### Query Definition

```python
# app/graphql/queries.py

@strawberry.type
class Query:
    @strawberry.field
    async def me(self, info) -> User:
        """Get current authenticated user."""
        return await get_current_user(info.context)
    
    @strawberry.field
    async def family(self, info, id: Optional[strawberry.ID] = None) -> Family:
        """Get family by ID or current family context."""
        family_id = id or info.context.family_id
        return await get_family(family_id, info.context)
    
    @strawberry.field
    async def accounts(self, info, user_id: Optional[strawberry.ID] = None) -> List[Account]:
        """Get accounts for user (defaults to current user)."""
        uid = user_id or info.context.user_id
        return await get_accounts(uid, info.context.family_id)
    
    @strawberry.field
    async def chores(
        self, 
        info, 
        is_active: Optional[bool] = True,
        assignment_type: Optional[str] = None
    ) -> List[Chore]:
        """Get chores for current family."""
        return await get_chores(info.context.family_id, is_active, assignment_type)
    
    @strawberry.field
    async def chore_assignments(
        self,
        info,
        user_id: Optional[strawberry.ID] = None,
        status: Optional[str] = None,
        due_before: Optional[date] = None
    ) -> List[ChoreAssignment]:
        """Get chore assignments with filters."""
        uid = user_id or info.context.user_id
        return await get_assignments(uid, info.context.family_id, status, due_before)
    
    @strawberry.field
    async def savings_goals(
        self,
        info,
        user_id: Optional[strawberry.ID] = None,
        is_active: Optional[bool] = True
    ) -> List[SavingsGoal]:
        """Get savings goals."""
        uid = user_id or info.context.user_id
        return await get_savings_goals(uid, info.context.family_id, is_active)
    
    @strawberry.field
    async def transactions(
        self,
        info,
        account_id: strawberry.ID,
        limit: int = 50,
        offset: int = 0
    ) -> List[Transaction]:
        """Get transactions for account."""
        return await get_transactions(account_id, info.context.family_id, limit, offset)
```

### Mutation Definition

```python
# app/graphql/mutations.py

@strawberry.type
class Mutation:
    @strawberry.mutation
    async def complete_chore(
        self,
        info,
        assignment_id: strawberry.ID,
        photo_url: Optional[str] = None
    ) -> ChoreAssignment:
        """Mark a chore assignment as complete."""
        return await complete_chore_assignment(
            assignment_id,
            info.context.user_id,
            photo_url
        )
    
    @strawberry.mutation
    async def approve_chore(
        self,
        info,
        assignment_id: strawberry.ID,
        feedback: Optional[str] = None
    ) -> ChoreAssignment:
        """Approve a completed chore (parent only)."""
        require_parent(info.context)
        return await approve_chore_assignment(assignment_id, feedback)
    
    @strawberry.mutation
    async def reject_chore(
        self,
        info,
        assignment_id: strawberry.ID,
        reason: str
    ) -> ChoreAssignment:
        """Reject a completed chore (parent only)."""
        require_parent(info.context)
        return await reject_chore_assignment(assignment_id, reason)
    
    @strawberry.mutation
    async def claim_chore(
        self,
        info,
        assignment_id: strawberry.ID
    ) -> ChoreAssignment:
        """Claim a first-dibs chore."""
        return await claim_chore_assignment(
            assignment_id,
            info.context.user_id
        )
    
    @strawberry.mutation
    async def create_transaction(
        self,
        info,
        account_id: strawberry.ID,
        amount: Decimal,
        description: str,
        transaction_type: str = "MANUAL"
    ) -> Transaction:
        """Create a manual transaction."""
        return await create_transaction(
            account_id,
            amount,
            description,
            transaction_type,
            info.context
        )
    
    @strawberry.mutation
    async def transfer(
        self,
        info,
        from_account_id: strawberry.ID,
        to_account_id: strawberry.ID,
        amount: Decimal,
        description: Optional[str] = None
    ) -> List[Transaction]:
        """Transfer between accounts."""
        return await transfer_between_accounts(
            from_account_id,
            to_account_id,
            amount,
            description,
            info.context
        )
    
    @strawberry.mutation
    async def create_savings_goal(
        self,
        info,
        input: "SavingsGoalInput"
    ) -> SavingsGoal:
        """Create a new savings goal."""
        return await create_savings_goal(input, info.context)
    
    @strawberry.mutation
    async def contribute_to_goal(
        self,
        info,
        goal_id: strawberry.ID,
        amount: Decimal
    ) -> SavingsGoal:
        """Add money to a savings goal."""
        return await contribute_to_goal(goal_id, amount, info.context)
```

### Subscription Definition

```python
# app/graphql/subscriptions.py

import asyncio
from typing import AsyncGenerator

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def chore_updates(
        self,
        info,
        family_id: strawberry.ID
    ) -> AsyncGenerator[ChoreAssignment, None]:
        """Subscribe to chore assignment updates for a family."""
        async for update in subscribe_to_chore_updates(family_id):
            yield update
    
    @strawberry.subscription
    async def transaction_updates(
        self,
        info,
        account_id: strawberry.ID
    ) -> AsyncGenerator[Transaction, None]:
        """Subscribe to new transactions for an account."""
        async for transaction in subscribe_to_transactions(account_id):
            yield transaction
    
    @strawberry.subscription
    async def notifications(
        self,
        info
    ) -> AsyncGenerator["Notification", None]:
        """Subscribe to user notifications."""
        user_id = info.context.user_id
        async for notification in subscribe_to_notifications(user_id):
            yield notification
```

---

## DataLoader Implementation

```python
# app/graphql/loaders.py

from strawberry.dataloader import DataLoader
from typing import List

async def load_users(keys: List[str]) -> List[User]:
    """Batch load users by ID."""
    users = await db.query(User).filter(User.id.in_(keys)).all()
    user_map = {str(u.id): u for u in users}
    return [user_map.get(key) for key in keys]

async def load_accounts_by_user(keys: List[str]) -> List[List[Account]]:
    """Batch load accounts grouped by user ID."""
    accounts = await db.query(Account).filter(Account.user_id.in_(keys)).all()
    
    result = {key: [] for key in keys}
    for account in accounts:
        result[str(account.user_id)].append(account)
    
    return [result[key] for key in keys]

class DataLoaders:
    def __init__(self):
        self.user = DataLoader(load_fn=load_users)
        self.accounts_by_user = DataLoader(load_fn=load_accounts_by_user)
        self.members_by_family = DataLoader(load_fn=load_members_by_family)
        self.chores_by_family = DataLoader(load_fn=load_chores_by_family)
        self.assignments_by_chore = DataLoader(load_fn=load_assignments_by_chore)
        self.assignments_by_user = DataLoader(load_fn=load_assignments_by_user)
        self.transactions_by_account = DataLoader(load_fn=load_transactions_by_account)
        self.interest_by_account = DataLoader(load_fn=load_interest_by_account)
        self.chore = DataLoader(load_fn=load_chores)
```

---

## Frontend Apollo Client Setup

### Client Configuration

```javascript
// src/graphql/client.js

import { ApolloClient, InMemoryCache, split, HttpLink } from '@apollo/client';
import { GraphQLWsLink } from '@apollo/client/link/subscriptions';
import { getMainDefinition } from '@apollo/client/utilities';
import { createClient } from 'graphql-ws';
import { setContext } from '@apollo/client/link/context';

const httpLink = new HttpLink({
  uri: `${import.meta.env.VITE_API_URL}/graphql`,
});

const wsLink = new GraphQLWsLink(createClient({
  url: `${import.meta.env.VITE_WS_URL}/graphql`,
  connectionParams: () => ({
    authToken: localStorage.getItem('access_token'),
  }),
}));

const authLink = setContext((_, { headers }) => {
  const token = localStorage.getItem('access_token');
  return {
    headers: {
      ...headers,
      authorization: token ? `Bearer ${token}` : '',
      'x-family-id': localStorage.getItem('current_family_id'),
    }
  };
});

const splitLink = split(
  ({ query }) => {
    const definition = getMainDefinition(query);
    return (
      definition.kind === 'OperationDefinition' &&
      definition.operation === 'subscription'
    );
  },
  wsLink,
  authLink.concat(httpLink),
);

export const client = new ApolloClient({
  link: splitLink,
  cache: new InMemoryCache({
    typePolicies: {
      Account: {
        fields: {
          transactions: {
            merge(existing = [], incoming) {
              return [...incoming];
            },
          },
        },
      },
    },
  }),
});
```

### Example Queries

```javascript
// src/graphql/queries.js

import { gql } from '@apollo/client';

export const GET_DASHBOARD_DATA = gql`
  query GetDashboardData {
    me {
      id
      firstName
      ageBracket
      accounts {
        id
        name
        accountType
        balance
      }
    }
    choreAssignments(status: "PENDING") {
      id
      dueDate
      dueTime
      totalOverdueDays
      chore {
        id
        title
        rewardAmount
      }
    }
    savingsGoals(isActive: true) {
      id
      name
      targetAmount
      currentAmount
      progressPercent
      icon
      color
    }
  }
`;

export const GET_CHORES = gql`
  query GetChores($isActive: Boolean, $assignmentType: String) {
    chores(isActive: $isActive, assignmentType: $assignmentType) {
      id
      title
      description
      rewardAmount
      rewardType
      assignmentType
      assignments(status: "PENDING") {
        id
        dueDate
        status
        assignedTo {
          id
          firstName
        }
      }
    }
  }
`;

export const GET_ACCOUNT_DETAILS = gql`
  query GetAccountDetails($id: ID!, $transactionLimit: Int) {
    account(id: $id) {
      id
      name
      accountType
      balance
      interestConfig {
        annualRate
        compoundFrequency
      }
      transactions(limit: $transactionLimit) {
        id
        amount
        transactionType
        description
        balanceAfter
        createdAt
      }
    }
  }
`;
```

### Example Mutations

```javascript
// src/graphql/mutations.js

import { gql } from '@apollo/client';

export const COMPLETE_CHORE = gql`
  mutation CompleteChore($assignmentId: ID!, $photoUrl: String) {
    completeChore(assignmentId: $assignmentId, photoUrl: $photoUrl) {
      id
      status
      completedAt
      chore {
        id
        title
        rewardAmount
      }
    }
  }
`;

export const APPROVE_CHORE = gql`
  mutation ApproveChore($assignmentId: ID!, $feedback: String) {
    approveChore(assignmentId: $assignmentId, feedback: $feedback) {
      id
      status
      approvedAt
    }
  }
`;

export const CREATE_TRANSACTION = gql`
  mutation CreateTransaction(
    $accountId: ID!
    $amount: Decimal!
    $description: String!
    $transactionType: String
  ) {
    createTransaction(
      accountId: $accountId
      amount: $amount
      description: $description
      transactionType: $transactionType
    ) {
      id
      amount
      description
      balanceAfter
      createdAt
    }
  }
`;
```

### Example Subscriptions

```javascript
// src/graphql/subscriptions.js

import { gql } from '@apollo/client';

export const CHORE_UPDATES_SUBSCRIPTION = gql`
  subscription ChoreUpdates($familyId: ID!) {
    choreUpdates(familyId: $familyId) {
      id
      status
      completedAt
      approvedAt
      chore {
        id
        title
      }
      assignedTo {
        id
        firstName
      }
    }
  }
`;

export const NOTIFICATIONS_SUBSCRIPTION = gql`
  subscription Notifications {
    notifications {
      id
      type
      message
      createdAt
      data
    }
  }
`;
```

### Hook Usage

```jsx
// components/Dashboard.jsx

import { useQuery, useSubscription } from '@apollo/client';
import { GET_DASHBOARD_DATA } from '@/graphql/queries';
import { CHORE_UPDATES_SUBSCRIPTION } from '@/graphql/subscriptions';

function Dashboard() {
  const { data, loading, error, refetch } = useQuery(GET_DASHBOARD_DATA);
  
  // Subscribe to real-time updates
  useSubscription(CHORE_UPDATES_SUBSCRIPTION, {
    variables: { familyId: currentFamily.id },
    onData: ({ data }) => {
      toast.info(`Chore update: ${data.choreUpdates.chore.title}`);
      refetch();
    }
  });
  
  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;
  
  const { me, choreAssignments, savingsGoals } = data;
  
  return (
    <div className={`ui-${me.ageBracket}`}>
      <AccountCards accounts={me.accounts} />
      <ChoreList assignments={choreAssignments} />
      <GoalCards goals={savingsGoals} />
    </div>
  );
}
```

---

## Optimistic Updates

```jsx
// Example: Optimistic chore completion

import { useMutation } from '@apollo/client';
import { COMPLETE_CHORE } from '@/graphql/mutations';
import { GET_DASHBOARD_DATA } from '@/graphql/queries';

function ChoreCard({ assignment }) {
  const [completeChore] = useMutation(COMPLETE_CHORE, {
    optimisticResponse: {
      completeChore: {
        __typename: 'ChoreAssignment',
        id: assignment.id,
        status: 'COMPLETED',
        completedAt: new Date().toISOString(),
        chore: assignment.chore,
      }
    },
    update(cache, { data: { completeChore } }) {
      // Update the cache with the completed status
      cache.modify({
        id: cache.identify(completeChore),
        fields: {
          status: () => 'COMPLETED',
          completedAt: () => completeChore.completedAt,
        }
      });
    },
    refetchQueries: [{ query: GET_DASHBOARD_DATA }],
  });
  
  return (
    <button onClick={() => completeChore({ variables: { assignmentId: assignment.id } })}>
      Complete
    </button>
  );
}
```

---

## Testing Requirements

### Backend GraphQL Tests

```python
def test_dashboard_query():
    """Test dashboard data query."""
    query = """
        query {
            me {
                id
                firstName
                accounts {
                    id
                    balance
                }
            }
            choreAssignments(status: "PENDING") {
                id
                chore {
                    title
                }
            }
        }
    """
    
    result = execute_graphql(query, context=user_context)
    
    assert result.data['me']['id'] == str(user.id)
    assert len(result.data['me']['accounts']) > 0
    assert result.errors is None

def test_complete_chore_mutation():
    """Test chore completion mutation."""
    assignment = create_pending_assignment(user)
    
    mutation = """
        mutation CompleteChore($assignmentId: ID!) {
            completeChore(assignmentId: $assignmentId) {
                id
                status
                completedAt
            }
        }
    """
    
    result = execute_graphql(
        mutation,
        variables={'assignmentId': str(assignment.id)},
        context=user_context
    )
    
    assert result.data['completeChore']['status'] == 'COMPLETED'
    assert result.data['completeChore']['completedAt'] is not None
```

### Coverage Requirements

- GraphQL queries: 80%
- GraphQL mutations: 100%
- Subscriptions: 80%
- DataLoaders: 80%

---

## Definition of Done

### Must Have
- [ ] Strawberry GraphQL schema defined
- [ ] All existing REST functionality available via GraphQL
- [ ] DataLoaders preventing N+1 queries
- [ ] Apollo Client configured in frontend
- [ ] Dashboard migrated to GraphQL
- [ ] Real-time subscriptions working
- [ ] GraphQL Playground available in dev

### Nice to Have
- [ ] Optimistic updates for all mutations
- [ ] Query result caching
- [ ] Automatic query retries
- [ ] GraphQL code generation

---

## Migration Strategy

1. **Week 1**: Backend GraphQL setup
   - Schema definition
   - Resolvers
   - DataLoaders
   - Keep REST endpoints active

2. **Week 2**: Frontend migration
   - Apollo Client setup
   - Migrate dashboard to GraphQL
   - Add subscriptions
   - Test both APIs

3. **Post-Phase**: Gradual migration
   - Migrate remaining pages
   - Deprecate REST endpoints (not remove)
   - Monitor performance

---

## Dependencies

- **Requires**: Phase 1-6 (existing features to expose)
- **Optional**: Can skip if GraphQL complexity not justified

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Increased complexity | Keep REST as fallback |
| Performance issues | DataLoaders, query complexity limits |
| Learning curve | Thorough documentation, examples |
| Subscription scalability | Redis pub/sub for multi-instance |

---

*Phase 8 Document - Last Updated: February 2026*
