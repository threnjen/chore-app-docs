# Frontend Component Architecture

## Overview

PicklesApp's frontend architecture follows a modular, reusable component design pattern built with React 19+. The architecture emphasizes:

- **Age-adaptive rendering** for three child age brackets
- **Component composition** over inheritance
- **Shared components** with variant support
- **Code splitting** for optimal performance
- **Clear separation** between presentational and container components

## Table of Contents
1. [Component Hierarchy](#component-hierarchy)
2. [Directory Structure](#directory-structure)
3. [Component Categories](#component-categories)
4. [Component Composition Patterns](#component-composition-patterns)
5. [Props Patterns and Conventions](#props-patterns-and-conventions)
6. [Age-Adaptive Component Wrappers](#age-adaptive-component-wrappers)
7. [Reusability Strategies](#reusability-strategies)
8. [Code Splitting and Lazy Loading](#code-splitting-and-lazy-loading)

---

## Component Hierarchy

```
App
├── AuthProvider (Context)
├── FamilyProvider (Context)
├── UserProvider (Context)
└── Router
    ├── PublicLayout
    │   ├── Header
    │   ├── PublicRoutes
    │   │   ├── Landing
    │   │   ├── Login
    │   │   ├── Register
    │   │   └── ResetPassword
    │   └── Footer
    │
    └── PrivateLayout
        ├── Navigation (age-adaptive)
        │   ├── ParentNavigation (sidebar + header)
        │   ├── TeenNavigation (sidebar)
        │   ├── TweenNavigation (bottom nav + header)
        │   └── YoungNavigation (bottom nav + icons)
        │
        ├── FamilySwitcher (multi-household)
        │
        └── PrivateRoutes
            ├── Dashboard (age-adaptive)
            │   ├── ParentDashboard
            │   ├── TeenDashboard
            │   ├── TweenDashboard
            │   └── YoungDashboard
            │
            ├── Chores
            │   ├── ChoreList
            │   ├── ChoreDetail
            │   ├── ChoreForm (parent only)
            │   └── ChoreApproval (parent only)
            │
            ├── Money
            │   ├── AccountsList
            │   ├── AccountDetail
            │   ├── TransactionHistory
            │   └── TransferForm
            │
            ├── Allowances (parent only)
            │   ├── AllowanceList
            │   └── AllowanceForm
            │
            ├── Goals
            │   ├── GoalList
            │   └── GoalForm
            │
            └── Settings
                ├── ProfileSettings
                ├── FamilySettings (parent only)
                └── NotificationSettings
```

---

## Directory Structure

```
src/
├── components/
│   ├── auth/                    # Authentication components
│   │   ├── LoginForm.jsx
│   │   ├── RegisterForm.jsx
│   │   ├── SocialAuthButtons.jsx
│   │   └── PasswordResetForm.jsx
│   │
│   ├── chores/                  # Chore-related components
│   │   ├── ChoreList.jsx
│   │   ├── ChoreCard.jsx
│   │   ├── ChoreDetail.jsx
│   │   ├── ChoreForm.jsx
│   │   ├── RecurrenceBuilder.jsx
│   │   ├── RecurrencePreview.jsx
│   │   ├── CompletionModal.jsx
│   │   └── ApprovalQueue.jsx
│   │
│   ├── money/                   # Money management components
│   │   ├── AccountCard.jsx
│   │   ├── AccountsList.jsx
│   │   ├── AccountDetail.jsx
│   │   ├── TransactionList.jsx
│   │   ├── TransactionItem.jsx
│   │   ├── TransferForm.jsx
│   │   └── BalanceSummary.jsx
│   │
│   ├── allowances/              # Allowance components (parent only)
│   │   ├── AllowanceList.jsx
│   │   ├── AllowanceCard.jsx
│   │   ├── AllowanceForm.jsx
│   │   └── NextPaymentPreview.jsx
│   │
│   ├── goals/                   # Savings goals components
│   │   ├── GoalList.jsx
│   │   ├── GoalCard.jsx
│   │   ├── GoalDetail.jsx
│   │   ├── GoalForm.jsx
│   │   └── GoalProgress.jsx
│   │
│   ├── layout/                  # Layout components
│   │   ├── PublicLayout.jsx
│   │   ├── PrivateLayout.jsx
│   │   ├── Header.jsx
│   │   ├── Footer.jsx
│   │   ├── Sidebar.jsx
│   │   ├── BottomNav.jsx
│   │   ├── FamilySwitcher.jsx
│   │   └── Navigation.jsx
│   │
│   ├── adaptive/                # Age-adaptive components
│   │   ├── AdaptiveComponent.jsx
│   │   ├── AdaptiveDashboard.jsx
│   │   ├── AdaptiveChoreCard.jsx
│   │   ├── AdaptiveButton.jsx
│   │   ├── FeatureGate.jsx
│   │   └── AgeUpModal.jsx
│   │
│   └── shared/                  # Shared/reusable components
│       ├── Button.jsx
│       ├── Input.jsx
│       ├── Select.jsx
│       ├── Textarea.jsx
│       ├── Modal.jsx
│       ├── Card.jsx
│       ├── Badge.jsx
│       ├── LoadingSpinner.jsx
│       ├── EmptyState.jsx
│       ├── ErrorBoundary.jsx
│       ├── ConfirmDialog.jsx
│       └── Toast.jsx
│
├── pages/                       # Page components (route containers)
│   ├── Landing.jsx
│   ├── Login.jsx
│   ├── Register.jsx
│   ├── Dashboard.jsx
│   ├── Chores.jsx
│   ├── ChoreDetail.jsx
│   ├── Money.jsx
│   ├── AccountDetail.jsx
│   ├── Allowances.jsx
│   ├── Goals.jsx
│   └── Settings.jsx
│
├── contexts/                    # React Context providers
│   ├── AuthContext.jsx
│   ├── FamilyContext.jsx
│   └── UserContext.jsx
│
├── hooks/                       # Custom React hooks
│   ├── useAuth.js
│   ├── useFamily.js
│   ├── useUser.js
│   ├── useAgeBracket.js
│   ├── useApi.js
│   ├── useChores.js
│   ├── useAccounts.js
│   └── useTransactions.js
│
├── services/                    # API service layer
│   ├── api.js                   # Axios instance + interceptors
│   ├── authService.js
│   ├── choreService.js
│   ├── accountService.js
│   ├── transactionService.js
│   ├── allowanceService.js
│   └── goalService.js
│
└── utils/                       # Utility functions
    ├── formatters.js            # Date, currency formatting
    ├── validators.js            # Form validation helpers
    ├── constants.js             # App-wide constants
    └── featureAccess.js         # Age-based feature gating
```

---

## Component Categories

### 1. Presentational Components

**Purpose:** Pure rendering, no state management, receive all data via props.

**Characteristics:**
- Focused on UI rendering
- Highly reusable
- Easy to test
- No business logic

**Examples:**

```jsx
// components/shared/Button.jsx
export function Button({ 
  children, 
  variant = 'primary', 
  size = 'md',
  disabled = false,
  onClick,
  ...props 
}) {
  const baseClasses = 'rounded font-medium transition-colors';
  const variantClasses = {
    primary: 'bg-blue-600 text-white hover:bg-blue-700',
    secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
    danger: 'bg-red-600 text-white hover:bg-red-700',
  };
  const sizeClasses = {
    sm: 'px-3 py-1.5 text-sm',
    md: 'px-4 py-2 text-base',
    lg: 'px-6 py-3 text-lg',
  };

  return (
    <button
      className={`${baseClasses} ${variantClasses[variant]} ${sizeClasses[size]}`}
      disabled={disabled}
      onClick={onClick}
      {...props}
    >
      {children}
    </button>
  );
}
```

```jsx
// components/chores/ChoreCard.jsx
export function ChoreCard({ chore, onComplete, onView }) {
  return (
    <Card className="hover:shadow-lg transition-shadow">
      <div className="flex items-start justify-between">
        <div className="flex-1">
          <h3 className="text-lg font-semibold">{chore.title}</h3>
          {chore.description && (
            <p className="text-gray-600 text-sm mt-1">{chore.description}</p>
          )}
        </div>
        {chore.reward_amount && (
          <Badge variant="success">${chore.reward_amount.toFixed(2)}</Badge>
        )}
      </div>
      
      {chore.due_date && (
        <div className="mt-3 text-sm text-gray-500">
          Due: {formatDate(chore.due_date)}
        </div>
      )}
      
      {chore.total_overdue_days > 0 && (
        <Badge variant="danger" className="mt-2">
          {chore.total_overdue_days} days overdue
        </Badge>
      )}
      
      <div className="mt-4 flex gap-2">
        <Button onClick={() => onView(chore.id)} variant="secondary" size="sm">
          View
        </Button>
        {chore.status === 'PENDING' && (
          <Button onClick={() => onComplete(chore.id)} size="sm">
            Mark Complete
          </Button>
        )}
      </div>
    </Card>
  );
}
```

### 2. Container Components

**Purpose:** Manage state, fetch data, handle business logic, pass data to presentational components.

**Characteristics:**
- Connect to Context or hooks
- Handle API calls
- Manage local state
- Pass data down to presentational components

**Example:**

```jsx
// pages/Chores.jsx
import { useState, useEffect } from 'react';
import { useFamily } from '@/contexts/FamilyContext';
import { useUser } from '@/contexts/UserContext';
import { choreService } from '@/services/choreService';
import { ChoreList } from '@/components/chores/ChoreList';
import { LoadingSpinner } from '@/components/shared/LoadingSpinner';
import { EmptyState } from '@/components/shared/EmptyState';
import toast from 'react-hot-toast';

export function ChoresPage() {
  const { currentFamily } = useFamily();
  const { user } = useUser();
  const [chores, setChores] = useState([]);
  const [loading, setLoading] = useState(true);
  const [filter, setFilter] = useState('all');

  useEffect(() => {
    if (currentFamily) {
      fetchChores();
    }
  }, [currentFamily, filter]);

  const fetchChores = async () => {
    try {
      setLoading(true);
      const data = await choreService.getChores(currentFamily.id, {
        status: filter === 'all' ? undefined : filter,
        assigned_to: user.role === 'CHILD' ? user.id : undefined,
      });
      setChores(data);
    } catch (error) {
      toast.error('Failed to load chores');
    } finally {
      setLoading(false);
    }
  };

  const handleCompleteChore = async (choreId) => {
    try {
      await choreService.completeChore(choreId);
      toast.success('Chore marked complete!');
      fetchChores();
    } catch (error) {
      toast.error('Failed to complete chore');
    }
  };

  if (loading) {
    return <LoadingSpinner />;
  }

  if (chores.length === 0) {
    return (
      <EmptyState
        title="No chores yet"
        description="Your chores will appear here once they're assigned."
      />
    );
  }

  return (
    <div className="container mx-auto px-4 py-8">
      <ChoreList
        chores={chores}
        filter={filter}
        onFilterChange={setFilter}
        onComplete={handleCompleteChore}
      />
    </div>
  );
}
```

### 3. Layout Components

**Purpose:** Define page structure, navigation, headers/footers.

**Example:**

```jsx
// components/layout/PrivateLayout.jsx
import { Outlet } from 'react-router-dom';
import { useUser } from '@/contexts/UserContext';
import { ParentNavigation } from './ParentNavigation';
import { TeenNavigation } from './TeenNavigation';
import { TweenNavigation } from './TweenNavigation';
import { YoungNavigation } from './YoungNavigation';
import { FamilySwitcher } from './FamilySwitcher';

export function PrivateLayout() {
  const { user, ageBracket } = useUser();

  const Navigation = () => {
    if (user.role === 'PARENT') {
      return <ParentNavigation />;
    }
    
    switch (ageBracket) {
      case 'teen':
        return <TeenNavigation />;
      case 'tween':
        return <TweenNavigation />;
      case 'young':
        return <YoungNavigation />;
      default:
        return <TweenNavigation />;
    }
  };

  return (
    <div className="min-h-screen bg-gray-50">
      <Navigation />
      
      {user.role === 'CHILD' && user.families.length > 1 && (
        <FamilySwitcher />
      )}
      
      <main className={ageBracket === 'teen' ? 'ml-64' : ''}>
        <Outlet />
      </main>
    </div>
  );
}
```

### 4. Form Components

**Purpose:** Handle user input with validation and submission.

**Best Practices:**
- Use React Hook Form for complex forms
- Use Zod for validation schemas
- Provide clear error messages
- Disable submit while loading

**Example:**

```jsx
// components/chores/ChoreForm.jsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { Button } from '@/components/shared/Button';
import { Input } from '@/components/shared/Input';
import { Select } from '@/components/shared/Select';
import { RecurrenceBuilder } from './RecurrenceBuilder';

const choreSchema = z.object({
  title: z.string().min(3, 'Title must be at least 3 characters'),
  description: z.string().optional(),
  reward_amount: z.number().min(0, 'Reward must be positive'),
  assignment_type: z.enum(['ASSIGNED', 'FIRST_DIBS']),
  assigned_to_id: z.string().uuid().optional(),
  recurrence_rule: z.object({
    frequency: z.enum(['DAILY', 'WEEKLY', 'MONTHLY']),
    interval: z.number().min(1),
    timing_mode: z.enum(['absolute', 'relative']),
    // ... more fields
  }),
});

export function ChoreForm({ chore, onSubmit, onCancel }) {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    control,
  } = useForm({
    resolver: zodResolver(choreSchema),
    defaultValues: chore || {},
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
      <Input
        label="Chore Title"
        {...register('title')}
        error={errors.title?.message}
      />
      
      <Input
        label="Description"
        as="textarea"
        {...register('description')}
        error={errors.description?.message}
      />
      
      <Input
        label="Reward Amount"
        type="number"
        step="0.01"
        {...register('reward_amount', { valueAsNumber: true })}
        error={errors.reward_amount?.message}
      />
      
      <Select
        label="Assignment Type"
        {...register('assignment_type')}
        error={errors.assignment_type?.message}
      >
        <option value="ASSIGNED">Assign to specific child</option>
        <option value="FIRST_DIBS">First to claim</option>
      </Select>
      
      <RecurrenceBuilder control={control} />
      
      <div className="flex gap-3 justify-end">
        <Button variant="secondary" onClick={onCancel} type="button">
          Cancel
        </Button>
        <Button type="submit" disabled={isSubmitting}>
          {isSubmitting ? 'Saving...' : 'Save Chore'}
        </Button>
      </div>
    </form>
  );
}
```

---

## Component Composition Patterns

### 1. Compound Components

Components that work together as a cohesive unit.

**Example:**

```jsx
// components/shared/Card.jsx
export function Card({ children, className = '', ...props }) {
  return (
    <div
      className={`bg-white rounded-lg shadow-md p-6 ${className}`}
      {...props}
    >
      {children}
    </div>
  );
}

Card.Header = function CardHeader({ children, className = '' }) {
  return (
    <div className={`border-b border-gray-200 pb-4 mb-4 ${className}`}>
      {children}
    </div>
  );
};

Card.Body = function CardBody({ children, className = '' }) {
  return <div className={className}>{children}</div>;
};

Card.Footer = function CardFooter({ children, className = '' }) {
  return (
    <div className={`border-t border-gray-200 pt-4 mt-4 ${className}`}>
      {children}
    </div>
  );
};

// Usage
<Card>
  <Card.Header>
    <h2>Account Balance</h2>
  </Card.Header>
  <Card.Body>
    <p className="text-3xl font-bold">$125.50</p>
  </Card.Body>
  <Card.Footer>
    <Button>View Transactions</Button>
  </Card.Footer>
</Card>
```

### 2. Render Props

Pass rendering logic to components for flexibility.

**Example:**

```jsx
// components/shared/DataFetcher.jsx
export function DataFetcher({ fetchFn, children }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchFn()
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, [fetchFn]);

  return children({ data, loading, error });
}

// Usage
<DataFetcher fetchFn={() => choreService.getChores()}>
  {({ data, loading, error }) => {
    if (loading) return <LoadingSpinner />;
    if (error) return <ErrorMessage error={error} />;
    return <ChoreList chores={data} />;
  }}
</DataFetcher>
```

### 3. Higher-Order Components (HOCs)

Enhance components with additional functionality.

**Example:**

```jsx
// components/shared/withAuth.jsx
export function withAuth(Component, requiredRole = null) {
  return function AuthenticatedComponent(props) {
    const { user, loading } = useAuth();
    const navigate = useNavigate();

    useEffect(() => {
      if (!loading && !user) {
        navigate('/login');
      }
      if (requiredRole && user?.role !== requiredRole) {
        navigate('/unauthorized');
      }
    }, [user, loading, navigate]);

    if (loading) {
      return <LoadingSpinner />;
    }

    if (!user) {
      return null;
    }

    return <Component {...props} />;
  };
}

// Usage
export const AdminSettings = withAuth(AdminSettingsComponent, 'PARENT');
```

---

## Props Patterns and Conventions

### 1. Prop Naming Conventions

```jsx
// Event handlers: on[EventName]
<Button onClick={handleClick} onHover={handleHover} />

// Boolean props: is[State] or has[Feature]
<ChoreCard isCompleted={true} hasReward={true} />

// Render props: render[ComponentName]
<List renderItem={(item) => <Item {...item} />} />

// Callbacks: on[Action]Complete
<ChoreForm onSubmitComplete={handleSuccess} />

// Data props: [entity] or [entity]Data
<ChoreCard chore={choreData} />
```

### 2. Default Props Pattern

```jsx
export function Badge({ 
  children, 
  variant = 'default',
  size = 'md',
  className = '',
}) {
  const variants = {
    default: 'bg-gray-100 text-gray-800',
    success: 'bg-green-100 text-green-800',
    danger: 'bg-red-100 text-red-800',
  };

  const sizes = {
    sm: 'px-2 py-0.5 text-xs',
    md: 'px-3 py-1 text-sm',
    lg: 'px-4 py-2 text-base',
  };

  return (
    <span
      className={`
        inline-flex items-center rounded-full font-medium
        ${variants[variant]}
        ${sizes[size]}
        ${className}
      `}
    >
      {children}
    </span>
  );
}
```

### 3. Spread Props Pattern

```jsx
export function Input({ 
  label, 
  error, 
  helperText,
  className = '',
  ...inputProps 
}) {
  return (
    <div className={`mb-4 ${className}`}>
      {label && (
        <label className="block text-sm font-medium mb-2">
          {label}
        </label>
      )}
      <input
        className={`
          w-full px-3 py-2 border rounded-md
          ${error ? 'border-red-500' : 'border-gray-300'}
        `}
        {...inputProps}
      />
      {error && <p className="text-red-500 text-sm mt-1">{error}</p>}
      {helperText && <p className="text-gray-500 text-sm mt-1">{helperText}</p>}
    </div>
  );
}
```

### 4. Children Pattern

```jsx
// String children
<Button>Click Me</Button>

// Component children
<Card>
  <h2>Title</h2>
  <p>Content</p>
</Card>

// Function children (render prop)
<List>
  {(items) => items.map(item => <Item key={item.id} {...item} />)}
</List>

// Conditional children
<Modal>
  {showContent && <ModalContent />}
</Modal>
```

---

## Age-Adaptive Component Wrappers

### AdaptiveComponent

Core wrapper for age-based rendering.

```jsx
// components/adaptive/AdaptiveComponent.jsx
import { useUser } from '@/contexts/UserContext';
import { FeatureGate } from './FeatureGate';

export function AdaptiveComponent({ 
  children, 
  feature,
  youngComponent: Young,
  tweenComponent: Tween,
  teenComponent: Teen,
  renderFallback,
}) {
  const { user, ageBracket } = useUser();

  // Feature gating
  if (feature) {
    return (
      <FeatureGate feature={feature}>
        <AdaptiveComponent
          youngComponent={Young}
          tweenComponent={Tween}
          teenComponent={Teen}
          renderFallback={renderFallback}
        />
      </FeatureGate>
    );
  }

  // Render age-appropriate component
  const components = {
    young: Young,
    tween: Tween,
    teen: Teen,
  };

  const Component = components[ageBracket];

  if (!Component) {
    return renderFallback ? renderFallback() : null;
  }

  return <Component />;
}

// Usage
<AdaptiveComponent
  feature="view_transaction_history"
  youngComponent={SimpleTransactionList}
  tweenComponent={TransactionList}
  teenComponent={TransactionTable}
/>
```

### AdaptiveDashboard

Age-specific dashboard layouts.

```jsx
// components/adaptive/AdaptiveDashboard.jsx
import { AdaptiveComponent } from './AdaptiveComponent';
import { YoungDashboard } from './YoungDashboard';
import { TweenDashboard } from './TweenDashboard';
import { TeenDashboard } from './TeenDashboard';

export function AdaptiveDashboard() {
  return (
    <AdaptiveComponent
      youngComponent={YoungDashboard}
      tweenComponent={TweenDashboard}
      teenComponent={TeenDashboard}
    />
  );
}
```

### FeatureGate

Control feature access by age bracket.

```jsx
// components/adaptive/FeatureGate.jsx
import { useUser } from '@/contexts/UserContext';
import { FEATURE_ACCESS } from '@/utils/featureAccess';

export function FeatureGate({ feature, children, fallback = null }) {
  const { user, ageBracket } = useUser();

  // Parents have access to all features
  if (user.role === 'PARENT') {
    return children;
  }

  // Check feature access for child's age bracket
  const hasAccess = FEATURE_ACCESS[ageBracket]?.[feature] ?? false;

  if (!hasAccess) {
    if (fallback) {
      return fallback;
    }
    return (
      <div className="text-center py-8">
        <p className="text-gray-500">
          This feature is not available for your age group yet.
        </p>
      </div>
    );
  }

  return children;
}

// Usage
<FeatureGate feature="transfer_between_accounts">
  <TransferForm />
</FeatureGate>
```

---

## Reusability Strategies

### 1. Extract Common Patterns

```jsx
// Before: Repeated code
function ChoreList() {
  const [loading, setLoading] = useState(true);
  const [data, setData] = useState([]);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchChores()
      .then(setData)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);

  // ...
}

// After: Custom hook
function useChores() {
  const [loading, setLoading] = useState(true);
  const [chores, setChores] = useState([]);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetchChores()
      .then(setChores)
      .catch(setError)
      .finally(() => setLoading(false));
  }, []);

  return { chores, loading, error };
}

// Usage
function ChoreList() {
  const { chores, loading, error } = useChores();
  // ...
}
```

### 2. Component Variants

```jsx
// components/shared/Button.jsx with variants
const VARIANTS = {
  primary: 'bg-blue-600 text-white hover:bg-blue-700',
  secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
  danger: 'bg-red-600 text-white hover:bg-red-700',
  success: 'bg-green-600 text-white hover:bg-green-700',
  ghost: 'bg-transparent text-blue-600 hover:bg-blue-50',
};

export function Button({ variant = 'primary', ...props }) {
  return (
    <button
      className={`px-4 py-2 rounded ${VARIANTS[variant]}`}
      {...props}
    />
  );
}
```

### 3. Composition over Duplication

```jsx
// Instead of separate SimpleCard, DetailedCard components
function Card({ children, variant = 'simple' }) {
  if (variant === 'simple') {
    return <div className="p-4 bg-white rounded">{children}</div>;
  }
  
  return (
    <div className="p-6 bg-white rounded-lg shadow-lg">
      {children}
    </div>
  );
}

// Better: Use composition
function Card({ children, className = '' }) {
  return <div className={`bg-white rounded ${className}`}>{children}</div>;
}

// Usage
<Card className="p-4">Simple content</Card>
<Card className="p-6 shadow-lg">Detailed content</Card>
```

---

## Code Splitting and Lazy Loading

### 1. Route-Level Code Splitting

```jsx
// src/App.jsx
import { lazy, Suspense } from 'react';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { LoadingSpinner } from '@/components/shared/LoadingSpinner';

// Eager load critical routes
import { Landing } from '@/pages/Landing';
import { Login } from '@/pages/Login';
import { PrivateLayout } from '@/components/layout/PrivateLayout';

// Lazy load non-critical routes
const Dashboard = lazy(() => import('@/pages/Dashboard'));
const Chores = lazy(() => import('@/pages/Chores'));
const ChoreDetail = lazy(() => import('@/pages/ChoreDetail'));
const Money = lazy(() => import('@/pages/Money'));
const Allowances = lazy(() => import('@/pages/Allowances'));
const Goals = lazy(() => import('@/pages/Goals'));
const Settings = lazy(() => import('@/pages/Settings'));

export function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<LoadingSpinner fullScreen />}>
        <Routes>
          <Route path="/" element={<Landing />} />
          <Route path="/login" element={<Login />} />
          
          <Route element={<PrivateLayout />}>
            <Route path="/dashboard" element={<Dashboard />} />
            <Route path="/chores" element={<Chores />} />
            <Route path="/chores/:id" element={<ChoreDetail />} />
            <Route path="/money" element={<Money />} />
            <Route path="/allowances" element={<Allowances />} />
            <Route path="/goals" element={<Goals />} />
            <Route path="/settings" element={<Settings />} />
          </Route>
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}
```

### 2. Component-Level Code Splitting

```jsx
// Lazy load heavy components
const RecurrenceBuilder = lazy(() => 
  import('@/components/chores/RecurrenceBuilder')
);

export function ChoreForm() {
  const [showAdvanced, setShowAdvanced] = useState(false);

  return (
    <form>
      {/* Basic fields always loaded */}
      <Input label="Title" />
      <Input label="Description" />
      
      {/* Heavy component loaded on demand */}
      {showAdvanced && (
        <Suspense fallback={<LoadingSpinner />}>
          <RecurrenceBuilder />
        </Suspense>
      )}
      
      <Button onClick={() => setShowAdvanced(true)}>
        Advanced Settings
      </Button>
    </form>
  );
}
```

### 3. Dynamic Imports

```jsx
// Load component based on condition
export function DynamicDashboard() {
  const { user, ageBracket } = useUser();
  const [DashboardComponent, setDashboardComponent] = useState(null);

  useEffect(() => {
    if (user.role === 'PARENT') {
      import('@/components/dashboard/ParentDashboard')
        .then(module => setDashboardComponent(() => module.default));
    } else {
      const dashboardPath = {
        young: '@/components/dashboard/YoungDashboard',
        tween: '@/components/dashboard/TweenDashboard',
        teen: '@/components/dashboard/TeenDashboard',
      }[ageBracket];
      
      import(dashboardPath)
        .then(module => setDashboardComponent(() => module.default));
    }
  }, [user, ageBracket]);

  if (!DashboardComponent) {
    return <LoadingSpinner />;
  }

  return <DashboardComponent />;
}
```

### 4. Preloading Strategy

```jsx
// Preload on hover for better UX
import { Link } from 'react-router-dom';

const preloadDashboard = () => {
  import('@/pages/Dashboard');
};

export function Navigation() {
  return (
    <nav>
      <Link 
        to="/dashboard" 
        onMouseEnter={preloadDashboard}
        onFocus={preloadDashboard}
      >
        Dashboard
      </Link>
    </nav>
  );
}
```

### 5. Bundle Analysis

```bash
# Analyze bundle sizes
npm run build
npm run analyze  # Uses rollup-plugin-visualizer

# Check bundle sizes in vite.config.js
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'react-vendor': ['react', 'react-dom', 'react-router-dom'],
          'ui-vendor': ['@headlessui/react', '@heroicons/react'],
          'form-vendor': ['react-hook-form', 'zod'],
        },
      },
    },
  },
});
```

---

## Testing Components

### Unit Test Example

```jsx
// components/chores/ChoreCard.test.jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { ChoreCard } from './ChoreCard';

describe('ChoreCard', () => {
  const mockChore = {
    id: '1',
    title: 'Clean your room',
    reward_amount: 5.00,
    status: 'PENDING',
  };

  it('renders chore title and reward', () => {
    render(<ChoreCard chore={mockChore} />);
    
    expect(screen.getByText('Clean your room')).toBeInTheDocument();
    expect(screen.getByText('$5.00')).toBeInTheDocument();
  });

  it('calls onComplete when button clicked', async () => {
    const user = userEvent.setup();
    const onComplete = vi.fn();
    
    render(<ChoreCard chore={mockChore} onComplete={onComplete} />);
    
    await user.click(screen.getByText('Mark Complete'));
    
    expect(onComplete).toHaveBeenCalledWith('1');
  });
});
```

---

## Performance Optimization

### 1. Memoization

```jsx
import { memo, useMemo, useCallback } from 'react';

// Memoize expensive components
export const ChoreCard = memo(function ChoreCard({ chore, onComplete }) {
  // Component only re-renders if chore or onComplete changes
  return (
    <Card>
      <h3>{chore.title}</h3>
      <Button onClick={() => onComplete(chore.id)}>Complete</Button>
    </Card>
  );
});

// Memoize expensive calculations
function ChoreList({ chores }) {
  const sortedChores = useMemo(() => {
    return [...chores].sort((a, b) => 
      new Date(a.due_date) - new Date(b.due_date)
    );
  }, [chores]);

  return <div>{sortedChores.map(chore => ...)}</div>;
}

// Memoize callbacks
function ChoreList({ chores, onComplete }) {
  const handleComplete = useCallback((choreId) => {
    onComplete(choreId);
  }, [onComplete]);

  return (
    <div>
      {chores.map(chore => (
        <ChoreCard key={chore.id} chore={chore} onComplete={handleComplete} />
      ))}
    </div>
  );
}
```

### 2. Virtualization

```jsx
// For long lists (100+ items), use virtualization
import { FixedSizeList } from 'react-window';

export function VirtualizedTransactionList({ transactions }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      <TransactionItem transaction={transactions[index]} />
    </div>
  );

  return (
    <FixedSizeList
      height={600}
      itemCount={transactions.length}
      itemSize={60}
      width="100%"
    >
      {Row}
    </FixedSizeList>
  );
}
```

---

## Best Practices Summary

1. **Keep components small and focused** - Single responsibility principle
2. **Prefer composition over inheritance** - Build complex UIs from simple components
3. **Use TypeScript or PropTypes** - Document expected props
4. **Extract reusable logic to hooks** - DRY principle
5. **Implement proper loading states** - Better UX
6. **Handle errors gracefully** - Error boundaries + fallback UI
7. **Optimize with memo/useMemo/useCallback** - Only when needed (measure first)
8. **Split code at route boundaries** - Faster initial load
9. **Test components in isolation** - Unit tests for logic, integration tests for workflows
10. **Follow accessibility guidelines** - Semantic HTML, ARIA labels, keyboard navigation

---

*Last Updated: January 29, 2026*
