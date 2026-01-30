# Age-Adaptive UI Guide

## Overview

PicklesApp automatically adapts its user interface based on the child's age bracket, providing age-appropriate complexity, language, and features. This ensures young children aren't overwhelmed while teens get full financial management capabilities.

## Table of Contents
1. [Age Bracket System](#age-bracket-system)
2. [Typography Scaling](#typography-scaling)
3. [Component Variants by Age](#component-variants-by-age)
4. [Feature Gating](#feature-gating)
5. [Navigation Differences](#navigation-differences)
6. [Icon vs Emoji Usage](#icon-vs-emoji-usage)
7. [Touch Target Sizes](#touch-target-sizes)
8. [Implementation Examples](#implementation-examples)
9. [Age Transition Handling](#age-transition-handling)

---

## Age Bracket System

### Three Age Brackets

| Bracket | Age Range | Characteristics | UI Philosophy |
|---------|-----------|-----------------|---------------|
| **Young** | 5-9 years | Early readers, concrete thinking, limited attention span | Large visuals, minimal text, gamification, simple features |
| **Tween** | 10-13 years | Developing independence, abstract thinking emerging | Balanced text/icons, moderate features, educational focus |
| **Teen** | 14-17 years | Adult-like cognition, financial planning capable | Full features, detailed interface, investment tools |

### Age Calculation

```javascript
// utils/ageBracket.js

/**
 * Calculate age from birthdate
 */
export function calculateAge(birthdate) {
  if (!birthdate) return null;
  
  const today = new Date();
  const birth = new Date(birthdate);
  
  let age = today.getFullYear() - birth.getFullYear();
  const monthDiff = today.getMonth() - birth.getMonth();
  
  // Adjust if birthday hasn't occurred this year
  if (monthDiff < 0 || (monthDiff === 0 && today.getDate() < birth.getDate())) {
    age--;
  }
  
  return age;
}

/**
 * Determine age bracket from birthdate
 */
export function calculateAgeBracket(birthdate) {
  if (!birthdate) return 'teen'; // Default for users without birthdate
  
  const age = calculateAge(birthdate);
  
  if (age < 10) return 'young';
  if (age < 14) return 'tween';
  return 'teen';
}

/**
 * Get effective age bracket with parent override
 */
export function getEffectiveAgeBracket(user, familyMembership) {
  // Parent role always gets adult interface
  if (user.role === 'PARENT') {
    return 'adult';
  }
  
  // Check for parent override
  if (familyMembership?.age_bracket_override) {
    return familyMembership.age_bracket_override;
  }
  
  // Calculate from birthdate
  return calculateAgeBracket(user.birthdate);
}
```

### Parent Override Support

Parents can override age bracket for individual children:

```jsx
// components/family/AgeBracketOverride.jsx
import { useState } from 'react';
import { Select } from '@/components/shared/Select';
import { Button } from '@/components/shared/Button';
import { familyService } from '@/services/familyService';
import toast from 'react-hot-toast';

export function AgeBracketOverride({ child, familyId }) {
  const [override, setOverride] = useState(child.age_bracket_override || 'auto');
  const [saving, setSaving] = useState(false);

  const calculatedBracket = calculateAgeBracket(child.birthdate);

  const handleSave = async () => {
    setSaving(true);
    try {
      await familyService.updateMemberSettings(familyId, child.id, {
        age_bracket_override: override === 'auto' ? null : override,
      });
      toast.success('Age bracket updated');
    } catch (error) {
      toast.error('Failed to update age bracket');
    } finally {
      setSaving(false);
    }
  };

  return (
    <div className="space-y-4">
      <div>
        <label className="block text-sm font-medium text-gray-700 mb-2">
          Age Bracket for {child.first_name}
        </label>
        <p className="text-sm text-gray-500 mb-3">
          Current age: {calculateAge(child.birthdate)} years old
          (auto-detected: {calculatedBracket})
        </p>
        
        <Select value={override} onChange={(e) => setOverride(e.target.value)}>
          <option value="auto">Auto (based on age)</option>
          <option value="young">Young (5-9) - Simplified interface</option>
          <option value="tween">Tween (10-13) - Balanced interface</option>
          <option value="teen">Teen (14-17) - Full features</option>
        </Select>
      </div>
      
      <Button onClick={handleSave} disabled={saving}>
        {saving ? 'Saving...' : 'Save Changes'}
      </Button>
    </div>
  );
}
```

---

## Typography Scaling

### Base Typography Classes

```css
/* styles/age-adaptive.css */

/* Young (5-9): Larger text for early readers */
.ui-young {
  --font-size-xs: 14px;
  --font-size-sm: 16px;
  --font-size-base: 18px;
  --font-size-lg: 24px;
  --font-size-xl: 32px;
  --font-size-2xl: 40px;
  --font-size-3xl: 48px;
  
  --font-weight-normal: 500;
  --font-weight-medium: 600;
  --font-weight-bold: 700;
  
  --line-height-tight: 1.3;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.75;
}

/* Tween (10-13): Standard sizing */
.ui-tween {
  --font-size-xs: 12px;
  --font-size-sm: 14px;
  --font-size-base: 16px;
  --font-size-lg: 20px;
  --font-size-xl: 24px;
  --font-size-2xl: 32px;
  --font-size-3xl: 40px;
  
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-bold: 700;
  
  --line-height-tight: 1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.625;
}

/* Teen (14-17): Compact sizing */
.ui-teen {
  --font-size-xs: 11px;
  --font-size-sm: 13px;
  --font-size-base: 14px;
  --font-size-lg: 18px;
  --font-size-xl: 20px;
  --font-size-2xl: 24px;
  --font-size-3xl: 30px;
  
  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-bold: 600;
  
  --line-height-tight: 1.25;
  --line-height-normal: 1.5;
  --line-height-relaxed: 1.625;
}
```

### Tailwind Configuration

```javascript
// tailwind.config.js
module.exports = {
  theme: {
    extend: {
      fontSize: {
        // Age-adaptive font sizes (use CSS variables)
        'adaptive-xs': 'var(--font-size-xs)',
        'adaptive-sm': 'var(--font-size-sm)',
        'adaptive-base': 'var(--font-size-base)',
        'adaptive-lg': 'var(--font-size-lg)',
        'adaptive-xl': 'var(--font-size-xl)',
        'adaptive-2xl': 'var(--font-size-2xl)',
        'adaptive-3xl': 'var(--font-size-3xl)',
      },
    },
  },
};
```

### Usage in Components

```jsx
export function AdaptiveText({ children, size = 'base', className = '' }) {
  const sizeClass = `text-adaptive-${size}`;
  
  return (
    <span className={`${sizeClass} ${className}`}>
      {children}
    </span>
  );
}

// Usage
<AdaptiveText size="xl">Account Balance</AdaptiveText>
<AdaptiveText size="base">$125.50</AdaptiveText>
```

---

## Component Variants by Age

### Dashboard Components

#### Young Dashboard (5-9)

```jsx
// components/dashboard/YoungDashboard.jsx
import { useAccounts } from '@/hooks/useAccounts';
import { useChores } from '@/hooks/useChores';

export function YoungDashboard() {
  const { totalBalance } = useAccounts();
  const { chores } = useChores({ status: 'PENDING' });

  return (
    <div className="ui-young p-6 space-y-6">
      {/* Big balance card with emoji */}
      <div className="bg-gradient-to-r from-green-400 to-green-500 rounded-3xl p-8 text-center text-white shadow-lg">
        <div className="text-6xl mb-4">💰</div>
        <div className="text-adaptive-3xl font-bold mb-2">
          ${totalBalance.toFixed(2)}
        </div>
        <div className="text-adaptive-lg">You have</div>
      </div>

      {/* Chores to do */}
      <div className="bg-blue-100 rounded-3xl p-8 text-center">
        <div className="text-6xl mb-4">🎯</div>
        <div className="text-adaptive-3xl font-bold text-blue-900 mb-2">
          {chores.length}
        </div>
        <div className="text-adaptive-lg text-blue-800">
          {chores.length === 1 ? 'Chore to do' : 'Chores to do'}
        </div>
      </div>

      {/* Goals (if any) */}
      <div className="bg-purple-100 rounded-3xl p-8 text-center">
        <div className="text-6xl mb-4">⭐</div>
        <div className="text-adaptive-lg text-purple-800">
          Save for your goals!
        </div>
      </div>
    </div>
  );
}
```

#### Tween Dashboard (10-13)

```jsx
// components/dashboard/TweenDashboard.jsx
import { useAccounts } from '@/hooks/useAccounts';
import { useChores } from '@/hooks/useChores';
import { useTransactions } from '@/hooks/useTransactions';
import { AccountCard } from '@/components/money/AccountCard';
import { ChoreList } from '@/components/chores/ChoreList';
import { TransactionList } from '@/components/money/TransactionList';

export function TweenDashboard() {
  const { accounts, totalBalance } = useAccounts();
  const { chores } = useChores({ status: 'PENDING', limit: 5 });
  const { transactions } = useTransactions(null, { limit: 5 });

  return (
    <div className="ui-tween container mx-auto p-4 space-y-6">
      {/* Balance summary */}
      <div className="bg-white rounded-lg shadow p-6">
        <h2 className="text-adaptive-xl font-semibold mb-4">
          Total Balance
        </h2>
        <div className="text-adaptive-3xl font-bold text-green-600">
          ${totalBalance.toFixed(2)}
        </div>
        
        <div className="mt-4 space-y-2">
          {accounts.map(account => (
            <div key={account.id} className="flex justify-between text-adaptive-sm">
              <span className="text-gray-600">{account.name}</span>
              <span className="font-medium">${account.balance.toFixed(2)}</span>
            </div>
          ))}
        </div>
      </div>

      {/* Today's chores */}
      <div className="bg-white rounded-lg shadow p-6">
        <h2 className="text-adaptive-xl font-semibold mb-4">
          Today's Chores ({chores.length})
        </h2>
        <ChoreList chores={chores} variant="compact" />
      </div>

      {/* Recent activity */}
      <div className="bg-white rounded-lg shadow p-6">
        <h2 className="text-adaptive-xl font-semibold mb-4">
          Recent Activity
        </h2>
        <TransactionList transactions={transactions} variant="simple" />
      </div>
    </div>
  );
}
```

#### Teen Dashboard (14-17)

```jsx
// components/dashboard/TeenDashboard.jsx
import { useAccounts } from '@/hooks/useAccounts';
import { useChores } from '@/hooks/useChores';
import { useTransactions } from '@/hooks/useTransactions';
import { AccountSummary } from '@/components/money/AccountSummary';
import { IncomeExpenseChart } from '@/components/money/IncomeExpenseChart';
import { ChoreSchedule } from '@/components/chores/ChoreSchedule';
import { GoalProgress } from '@/components/goals/GoalProgress';
import { TransactionTable } from '@/components/money/TransactionTable';

export function TeenDashboard() {
  const { accounts } = useAccounts();
  const { chores } = useChores();
  const { transactions } = useTransactions();

  return (
    <div className="ui-teen container mx-auto p-6 space-y-6">
      {/* Top row: Account summary + Chart */}
      <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <div className="lg:col-span-1">
          <AccountSummary accounts={accounts} />
        </div>
        <div className="lg:col-span-2">
          <IncomeExpenseChart />
        </div>
      </div>

      {/* Middle row: Chores + Goals */}
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <ChoreSchedule chores={chores} />
        <GoalProgress />
      </div>

      {/* Bottom: Transaction table */}
      <div className="bg-white rounded-lg shadow">
        <TransactionTable 
          transactions={transactions} 
          showFilters 
          exportable 
        />
      </div>
    </div>
  );
}
```

### Chore Card Variants

#### Young (5-9)

```jsx
// components/chores/YoungChoreCard.jsx
export function YoungChoreCard({ chore, onComplete }) {
  return (
    <div className="bg-white rounded-3xl shadow-lg p-8 text-center">
      {/* Big icon */}
      <div className="text-7xl mb-4">🧹</div>
      
      {/* Simple title */}
      <h3 className="text-adaptive-xl font-bold mb-4">
        {chore.title}
      </h3>
      
      {/* Reward amount */}
      {chore.reward_amount && (
        <div className="bg-green-100 rounded-2xl py-3 px-6 inline-block mb-6">
          <div className="text-adaptive-2xl font-bold text-green-700">
            ${chore.reward_amount.toFixed(2)} 💰
          </div>
        </div>
      )}
      
      {/* Big action button */}
      <button
        onClick={() => onComplete(chore.id)}
        className="w-full bg-blue-500 hover:bg-blue-600 text-white rounded-2xl py-6 text-adaptive-xl font-bold shadow-lg"
      >
        ✓ I did it!
      </button>
    </div>
  );
}
```

#### Tween (10-13)

```jsx
// components/chores/TweenChoreCard.jsx
import { ClockIcon, CurrencyDollarIcon } from '@heroicons/react/24/outline';

export function TweenChoreCard({ chore, onComplete, onView }) {
  return (
    <div className="bg-white rounded-lg shadow hover:shadow-lg transition-shadow p-4">
      <div className="flex items-start justify-between mb-3">
        <div className="flex-1">
          <h3 className="text-adaptive-lg font-semibold">{chore.title}</h3>
          {chore.description && (
            <p className="text-adaptive-sm text-gray-600 mt-1">
              {chore.description}
            </p>
          )}
        </div>
        
        {chore.reward_amount && (
          <div className="flex items-center gap-1 bg-green-100 text-green-700 px-3 py-1 rounded-full text-adaptive-sm font-medium">
            <CurrencyDollarIcon className="w-4 h-4" />
            {chore.reward_amount.toFixed(2)}
          </div>
        )}
      </div>
      
      {chore.due_date && (
        <div className="flex items-center gap-2 text-adaptive-sm text-gray-500 mb-4">
          <ClockIcon className="w-4 h-4" />
          Due: {formatDate(chore.due_date)}
        </div>
      )}
      
      <div className="flex gap-2">
        <button
          onClick={() => onView(chore.id)}
          className="flex-1 border border-gray-300 rounded-lg py-2 text-adaptive-sm font-medium hover:bg-gray-50"
        >
          View Details
        </button>
        <button
          onClick={() => onComplete(chore.id)}
          className="flex-1 bg-blue-600 text-white rounded-lg py-2 text-adaptive-sm font-medium hover:bg-blue-700"
        >
          Mark Complete
        </button>
      </div>
    </div>
  );
}
```

#### Teen (14-17)

```jsx
// components/chores/TeenChoreCard.jsx
import { 
  ClockIcon, 
  CurrencyDollarIcon, 
  CalendarIcon 
} from '@heroicons/react/24/outline';

export function TeenChoreCard({ chore, onComplete, onView }) {
  return (
    <div className="bg-white rounded-lg shadow-sm border border-gray-200 p-4 hover:border-blue-300 transition-colors">
      <div className="flex items-start justify-between mb-3">
        <div className="flex-1">
          <h3 className="text-adaptive-base font-semibold">{chore.title}</h3>
          {chore.description && (
            <p className="text-adaptive-sm text-gray-600 mt-1 line-clamp-2">
              {chore.description}
            </p>
          )}
        </div>
        
        <StatusBadge status={chore.status} />
      </div>
      
      <div className="grid grid-cols-3 gap-3 mb-3 text-adaptive-xs">
        {chore.reward_amount && (
          <div className="flex items-center gap-1 text-gray-600">
            <CurrencyDollarIcon className="w-4 h-4" />
            <span>${chore.reward_amount.toFixed(2)}</span>
          </div>
        )}
        
        {chore.due_date && (
          <div className="flex items-center gap-1 text-gray-600">
            <CalendarIcon className="w-4 h-4" />
            <span>{formatShortDate(chore.due_date)}</span>
          </div>
        )}
        
        {chore.estimated_duration && (
          <div className="flex items-center gap-1 text-gray-600">
            <ClockIcon className="w-4 h-4" />
            <span>{chore.estimated_duration}min</span>
          </div>
        )}
      </div>
      
      <div className="flex gap-2">
        <button
          onClick={() => onView(chore.id)}
          className="text-adaptive-xs text-blue-600 hover:underline"
        >
          View History
        </button>
        {chore.status === 'PENDING' && (
          <button
            onClick={() => onComplete(chore.id)}
            className="ml-auto bg-blue-600 text-white px-4 py-1.5 rounded text-adaptive-xs font-medium hover:bg-blue-700"
          >
            Complete
          </button>
        )}
      </div>
    </div>
  );
}
```

---

## Feature Gating

### Feature Access Matrix

```javascript
// utils/featureAccess.js

export const FEATURE_ACCESS = {
  young: {
    // Core features
    view_chores: true,
    complete_chores: true,
    view_balance: true,
    view_goals: true,
    
    // Limited features
    request_money: false,
    transfer_between_accounts: false,
    view_transaction_history: false,
    manage_budget: false,
    view_investment_accounts: false,
    set_own_goals: false,
    view_analytics: false,
    export_data: false,
  },
  
  tween: {
    // Core features
    view_chores: true,
    complete_chores: true,
    view_balance: true,
    view_goals: true,
    view_transaction_history: true,
    
    // Moderate features
    request_money: true,
    transfer_between_accounts: true, // With limits
    set_own_goals: true,
    
    // Still limited
    manage_budget: false,
    view_investment_accounts: false,
    view_analytics: false,
    export_data: false,
  },
  
  teen: {
    // All features enabled
    view_chores: true,
    complete_chores: true,
    view_balance: true,
    view_goals: true,
    view_transaction_history: true,
    request_money: true,
    transfer_between_accounts: true,
    manage_budget: true,
    view_investment_accounts: true,
    set_own_goals: true,
    view_analytics: true,
    export_data: true,
  },
};

export function hasFeatureAccess(ageBracket, feature) {
  return FEATURE_ACCESS[ageBracket]?.[feature] ?? false;
}
```

### FeatureGate Component

```jsx
// components/adaptive/FeatureGate.jsx
import { useAgeBracket } from '@/hooks/useAgeBracket';
import { hasFeatureAccess } from '@/utils/featureAccess';

export function FeatureGate({ 
  feature, 
  children, 
  fallback = null,
  showLockedMessage = true,
}) {
  const { ageBracket, isParent } = useAgeBracket();

  // Parents have access to all features
  if (isParent) {
    return children;
  }

  // Check feature access
  if (!hasFeatureAccess(ageBracket, feature)) {
    if (showLockedMessage) {
      return (
        <div className="bg-gray-50 rounded-lg p-8 text-center">
          <div className="text-4xl mb-3">🔒</div>
          <h3 className="text-lg font-semibold mb-2">
            Feature Not Available Yet
          </h3>
          <p className="text-gray-600">
            This feature will unlock as you get older!
          </p>
        </div>
      );
    }
    return fallback;
  }

  return children;
}

// Usage examples
<FeatureGate feature="transfer_between_accounts">
  <TransferForm />
</FeatureGate>

<FeatureGate feature="view_analytics" showLockedMessage={false}>
  <AnalyticsDashboard />
</FeatureGate>
```

---

## Navigation Differences

### Young (5-9): Bottom Navigation with Emojis

```jsx
// components/layout/YoungNavigation.jsx
import { useLocation, Link } from 'react-router-dom';

export function YoungNavigation() {
  const location = useLocation();

  const navItems = [
    { path: '/chores', emoji: '🎯', label: 'Chores' },
    { path: '/money', emoji: '💰', label: 'Money' },
    { path: '/goals', emoji: '⭐', label: 'Goals' },
  ];

  return (
    <nav className="fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 safe-area-bottom">
      <div className="flex justify-around py-3">
        {navItems.map(item => (
          <Link
            key={item.path}
            to={item.path}
            className={`flex flex-col items-center min-w-[80px] ${
              location.pathname === item.path
                ? 'text-blue-600'
                : 'text-gray-500'
            }`}
          >
            <div className="text-4xl mb-1">{item.emoji}</div>
            <span className="text-adaptive-sm font-medium">
              {item.label}
            </span>
          </Link>
        ))}
      </div>
    </nav>
  );
}
```

### Tween (10-13): Bottom Navigation with Icons

```jsx
// components/layout/TweenNavigation.jsx
import { useLocation, Link } from 'react-router-dom';
import {
  ClipboardListIcon,
  CurrencyDollarIcon,
  ChartBarIcon,
  CogIcon,
} from '@heroicons/react/24/outline';

export function TweenNavigation() {
  const location = useLocation();

  const navItems = [
    { path: '/chores', icon: ClipboardListIcon, label: 'Chores' },
    { path: '/money', icon: CurrencyDollarIcon, label: 'Money' },
    { path: '/goals', icon: ChartBarIcon, label: 'Goals' },
    { path: '/settings', icon: CogIcon, label: 'Settings' },
  ];

  return (
    <nav className="fixed bottom-0 left-0 right-0 bg-white border-t border-gray-200 safe-area-bottom">
      <div className="flex justify-around py-2">
        {navItems.map(item => {
          const Icon = item.icon;
          const isActive = location.pathname === item.path;
          
          return (
            <Link
              key={item.path}
              to={item.path}
              className={`flex flex-col items-center min-w-[70px] p-2 ${
                isActive
                  ? 'text-blue-600'
                  : 'text-gray-500 hover:text-gray-700'
              }`}
            >
              <Icon className="w-6 h-6 mb-1" />
              <span className="text-adaptive-xs font-medium">
                {item.label}
              </span>
            </Link>
          );
        })}
      </div>
    </nav>
  );
}
```

### Teen (14-17): Sidebar Navigation

```jsx
// components/layout/TeenNavigation.jsx
import { useLocation, Link } from 'react-router-dom';
import {
  HomeIcon,
  ClipboardListIcon,
  CurrencyDollarIcon,
  BanknotesIcon,
  ChartBarIcon,
  CogIcon,
} from '@heroicons/react/24/outline';

export function TeenNavigation() {
  const location = useLocation();

  const sections = [
    {
      title: 'Dashboard',
      items: [
        { path: '/dashboard', icon: HomeIcon, label: 'Overview' },
      ],
    },
    {
      title: 'Financial',
      items: [
        { path: '/accounts', icon: BanknotesIcon, label: 'Accounts' },
        { path: '/transactions', icon: CurrencyDollarIcon, label: 'Transactions' },
        { path: '/goals', icon: ChartBarIcon, label: 'Goals' },
      ],
    },
    {
      title: 'Tasks',
      items: [
        { path: '/chores', icon: ClipboardListIcon, label: 'Chores' },
      ],
    },
    {
      title: 'Settings',
      items: [
        { path: '/settings', icon: CogIcon, label: 'Settings' },
      ],
    },
  ];

  return (
    <nav className="fixed left-0 top-0 bottom-0 w-64 bg-white border-r border-gray-200 overflow-y-auto">
      <div className="p-4">
        <h1 className="text-xl font-bold text-blue-600 mb-8">PicklesApp</h1>
        
        {sections.map(section => (
          <div key={section.title} className="mb-6">
            <h2 className="text-xs font-semibold text-gray-500 uppercase tracking-wider mb-2 px-3">
              {section.title}
            </h2>
            <div className="space-y-1">
              {section.items.map(item => {
                const Icon = item.icon;
                const isActive = location.pathname === item.path;
                
                return (
                  <Link
                    key={item.path}
                    to={item.path}
                    className={`flex items-center gap-3 px-3 py-2 rounded-lg text-adaptive-sm ${
                      isActive
                        ? 'bg-blue-50 text-blue-600 font-medium'
                        : 'text-gray-700 hover:bg-gray-50'
                    }`}
                  >
                    <Icon className="w-5 h-5" />
                    <span>{item.label}</span>
                  </Link>
                );
              })}
            </div>
          </div>
        ))}
      </div>
    </nav>
  );
}
```

---

## Icon vs Emoji Usage

### Guidelines

**Use Emojis for Young (5-9):**
- More colorful and playful
- Universally recognizable
- Larger visual impact
- Examples: 🎯 💰 ⭐ 🎉 🔥

**Use Icons for Tween/Teen:**
- More professional appearance
- Consistent design language
- Better scalability
- Use Heroicons or similar

### Implementation

```jsx
// components/adaptive/AdaptiveIcon.jsx
import { useAgeBracket } from '@/hooks/useAgeBracket';

export function AdaptiveIcon({ emoji, icon: Icon, className = '' }) {
  const { isYoung } = useAgeBracket();

  if (isYoung) {
    return <span className={`text-4xl ${className}`}>{emoji}</span>;
  }

  return <Icon className={`w-6 h-6 ${className}`} />;
}

// Usage
<AdaptiveIcon 
  emoji="💰" 
  icon={CurrencyDollarIcon}
  className="mr-2"
/>
```

---

## Touch Target Sizes

### Minimum Sizes by Age

```css
/* styles/age-adaptive.css */

/* Young (5-9): Large touch targets */
.ui-young {
  --button-height: 60px;
  --button-min-width: 120px;
  --touch-target-min: 60px;
  --spacing-base: 24px;
  --border-radius: 24px;
}

/* Tween (10-13): Medium touch targets */
.ui-tween {
  --button-height: 48px;
  --button-min-width: 100px;
  --touch-target-min: 48px;
  --spacing-base: 16px;
  --border-radius: 12px;
}

/* Teen (14-17): Standard touch targets */
.ui-teen {
  --button-height: 40px;
  --button-min-width: 80px;
  --touch-target-min: 44px; /* iOS minimum */
  --spacing-base: 12px;
  --border-radius: 8px;
}
```

### Adaptive Button Component

```jsx
// components/adaptive/AdaptiveButton.jsx
import { useAgeBracket } from '@/hooks/useAgeBracket';

export function AdaptiveButton({ children, variant = 'primary', ...props }) {
  const { ageBracket } = useAgeBracket();

  const baseClasses = 'font-medium transition-colors';
  const variantClasses = {
    primary: 'bg-blue-600 text-white hover:bg-blue-700',
    secondary: 'bg-gray-200 text-gray-800 hover:bg-gray-300',
  };

  return (
    <button
      className={`
        ${baseClasses}
        ${variantClasses[variant]}
        min-h-[var(--button-height)]
        min-w-[var(--button-min-width)]
        rounded-[var(--border-radius)]
        px-[var(--spacing-base)]
        text-adaptive-base
      `}
      {...props}
    >
      {children}
    </button>
  );
}
```

---

## Implementation Examples

### Complete Adaptive Component

```jsx
// components/adaptive/AdaptiveChoreList.jsx
import { useAgeBracket } from '@/hooks/useAgeBracket';
import { YoungChoreCard } from '@/components/chores/YoungChoreCard';
import { TweenChoreCard } from '@/components/chores/TweenChoreCard';
import { TeenChoreCard } from '@/components/chores/TeenChoreCard';

export function AdaptiveChoreList({ chores, onComplete, onView }) {
  const { ageBracket } = useAgeBracket();

  const CardComponent = {
    young: YoungChoreCard,
    tween: TweenChoreCard,
    teen: TeenChoreCard,
  }[ageBracket] || TweenChoreCard;

  const gridClasses = {
    young: 'grid grid-cols-1 gap-6',
    tween: 'grid grid-cols-1 md:grid-cols-2 gap-4',
    teen: 'grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-3',
  }[ageBracket];

  return (
    <div className={gridClasses}>
      {chores.map(chore => (
        <CardComponent
          key={chore.id}
          chore={chore}
          onComplete={onComplete}
          onView={onView}
        />
      ))}
    </div>
  );
}
```

---

## Age Transition Handling

### Birthday Detection

```jsx
// hooks/useAgeTransition.js
import { useEffect, useState } from 'react';
import { useUser } from './useUser';
import { calculateAgeBracket } from '@/utils/ageBracket';

export function useAgeTransition() {
  const { user } = useUser();
  const [showAgeUpModal, setShowAgeUpModal] = useState(false);

  useEffect(() => {
    // Check if user aged up today
    const checkAgeUp = () => {
      const today = new Date();
      const birthdate = new Date(user.birthdate);
      
      // Is it their birthday?
      if (
        today.getMonth() === birthdate.getMonth() &&
        today.getDate() === birthdate.getDate()
      ) {
        // Did they move to new bracket?
        const yesterdayDate = new Date(today);
        yesterdayDate.setDate(yesterdayDate.getDate() - 1);
        
        const oldBracket = calculateAgeBracket(yesterdayDate);
        const newBracket = calculateAgeBracket(today);
        
        if (oldBracket !== newBracket) {
          setShowAgeUpModal(true);
        }
      }
    };

    if (user?.birthdate && user.role === 'CHILD') {
      checkAgeUp();
    }
  }, [user]);

  return { showAgeUpModal, setShowAgeUpModal };
}
```

### Age-Up Celebration Modal

```jsx
// components/adaptive/AgeUpModal.jsx
import { Modal } from '@/components/shared/Modal';
import { Button } from '@/components/shared/Button';
import { useUser } from '@/hooks/useUser';
import { FEATURE_ACCESS } from '@/utils/featureAccess';
import Confetti from 'react-confetti';

export function AgeUpModal({ isOpen, onClose }) {
  const { user, ageBracket } = useUser();

  // Get newly unlocked features
  const getNewFeatures = () => {
    const features = {
      tween: [
        { icon: '💸', name: 'Money Requests', description: 'Ask parents for money' },
        { icon: '↔️', name: 'Account Transfers', description: 'Move money between accounts' },
        { icon: '📊', name: 'Transaction History', description: 'See all your transactions' },
      ],
      teen: [
        { icon: '📈', name: 'Budget Tools', description: 'Create and track budgets' },
        { icon: '💼', name: 'Investment Tracking', description: 'Monitor investment accounts' },
        { icon: '📉', name: 'Analytics', description: 'View spending patterns' },
      ],
    };
    
    return features[ageBracket] || [];
  };

  return (
    <Modal isOpen={isOpen} onClose={onClose} size="lg">
      <Confetti numberOfPieces={200} recycle={false} />
      
      <div className="text-center py-8">
        <div className="text-8xl mb-4">🎂</div>
        
        <h2 className="text-3xl font-bold mb-2">
          Happy Birthday, {user.first_name}!
        </h2>
        
        <p className="text-xl text-gray-600 mb-8">
          You've leveled up! 🎉
        </p>
        
        <div className="bg-blue-50 rounded-lg p-6 mb-8">
          <h3 className="text-lg font-semibold mb-4">
            Check out your new features:
          </h3>
          
          <div className="space-y-4">
            {getNewFeatures().map((feature, index) => (
              <div
                key={index}
                className="bg-white rounded-lg p-4 flex items-start gap-4 text-left"
              >
                <div className="text-4xl">{feature.icon}</div>
                <div>
                  <h4 className="font-semibold">{feature.name}</h4>
                  <p className="text-sm text-gray-600">{feature.description}</p>
                </div>
              </div>
            ))}
          </div>
        </div>
        
        <Button onClick={onClose} size="lg">
          Explore my new dashboard!
        </Button>
      </div>
    </Modal>
  );
}
```

---

## Testing Age-Adaptive Features

```javascript
// tests/components/adaptive/AdaptiveComponent.test.jsx
import { render, screen } from '@testing-library/react';
import { UserContext } from '@/contexts/UserContext';
import { AdaptiveChoreList } from '@/components/adaptive/AdaptiveChoreList';

describe('AdaptiveChoreList', () => {
  const mockChores = [
    { id: '1', title: 'Clean room', reward_amount: 5 },
  ];

  it('renders young variant for 7-year-old', () => {
    const user = { id: '1', birthdate: '2019-01-01', role: 'CHILD' };
    
    render(
      <UserContext.Provider value={{ user, ageBracket: 'young' }}>
        <AdaptiveChoreList chores={mockChores} />
      </UserContext.Provider>
    );
    
    // Young variant has emoji
    expect(screen.getByText('🧹')).toBeInTheDocument();
    expect(screen.getByText('I did it!')).toBeInTheDocument();
  });

  it('renders teen variant for 15-year-old', () => {
    const user = { id: '1', birthdate: '2011-01-01', role: 'CHILD' };
    
    render(
      <UserContext.Provider value={{ user, ageBracket: 'teen' }}>
        <AdaptiveChoreList chores={mockChores} />
      </UserContext.Provider>
    );
    
    // Teen variant has compact layout
    expect(screen.getByText('Complete')).toBeInTheDocument();
    expect(screen.getByText('View History')).toBeInTheDocument();
  });
});
```

---

## Best Practices

1. **Always calculate age bracket on the server** - Don't trust client-side birthdate manipulation
2. **Support parent overrides** - Some children mature faster/slower than average
3. **Test all three age brackets** - Don't forget young kids in testing
4. **Use consistent emoji sets** - Pick one style (native, Twemoji, etc.)
5. **Respect touch target sizes** - Especially for young children
6. **Celebrate transitions** - Make aging up feel special
7. **Feature-gate appropriately** - Don't overwhelm young kids
8. **Keep young UI simple** - Resist adding complexity
9. **Document age transitions** - Help parents understand what changes
10. **Consider accessibility** - Age-adaptive doesn't mean inaccessible

---

*Last Updated: January 29, 2026*
