### Core Tables

#### users
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255), -- NULL if social auth only
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    birthdate DATE, -- Required for kids, optional for parents
    role VARCHAR(20) NOT NULL CHECK (role IN ('PARENT', 'CHILD')),
    timezone VARCHAR(50) DEFAULT 'America/New_York',
    email_verified BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
```

#### social_auth_providers
```sql
CREATE TABLE social_auth_providers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider VARCHAR(50) NOT NULL CHECK (provider IN ('google', 'apple', 'facebook')),
    provider_user_id VARCHAR(255) NOT NULL,
    access_token TEXT, -- Encrypted
    refresh_token TEXT, -- Encrypted
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(provider, provider_user_id)
);

CREATE INDEX idx_social_auth_user ON social_auth_providers(user_id);
CREATE INDEX idx_social_auth_provider ON social_auth_providers(provider, provider_user_id);
```

#### families
```sql
CREATE TABLE families (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    timezone VARCHAR(50) DEFAULT 'America/New_York',
    currency VARCHAR(3) DEFAULT 'USD',
    settings JSONB DEFAULT '{}', -- Customizable family settings
    subscription_tier VARCHAR(50) DEFAULT 'free', -- free, premium
    trial_ends_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_families_subscription ON families(subscription_tier);
```

#### family_memberships
```sql
CREATE TABLE family_memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    role VARCHAR(20) NOT NULL CHECK (role IN ('PARENT', 'CHILD')),
    is_active BOOLEAN DEFAULT TRUE,
    age_bracket_override VARCHAR(20), -- 'young', 'tween', 'teen', NULL = auto
    joined_at TIMESTAMP DEFAULT NOW(),
    ended_at TIMESTAMP,
    UNIQUE(user_id, family_id)
);

CREATE INDEX idx_family_memberships_user ON family_memberships(user_id);
CREATE INDEX idx_family_memberships_family ON family_memberships(family_id);
CREATE INDEX idx_family_memberships_active ON family_memberships(is_active) WHERE is_active = TRUE;
```

#### accounts
```sql
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    account_type VARCHAR(50) NOT NULL CHECK (account_type IN ('SPENDING', 'SAVING', 'GIVING', 'INVESTMENT', 'LOAN')),
    balance NUMERIC(12, 2) DEFAULT 0.00, -- Cents precision
    allow_negative BOOLEAN DEFAULT FALSE,
    negative_limit NUMERIC(12, 2), -- NULL = no limit if negatives allowed
    interest_rate NUMERIC(5, 4), -- Annual rate, e.g., 0.0500 = 5%
    interest_frequency VARCHAR(20), -- 'weekly', 'monthly', 'yearly'
    interest_cap NUMERIC(12, 2), -- Max interest per period
    is_hidden BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(family_id, user_id, name)
);

CREATE INDEX idx_accounts_family ON accounts(family_id);
CREATE INDEX idx_accounts_user ON accounts(user_id);
CREATE INDEX idx_accounts_type ON accounts(account_type);
```

#### transactions
```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    transaction_type VARCHAR(50) NOT NULL CHECK (transaction_type IN (
        'CREDIT', 'DEBIT', 'TRANSFER_IN', 'TRANSFER_OUT', 
        'ALLOWANCE', 'CHORE_REWARD', 'INTEREST', 'PENALTY', 'ADJUSTMENT'
    )),
    amount NUMERIC(12, 2) NOT NULL,
    balance_after NUMERIC(12, 2) NOT NULL, -- Snapshot for audit
    description TEXT,
    related_transaction_id UUID REFERENCES transactions(id), -- For transfers
    related_chore_id UUID, -- References chores(id) but no FK for flexibility
    related_allowance_id UUID, -- References allowances(id)
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_transactions_family ON transactions(family_id);
CREATE INDEX idx_transactions_account ON transactions(account_id);
CREATE INDEX idx_transactions_created_at ON transactions(created_at DESC);
CREATE INDEX idx_transactions_type ON transactions(transaction_type);
```

#### chores
```sql
CREATE TABLE chores (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    reward_amount NUMERIC(12, 2), -- NULL = non-monetary
    reward_type VARCHAR(20) DEFAULT 'money' CHECK (reward_type IN ('money', 'points', 'stars')),
    penalty_amount NUMERIC(12, 2),
    assignment_type VARCHAR(20) NOT NULL CHECK (assignment_type IN ('ASSIGNED', 'FIRST_DIBS')),
    recurrence_rule JSONB, -- iCalendar RRULE + custom fields
    timing_mode VARCHAR(20) DEFAULT 'absolute' CHECK (timing_mode IN ('absolute', 'relative')),
    deadline_time TIME, -- e.g., "8:00 AM"
    estimated_duration INTEGER, -- Minutes
    require_photo BOOLEAN DEFAULT FALSE,
    require_approval BOOLEAN DEFAULT TRUE,
    auto_approve_after INTEGER, -- Hours, NULL = never auto-approve
    is_template BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_chores_family ON chores(family_id);
CREATE INDEX idx_chores_active ON chores(is_active) WHERE is_active = TRUE;
CREATE INDEX idx_chores_template ON chores(is_template) WHERE is_template = TRUE;
```

#### chore_assignments
```sql
CREATE TABLE chore_assignments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chore_id UUID NOT NULL REFERENCES chores(id) ON DELETE CASCADE,
    assigned_to_user_id UUID REFERENCES users(id) ON DELETE SET NULL, -- NULL for first-dibs
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    due_date DATE NOT NULL,
    due_time TIME,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING' CHECK (status IN (
        'PENDING', 'AVAILABLE', 'CLAIMED', 'COMPLETED', 'APPROVED', 'REJECTED', 'EXPIRED'
    )),
    claimed_by_user_id UUID REFERENCES users(id),
    claimed_at TIMESTAMP,
    completed_at TIMESTAMP,
    completed_by_user_id UUID REFERENCES users(id),
    approved_at TIMESTAMP,
    approved_by_user_id UUID REFERENCES users(id),
    rejected_at TIMESTAMP,
    rejection_reason TEXT,
    total_overdue_days INTEGER DEFAULT 0, -- Cumulative overdue time
    photo_urls TEXT[], -- Array of S3 URLs
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_chore_assignments_chore ON chore_assignments(chore_id);
CREATE INDEX idx_chore_assignments_assigned_to ON chore_assignments(assigned_to_user_id);
CREATE INDEX idx_chore_assignments_status ON chore_assignments(status);
CREATE INDEX idx_chore_assignments_due_date ON chore_assignments(due_date);
CREATE INDEX idx_chore_assignments_family ON chore_assignments(family_id);
```

#### allowances
```sql
CREATE TABLE allowances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    amount NUMERIC(12, 2) NOT NULL, -- Or formula
    amount_formula VARCHAR(255), -- e.g., "age * 2" for age-based
    frequency VARCHAR(20) NOT NULL CHECK (frequency IN ('weekly', 'biweekly', 'monthly', 'semimonthly')),
    day_of_week INTEGER, -- 0-6 for weekly
    day_of_month INTEGER, -- 1-31 for monthly
    start_date DATE NOT NULL,
    end_date DATE, -- NULL = indefinite
    split_id UUID, -- References splits(id)
    is_active BOOLEAN DEFAULT TRUE,
    next_payment_date DATE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_allowances_family ON allowances(family_id);
CREATE INDEX idx_allowances_user ON allowances(user_id);
CREATE INDEX idx_allowances_next_payment ON allowances(next_payment_date) WHERE is_active = TRUE;
```

#### splits
```sql
CREATE TABLE splits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    percentages JSONB NOT NULL, -- {"spending": 60, "saving": 20, "giving": 20}
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(family_id, user_id, name)
);

CREATE INDEX idx_splits_family ON splits(family_id);
CREATE INDEX idx_splits_user ON splits(user_id);
```

#### savings_goals
```sql
CREATE TABLE savings_goals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    account_id UUID NOT NULL REFERENCES accounts(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    target_amount NUMERIC(12, 2) NOT NULL,
    target_date DATE,
    image_url TEXT,
    is_completed BOOLEAN DEFAULT FALSE,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_savings_goals_account ON savings_goals(account_id);
CREATE INDEX idx_savings_goals_family ON savings_goals(family_id);
```

#### budgets
```sql
CREATE TABLE budgets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    categories JSONB NOT NULL, -- {"food": 100, "entertainment": 50}
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_budgets_family ON budgets(family_id);
CREATE INDEX idx_budgets_user ON budgets(user_id);
```

#### notifications
```sql
CREATE TABLE notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    family_id UUID NOT NULL REFERENCES families(id) ON DELETE CASCADE,
    notification_type VARCHAR(50) NOT NULL,
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    related_entity_type VARCHAR(50), -- 'chore', 'transaction', 'allowance'
    related_entity_id UUID,
    is_read BOOLEAN DEFAULT FALSE,
    sent_via_push BOOLEAN DEFAULT FALSE,
    sent_via_email BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_notifications_user ON notifications(user_id);
CREATE INDEX idx_notifications_unread ON notifications(user_id, is_read) WHERE is_read = FALSE;
CREATE INDEX idx_notifications_created ON notifications(created_at DESC);
```

#### audit_log
```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    family_id UUID REFERENCES families(id) ON DELETE SET NULL,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    action VARCHAR(100) NOT NULL,
    entity_type VARCHAR(50) NOT NULL,
    entity_id UUID,
    changes JSONB, -- Before/after values
    ip_address INET,
    user_agent TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_log_family ON audit_log(family_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_created ON audit_log(created_at DESC);
```

### Row-Level Security Policies

```sql
-- Enable RLS on all family-scoped tables
ALTER TABLE families ENABLE ROW LEVEL SECURITY;
ALTER TABLE family_memberships ENABLE ROW LEVEL SECURITY;
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;
ALTER TABLE chores ENABLE ROW LEVEL SECURITY;
ALTER TABLE chore_assignments ENABLE ROW LEVEL SECURITY;
ALTER TABLE allowances ENABLE ROW LEVEL SECURITY;
ALTER TABLE splits ENABLE ROW LEVEL SECURITY;
ALTER TABLE savings_goals ENABLE ROW LEVEL SECURITY;
ALTER TABLE budgets ENABLE ROW LEVEL SECURITY;
ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

-- Example policy for chores table
CREATE POLICY chores_family_isolation ON chores
    FOR ALL
    USING (family_id = current_setting('app.current_family_id', TRUE)::uuid);

-- Repeat for all tables with family_id
```
