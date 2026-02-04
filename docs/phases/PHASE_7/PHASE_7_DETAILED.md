# Phase 7: Age-Adaptive Mobile UI
**Duration: 2-3 weeks**

> **Prerequisite**: Requires AWS deployment (Phase 5) for mobile device testing with secure HTTPS access.

---

## Overview

Phase 7 implements the age-bracket UI system, PWA optimization, and responsive design. The interface automatically adapts to the child's age group, providing age-appropriate features and visual design.

---

## Goals

- Age-bracket UI system
- Mobile PWA optimization
- Responsive design testing and refinement
- Feature gating

---

## Deliverables

### 1. Backend

- Age bracket calculation logic
- Feature access matrix
- Preset profile system
- Age-up notification system

### 2. Frontend

- Age-adaptive component system
- Young (5-9) UI components
- Tween (10-13) UI components
- Teen (14-17) UI components
- Feature gating logic
- Parent override for age bracket
- Age-up celebration modal
- PWA manifest and service worker
- Offline support

### 3. Features

- Automatic age bracket detection
- Age-appropriate dashboards
- Simplified young kid UI (big icons, minimal text)
- Full-featured teen UI
- Feature gating (transfers, history, budgets)
- Birthday celebrations
- PWA installation
- Offline chore viewing

---

## Age Brackets

| Bracket | Ages | Characteristics | UI Approach |
|---------|------|-----------------|-------------|
| **Young** | 5-9 | Early readers, concrete thinking, needs structure | Large icons, minimal text, gamified, limited features |
| **Tween** | 10-13 | Developing independence, abstract thinking emerging | Balanced text/icons, more features, educational |
| **Teen** | 14-17 | Adult-like cognition, financial planning | Full features, adult interface, investment focus |

---

## Age Calculation Logic

### Backend Implementation

```python
from datetime import date
from typing import Optional

def calculate_age(birthdate: date) -> int:
    """Calculate current age from birthdate."""
    if not birthdate:
        return 0
    
    today = date.today()
    age = today.year - birthdate.year
    
    # Adjust for birthday not yet occurred this year
    if (today.month, today.day) < (birthdate.month, birthdate.day):
        age -= 1
    
    return age

def calculate_age_bracket(birthdate: Optional[date]) -> str:
    """
    Calculate current age and determine UI bracket.
    """
    if not birthdate:
        return "teen"  # Default for no birthdate
    
    age = calculate_age(birthdate)
    
    if age < 10:
        return "young"
    elif age < 14:
        return "tween"
    else:
        return "teen"

def get_effective_age_bracket(user, family_membership) -> str:
    """
    Get age bracket with parent override support.
    """
    if family_membership.age_bracket_override:
        return family_membership.age_bracket_override
    
    return calculate_age_bracket(user.birthdate)
```

### Database Schema

```sql
-- Add age bracket override to family_memberships
ALTER TABLE family_memberships 
ADD COLUMN age_bracket_override VARCHAR(20) 
CHECK (age_bracket_override IN ('young', 'tween', 'teen', NULL));
```

---

## Feature Access Matrix

```python
FEATURE_ACCESS = {
    "young": {
        "view_chores": True,
        "complete_chores": True,
        "view_balance": True,
        "view_goals": True,
        "request_money": False,
        "transfer_between_accounts": False,
        "view_transaction_history": False,
        "manage_budget": False,
        "view_investment_accounts": False,
        "set_own_goals": False,
        "view_analytics": False,
    },
    "tween": {
        "view_chores": True,
        "complete_chores": True,
        "view_balance": True,
        "view_goals": True,
        "request_money": True,
        "transfer_between_accounts": True,  # With limits
        "view_transaction_history": True,
        "manage_budget": False,
        "view_investment_accounts": False,
        "set_own_goals": True,
        "view_analytics": False,
    },
    "teen": {
        "view_chores": True,
        "complete_chores": True,
        "view_balance": True,
        "view_goals": True,
        "request_money": True,
        "transfer_between_accounts": True,
        "view_transaction_history": True,
        "manage_budget": True,
        "view_investment_accounts": True,
        "set_own_goals": True,
        "view_analytics": True,
    }
}
```

---

## UI Component Variants

### Navigation by Age Bracket

**Young (5-9):**
```jsx
<BottomNav>
  <NavItem icon="🎯" label="">Chores</NavItem>
  <NavItem icon="💰" label="">Money</NavItem>
  <NavItem icon="⭐" label="">Goals</NavItem>
</BottomNav>
```

**Tween (10-13):**
```jsx
<BottomNav>
  <NavItem icon={<CurrencyDollarIcon />} label="My Money" />
  <NavItem icon={<ClipboardListIcon />} label="Chores" />
  <NavItem icon={<ChartBarIcon />} label="Goals" />
  <NavItem icon={<CogIcon />} label="Settings" />
</BottomNav>
```

**Teen (14-17):**
```jsx
<SideNav>
  <NavSection title="Financial">
    <NavItem>Accounts</NavItem>
    <NavItem>Transactions</NavItem>
    <NavItem>Budget</NavItem>
  </NavSection>
  <NavSection title="Tasks">
    <NavItem>Chores</NavItem>
    <NavItem>Goals</NavItem>
  </NavSection>
</SideNav>
```

### Dashboard by Age Bracket

**Young Dashboard:**
```jsx
<Dashboard>
  {/* Big cards with emoji */}
  <BalanceCard>
    <BigEmoji>💰</BigEmoji>
    <BigNumber>${balance}</BigNumber>
    <SimpleLabel>You have</SimpleLabel>
  </BalanceCard>
  
  <ChoresCard>
    <BigEmoji>🎯</BigEmoji>
    <BigNumber>{pendingChores}</BigNumber>
    <SimpleLabel>Chores to do</SimpleLabel>
  </ChoresCard>
  
  {/* Visual progress bars */}
  <GoalCard>
    <BigEmoji>🎮</BigEmoji>
    <ProgressBar value={75} max={100} />
    <SimpleLabel>$45 of $60 saved!</SimpleLabel>
  </GoalCard>
</Dashboard>
```

**Tween Dashboard:**
```jsx
<Dashboard>
  <BalanceCard>
    <Label>Total Balance</Label>
    <Amount>${totalBalance}</Amount>
    <AccountBreakdown>
      <div>Spending: ${spending}</div>
      <div>Saving: ${saving}</div>
    </AccountBreakdown>
  </BalanceCard>
  
  <ChoreList>
    <Header>Today's Chores ({pending})</Header>
    {chores.map(chore => (
      <ChoreItem key={chore.id} chore={chore} />
    ))}
  </ChoreList>
  
  <RecentActivity>
    <Header>Recent</Header>
    {transactions.slice(0, 5).map(tx => (
      <TransactionItem key={tx.id} transaction={tx} />
    ))}
  </RecentActivity>
</Dashboard>
```

**Teen Dashboard:**
```jsx
<Dashboard>
  {/* Multi-column layout */}
  <Grid cols={3}>
    <AccountSummary accounts={accounts} />
    <IncomeExpenseChart data={chartData} />
    <UpcomingPayments allowances={allowances} />
  </Grid>
  
  <Grid cols={2}>
    <ChoreSchedule />
    <GoalProgress goals={goals} />
  </Grid>
  
  <FullWidthCard>
    <TransactionTable 
      transactions={transactions} 
      filters={filters}
      exportable
    />
  </FullWidthCard>
</Dashboard>
```

### Chore Card by Age Bracket

**Young:**
```jsx
<ChoreCard chore={chore}>
  <BigIcon>🧹</BigIcon>
  <Title>Clean your room</Title>
  
  <BigButton onClick={complete}>
    ✓ I did it!
  </BigButton>
</ChoreCard>

{/* Celebration on completion */}
<CompletionModal>
  <BigEmoji>🎉</BigEmoji>
  <Title>Great job!</Title>
  <Reward>You earned $5! 💰</Reward>
  <Button>Yay!</Button>
</CompletionModal>
```

**Tween:**
```jsx
<ChoreCard chore={chore}>
  <Header>
    <Title>Clean your room</Title>
    <Reward>$5</Reward>
  </Header>
  
  <Description>
    Make bed, pick up toys, vacuum
  </Description>
  
  <Footer>
    <DueDate>Due: Today at 8 PM</DueDate>
    <Button onClick={complete}>Mark Complete</Button>
  </Footer>
</ChoreCard>

{/* Optional photo upload */}
<CompletionModal>
  <Title>Mark Complete</Title>
  <PhotoUpload optional />
  <Note>Waiting for parent approval</Note>
  <Button>Submit</Button>
</CompletionModal>
```

**Teen:**
```jsx
<ChoreCard chore={chore}>
  <Header>
    <Title>{chore.title}</Title>
    <StatusBadge status={chore.status} />
  </Header>
  
  <Details>
    <Detail label="Reward">${chore.reward}</Detail>
    <Detail label="Due">{formatDate(chore.dueDate)}</Detail>
    <Detail label="Estimated time">{chore.duration}min</Detail>
  </Details>
  
  <Actions>
    <Button onClick={complete}>Complete</Button>
    <Button variant="secondary" onClick={viewHistory}>
      History
    </Button>
  </Actions>
</ChoreCard>
```

---

## Typography & Sizing

```css
/* Young (5-9) */
.young {
  --font-size-base: 18px;
  --font-size-large: 32px;
  --button-height: 60px;
  --touch-target-min: 60px;
  --spacing-base: 24px;
}

/* Tween (10-13) */
.tween {
  --font-size-base: 16px;
  --font-size-large: 24px;
  --button-height: 48px;
  --touch-target-min: 48px;
  --spacing-base: 16px;
}

/* Teen (14-17) */
.teen {
  --font-size-base: 14px;
  --font-size-large: 20px;
  --button-height: 40px;
  --touch-target-min: 44px;
  --spacing-base: 12px;
}
```

---

## Age-Adaptive Component Wrapper

```jsx
// components/AdaptiveComponent.jsx

import { useUser } from '@/contexts/UserContext';
import { FEATURE_ACCESS } from '@/constants/features';

export function AdaptiveComponent({ children, feature }) {
  const { user, ageBracket } = useUser();
  
  // Check feature access
  if (feature && !FEATURE_ACCESS[ageBracket][feature]) {
    return <FeatureLockedMessage ageBracket={ageBracket} feature={feature} />;
  }
  
  // Render age-appropriate version
  return (
    <div className={`ui-${ageBracket}`}>
      {children}
    </div>
  );
}

// Usage
<AdaptiveComponent feature="view_transaction_history">
  {ageBracket === 'young' && <SimpleTransactionList />}
  {ageBracket === 'tween' && <TransactionList />}
  {ageBracket === 'teen' && <TransactionTable />}
</AdaptiveComponent>
```

---

## Age-Up Celebration

### Background Job

```python
# Check for birthdays daily
scheduler.add_job(
    func=check_age_bracket_transitions,
    trigger="cron",
    hour=0,  # Midnight
    id="age_bracket_checker"
)

def check_age_bracket_transitions():
    """
    Check if any kids aged into new bracket today.
    Send notification if bracket changed.
    """
    today = date.today()
    users = User.query.filter(User.role == "CHILD").all()
    
    for user in users:
        if is_birthday_today(user.birthdate):
            old_bracket = get_age_bracket(user.birthdate - timedelta(days=1))
            new_bracket = get_age_bracket(user.birthdate)
            
            if old_bracket != new_bracket:
                send_age_up_notification(user, new_bracket)
                record_age_bracket_change(user, old_bracket, new_bracket)
```

### Frontend Celebration Modal

```jsx
<AgeUpModal>
  <Confetti />
  <Emoji>🎂</Emoji>
  <Title>Happy Birthday, {name}!</Title>
  <Subtitle>You've leveled up! 🎉</Subtitle>
  
  <FeatureShowcase>
    <Title>Check out your new features:</Title>
    <FeatureList>
      {newFeatures.map(feature => (
        <Feature key={feature.id}>
          <Icon>{feature.icon}</Icon>
          <Name>{feature.name}</Name>
          <Description>{feature.description}</Description>
        </Feature>
      ))}
    </FeatureList>
  </FeatureShowcase>
  
  <Button>Explore my new dashboard!</Button>
</AgeUpModal>
```

### New Features by Bracket Transition

```javascript
const BRACKET_FEATURES = {
  'young_to_tween': [
    { icon: '💸', name: 'Money Transfers', description: 'Move money between your accounts' },
    { icon: '📜', name: 'Transaction History', description: 'See all your past transactions' },
    { icon: '🎯', name: 'Set Your Own Goals', description: 'Create savings goals for things you want' },
    { icon: '💬', name: 'Request Money', description: 'Ask parents for money when you need it' },
  ],
  'tween_to_teen': [
    { icon: '📊', name: 'Budget Worksheet', description: 'Plan how you spend your money' },
    { icon: '📈', name: 'Analytics Dashboard', description: 'See charts and graphs of your finances' },
    { icon: '🏦', name: 'Investment Accounts', description: 'Track investment and long-term savings' },
    { icon: '💹', name: 'Advanced Reports', description: 'Detailed financial reports and exports' },
  ]
};
```

---

## Preset Parenting Profiles

```python
PARENTING_PRESETS = {
    "young": {
        "strict": {
            "allow_negative_balances": False,
            "transfer_limit": None,  # No transfers
            "require_chore_approval": True,
            "auto_approve_hours": None,
            "notification_preferences": {
                "chore_reminders": True,
                "parent_gets_completion_notifications": True
            }
        },
        "balanced": {
            "allow_negative_balances": False,
            "transfer_limit": 10.00,
            "require_chore_approval": True,
            "auto_approve_hours": 24,
            "notification_preferences": {
                "chore_reminders": True,
                "parent_gets_completion_notifications": False
            }
        },
        "flexible": {
            "allow_negative_balances": True,
            "negative_limit": 5.00,
            "transfer_limit": 20.00,
            "require_chore_approval": False,
            "notification_preferences": {
                "chore_reminders": False,
                "parent_gets_completion_notifications": False
            }
        }
    },
    "tween": {
        "strict": {
            "allow_negative_balances": False,
            "transfer_limit": 25.00,
            "require_chore_approval": True,
        },
        "balanced": {
            "allow_negative_balances": True,
            "negative_limit": 20.00,
            "transfer_limit": 50.00,
            "require_chore_approval": True,
            "auto_approve_hours": 12,
        },
        "flexible": {
            "allow_negative_balances": True,
            "negative_limit": 50.00,
            "transfer_limit": None,  # No limit
            "require_chore_approval": False,
        }
    },
    "teen": {
        "strict": {
            "allow_negative_balances": True,
            "negative_limit": 50.00,
            "transfer_limit": 100.00,
            "require_chore_approval": False,
        },
        "balanced": {
            "allow_negative_balances": True,
            "negative_limit": 100.00,
            "transfer_limit": None,
            "require_chore_approval": False,
        },
        "flexible": {
            # Almost full autonomy
            "allow_negative_balances": True,
            "negative_limit": 200.00,
            "transfer_limit": None,
            "require_chore_approval": False,
        }
    }
}
```

---

## PWA Implementation

### Vite PWA Plugin Configuration

```javascript
// vite.config.js
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    react(),
    VitePWA({
      registerType: 'autoUpdate',
      includeAssets: ['favicon.ico', 'robots.txt', 'apple-touch-icon.png'],
      manifest: {
        name: 'PicklesApp',
        short_name: 'PicklesApp',
        description: 'Family Chore & Allowance Management',
        theme_color: '#10B981',
        background_color: '#ffffff',
        display: 'standalone',
        orientation: 'portrait',
        start_url: '/',
        icons: [
          {
            src: 'pwa-192x192.png',
            sizes: '192x192',
            type: 'image/png'
          },
          {
            src: 'pwa-512x512.png',
            sizes: '512x512',
            type: 'image/png',
            purpose: 'any maskable'
          }
        ]
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,woff2}'],
        runtimeCaching: [
          {
            urlPattern: /^https:\/\/api\.picklesapp\.com\/api\/v1\/.*/i,
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache',
              expiration: {
                maxEntries: 100,
                maxAgeSeconds: 60 * 60 // 1 hour
              },
              cacheableResponse: {
                statuses: [0, 200]
              }
            }
          }
        ]
      }
    })
  ]
});
```

### Offline Support

```javascript
// hooks/useOffline.js

import { useState, useEffect } from 'react';

export function useOffline() {
  const [isOffline, setIsOffline] = useState(!navigator.onLine);
  
  useEffect(() => {
    const handleOnline = () => setIsOffline(false);
    const handleOffline = () => setIsOffline(true);
    
    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);
  
  return isOffline;
}

// Offline indicator component
function OfflineBanner() {
  const isOffline = useOffline();
  
  if (!isOffline) return null;
  
  return (
    <div className="bg-yellow-500 text-white px-4 py-2 text-center">
      📴 You're offline. Some features may be limited.
    </div>
  );
}
```

### Cached Data Access

```javascript
// services/offlineStorage.js

const DB_NAME = 'picklesapp-offline';
const STORE_NAME = 'cached-data';

export async function cacheData(key, data) {
  const db = await openDB();
  const tx = db.transaction(STORE_NAME, 'readwrite');
  await tx.store.put({ key, data, timestamp: Date.now() });
}

export async function getCachedData(key) {
  const db = await openDB();
  const result = await db.get(STORE_NAME, key);
  return result?.data;
}

// Cache chores for offline viewing
async function fetchChores() {
  try {
    const response = await api.get('/chores');
    await cacheData('chores', response.data);
    return response.data;
  } catch (error) {
    if (!navigator.onLine) {
      const cached = await getCachedData('chores');
      if (cached) return cached;
    }
    throw error;
  }
}
```

---

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/users/{id}/age-bracket` | GET | Get user's age bracket |
| `/api/v1/users/{id}/age-bracket/override` | PUT | Parent override age bracket |
| `/api/v1/users/{id}/features` | GET | Get accessible features for user |
| `/api/v1/presets` | GET | Get all parenting presets |
| `/api/v1/presets/{bracket}/{style}` | GET | Get specific preset |

---

## Testing Requirements

### Age Bracket Tests

```python
def test_age_bracket_young():
    """Test young age bracket (5-9)."""
    user = create_user(birthdate=date(2019, 6, 15))  # 6 years old
    assert calculate_age_bracket(user.birthdate) == 'young'

def test_age_bracket_tween():
    """Test tween age bracket (10-13)."""
    user = create_user(birthdate=date(2014, 6, 15))  # 11 years old
    assert calculate_age_bracket(user.birthdate) == 'tween'

def test_age_bracket_teen():
    """Test teen age bracket (14+)."""
    user = create_user(birthdate=date(2010, 6, 15))  # 15 years old
    assert calculate_age_bracket(user.birthdate) == 'teen'

def test_age_bracket_override():
    """Test parent can override age bracket."""
    user = create_user(birthdate=date(2017, 6, 15))  # 8 years old (young)
    membership = get_family_membership(user.id)
    membership.age_bracket_override = 'tween'
    
    assert get_effective_age_bracket(user, membership) == 'tween'

def test_feature_access_young():
    """Test young users can't access restricted features."""
    response = client.get('/budgets', headers=young_user_headers)
    assert response.status_code == 403

def test_age_up_notification():
    """Test notification sent when kid ages into new bracket."""
    user = create_user(birthdate=date.today().replace(year=date.today().year - 10))
    
    check_age_bracket_transitions()
    
    # Should have sent notification for young -> tween transition
    notifications = get_notifications(user.id)
    assert any(n.type == 'AGE_BRACKET_CHANGE' for n in notifications)
```

### PWA Tests

```javascript
describe('PWA Functionality', () => {
  test('app is installable', async () => {
    // Check manifest is valid
    const response = await fetch('/manifest.json');
    const manifest = await response.json();
    
    expect(manifest.name).toBe('PicklesApp');
    expect(manifest.icons).toHaveLength(2);
  });
  
  test('service worker registers', async () => {
    const registration = await navigator.serviceWorker.register('/sw.js');
    expect(registration.active).toBeTruthy();
  });
  
  test('offline chores are cached', async () => {
    // Fetch chores while online
    await fetchChores();
    
    // Go offline
    await page.setOfflineMode(true);
    
    // Should still get cached chores
    const chores = await fetchChores();
    expect(chores).toBeDefined();
  });
});
```

### Coverage Requirements

- Age bracket calculation: 100%
- Feature access checks: 100%
- Preset profiles: 80%
- PWA functionality: 80%
- Offline mode: 80%

---

## Definition of Done

### Must Have
- [ ] Automatic age bracket detection working
- [ ] Young, Tween, Teen UI variants implemented
- [ ] Feature gating by age bracket
- [ ] Parent can override age bracket
- [ ] Age-up celebration modal
- [ ] PWA installable on mobile
- [ ] Basic offline support (cached chore viewing)
- [ ] Responsive design tested on real devices
- [ ] All tests passing with required coverage

### Nice to Have
- [ ] Animated transitions between brackets
- [ ] Customizable UI themes per kid
- [ ] Offline chore completion (sync when online)
- [ ] Add to home screen prompt

---

## Dependencies

- **Requires**: Phase 5 (AWS deployment), Phase 6 (core features)
- **Enables**: Phase 10 (native app wrapping)

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| UI too simple for mature young kids | Parent override option |
| PWA not supported on older browsers | Graceful fallback to web app |
| Offline sync conflicts | Queue changes, resolve on sync |
| Birthday detection across timezones | Use family timezone for calculation |

---

*Phase 7 Document - Last Updated: February 2026*
