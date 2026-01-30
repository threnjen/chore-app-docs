# Frontend Testing Strategy

## Overview

The frontend testing strategy ensures UI components render correctly, user interactions work as expected, age-adaptive features function properly, and the application integrates correctly with the backend API.

## Test Coverage Targets

- **Critical User Flows** (authentication, chore completion, transactions): **100% coverage**
- **Components**: **80% coverage minimum**
- **Integration Tests**: All major user workflows
- **E2E Tests**: Complete user journeys

## Testing Framework

### Core Tools
- **Vitest**: Fast unit test runner (Vite-native)
- **React Testing Library**: Component testing with user-centric approach
- **Playwright**: End-to-end browser testing
- **MSW (Mock Service Worker)**: API mocking for integration tests
- **@testing-library/user-event**: Realistic user interaction simulation

### Installation

```bash
# Install test dependencies
npm install -D vitest @vitest/ui @testing-library/react @testing-library/jest-dom \
  @testing-library/user-event msw playwright @playwright/test
```

## Test Structure

```
tests/
├── components/              # Component unit tests
│   ├── auth/
│   │   ├── LoginForm.test.jsx
│   │   └── RegisterForm.test.jsx
│   ├── chores/
│   │   ├── ChoreCard.test.jsx
│   │   ├── ChoreList.test.jsx
│   │   └── RecurrenceBuilder.test.jsx
│   ├── layout/
│   │   ├── FamilySwitcher.test.jsx
│   │   └── Navigation.test.jsx
│   └── shared/
│       └── AdaptiveComponent.test.jsx
├── integration/            # Integration tests
│   ├── ChoreFlow.test.jsx
│   ├── TransactionFlow.test.jsx
│   └── MultiHousehold.test.jsx
├── e2e/                   # Playwright E2E tests
│   ├── auth.spec.js
│   ├── chore-workflow.spec.js
│   ├── allowance.spec.js
│   └── multi-household.spec.js
├── mocks/                 # MSW API mocks
│   ├── handlers.js
│   └── server.js
├── setup.js              # Test setup and global mocks
└── utils.jsx             # Test utilities and helpers
```

## Component Unit Tests

### Simple Component Test

```javascript
// tests/components/chores/ChoreCard.test.jsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import userEvent from '@testing-library/user-event';
import ChoreCard from '@/components/chores/ChoreCard';

describe('ChoreCard', () => {
  it('displays chore title and reward amount', () => {
    const chore = {
      id: '1',
      title: 'Clean your room',
      description: 'Vacuum and organize',
      reward_amount: 5.00,
      status: 'PENDING'
    };
    
    render(<ChoreCard chore={chore} />);
    
    expect(screen.getByText('Clean your room')).toBeInTheDocument();
    expect(screen.getByText('$5.00')).toBeInTheDocument();
  });
  
  it('calls onComplete when mark complete button is clicked', async () => {
    const user = userEvent.setup();
    const onComplete = vi.fn();
    const chore = {
      id: '1',
      title: 'Clean your room',
      reward_amount: 5.00,
      status: 'PENDING'
    };
    
    render(<ChoreCard chore={chore} onComplete={onComplete} />);
    
    const completeButton = screen.getByRole('button', { name: /mark complete/i });
    await user.click(completeButton);
    
    expect(onComplete).toHaveBeenCalledWith('1');
    expect(onComplete).toHaveBeenCalledTimes(1);
  });
  
  it('shows completed status when chore is completed', () => {
    const chore = {
      id: '1',
      title: 'Clean your room',
      reward_amount: 5.00,
      status: 'COMPLETED'
    };
    
    render(<ChoreCard chore={chore} />);
    
    expect(screen.getByText(/completed/i)).toBeInTheDocument();
    expect(screen.queryByRole('button', { name: /mark complete/i })).not.toBeInTheDocument();
  });
});
```

### Age-Adaptive Component Test

```javascript
// tests/components/shared/AdaptiveComponent.test.jsx
import { render, screen } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { UserContext } from '@/contexts/UserContext';
import ChoreCard from '@/components/chores/ChoreCard';

const renderWithUser = (component, user) => {
  return render(
    <UserContext.Provider value={{ user }}>
      {component}
    </UserContext.Provider>
  );
};

describe('Age-Adaptive ChoreCard', () => {
  const chore = {
    id: '1',
    title: 'Clean room',
    reward_amount: 5.00
  };
  
  it('renders large text for young age bracket (5-9)', () => {
    const youngUser = { birthdate: '2020-01-01', age_bracket: 'YOUNG' };
    
    const { container } = renderWithUser(<ChoreCard chore={chore} />, youngUser);
    
    // Check for large text class
    const title = screen.getByText('Clean room');
    expect(title).toHaveClass('text-2xl'); // Large for young kids
  });
  
  it('renders medium text for tween age bracket (10-13)', () => {
    const tweenUser = { birthdate: '2015-01-01', age_bracket: 'TWEEN' };
    
    const { container } = renderWithUser(<ChoreCard chore={chore} />, tweenUser);
    
    const title = screen.getByText('Clean room');
    expect(title).toHaveClass('text-lg'); // Medium for tweens
  });
  
  it('renders compact text for teen age bracket (14-17)', () => {
    const teenUser = { birthdate: '2011-01-01', age_bracket: 'TEEN' };
    
    const { container } = renderWithUser(<ChoreCard chore={chore} />, teenUser);
    
    const title = screen.getByText('Clean room');
    expect(title).toHaveClass('text-base'); // Compact for teens
  });
  
  it('shows emoji for young users, icons for teens', () => {
    const youngUser = { age_bracket: 'YOUNG' };
    const teenUser = { age_bracket: 'TEEN' };
    
    const { rerender } = renderWithUser(<ChoreCard chore={chore} />, youngUser);
    expect(screen.getByText('✨')).toBeInTheDocument(); // Emoji for young
    
    rerender(
      <UserContext.Provider value={{ user: teenUser }}>
        <ChoreCard chore={chore} />
      </UserContext.Provider>
    );
    expect(screen.queryByText('✨')).not.toBeInTheDocument(); // No emoji for teen
    expect(screen.getByTestId('icon-checkmark')).toBeInTheDocument(); // Icon instead
  });
});
```

### Form Component Test

```javascript
// tests/components/auth/LoginForm.test.jsx
import { render, screen, waitFor } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import userEvent from '@testing-library/user-event';
import LoginForm from '@/components/auth/LoginForm';

describe('LoginForm', () => {
  it('submits form with email and password', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    
    render(<LoginForm onSubmit={onSubmit} />);
    
    // Fill out form
    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    
    // Submit
    await user.click(screen.getByRole('button', { name: /log in/i }));
    
    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'password123'
      });
    });
  });
  
  it('shows validation error for invalid email', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn();
    
    render(<LoginForm onSubmit={onSubmit} />);
    
    await user.type(screen.getByLabelText(/email/i), 'invalid-email');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    await user.click(screen.getByRole('button', { name: /log in/i }));
    
    expect(await screen.findByText(/invalid email/i)).toBeInTheDocument();
    expect(onSubmit).not.toHaveBeenCalled();
  });
  
  it('disables submit button while submitting', async () => {
    const user = userEvent.setup();
    const onSubmit = vi.fn(() => new Promise(resolve => setTimeout(resolve, 100)));
    
    render(<LoginForm onSubmit={onSubmit} />);
    
    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'password123');
    
    const submitButton = screen.getByRole('button', { name: /log in/i });
    await user.click(submitButton);
    
    expect(submitButton).toBeDisabled();
    
    await waitFor(() => {
      expect(submitButton).not.toBeDisabled();
    });
  });
});
```

## Integration Tests

### API Integration with MSW

```javascript
// tests/mocks/handlers.js
import { rest } from 'msw';

export const handlers = [
  // Auth endpoints
  rest.post('http://localhost:8000/api/v1/auth/login', (req, res, ctx) => {
    const { email, password } = req.body;
    
    if (email === 'test@example.com' && password === 'password123') {
      return res(
        ctx.status(200),
        ctx.json({
          success: true,
          data: {
            access_token: 'mock-jwt-token',
            user: {
              id: '1',
              email: 'test@example.com',
              name: 'Test User',
              role: 'PARENT',
              families: [{ id: 'family-1', name: "Test Family" }]
            }
          }
        })
      );
    }
    
    return res(
      ctx.status(401),
      ctx.json({
        success: false,
        error: { message: 'Invalid credentials' }
      })
    );
  }),
  
  // Chores endpoint
  rest.get('http://localhost:8000/api/v1/chores', (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json({
        success: true,
        data: {
          items: [
            {
              id: 'chore-1',
              title: 'Clean room',
              reward_amount: 5.00,
              status: 'PENDING'
            }
          ]
        }
      })
    );
  }),
  
  // Chore completion
  rest.post('http://localhost:8000/api/v1/chore-assignments/:id/complete', (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json({
        success: true,
        data: {
          id: req.params.id,
          status: 'COMPLETED',
          completed_at: new Date().toISOString()
        }
      })
    );
  })
];

// tests/mocks/server.js
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
```

```javascript
// tests/setup.js
import { afterEach, beforeAll, afterAll } from 'vitest';
import { cleanup } from '@testing-library/react';
import '@testing-library/jest-dom';
import { server } from './mocks/server';

// Start MSW server
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));

// Reset handlers after each test
afterEach(() => {
  server.resetHandlers();
  cleanup();
});

// Clean up after all tests
afterAll(() => server.close());
```

### Complete User Flow Test

```javascript
// tests/integration/ChoreFlow.test.jsx
import { render, screen, waitFor } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import userEvent from '@testing-library/user-event';
import { BrowserRouter } from 'react-router-dom';
import { AuthContext } from '@/contexts/AuthContext';
import Dashboard from '@/pages/Dashboard';

const renderWithAuth = (component, authValue) => {
  return render(
    <BrowserRouter>
      <AuthContext.Provider value={authValue}>
        {component}
      </AuthContext.Provider>
    </BrowserRouter>
  );
};

describe('Chore Completion Flow', () => {
  it('completes full chore flow from child perspective', async () => {
    const user = userEvent.setup();
    
    const authValue = {
      user: {
        id: 'child-1',
        name: 'Emma',
        role: 'CHILD',
        age_bracket: 'TWEEN'
      },
      token: 'mock-token'
    };
    
    renderWithAuth(<Dashboard />, authValue);
    
    // Wait for chores to load
    await waitFor(() => {
      expect(screen.getByText('Clean room')).toBeInTheDocument();
    });
    
    // Click mark complete
    const completeButton = screen.getByRole('button', { name: /mark complete/i });
    await user.click(completeButton);
    
    // Should show waiting for approval status
    await waitFor(() => {
      expect(screen.getByText(/waiting for approval/i)).toBeInTheDocument();
    });
    
    // Should show success notification
    expect(screen.getByText(/chore marked complete/i)).toBeInTheDocument();
  });
  
  it('handles API errors gracefully', async () => {
    const user = userEvent.setup();
    
    // Override MSW handler to return error
    const { server } = await import('./mocks/server');
    server.use(
      rest.post('http://localhost:8000/api/v1/chore-assignments/:id/complete', (req, res, ctx) => {
        return res(
          ctx.status(500),
          ctx.json({
            success: false,
            error: { message: 'Server error' }
          })
        );
      })
    );
    
    const authValue = { user: { role: 'CHILD' }, token: 'mock-token' };
    renderWithAuth(<Dashboard />, authValue);
    
    await waitFor(() => {
      expect(screen.getByText('Clean room')).toBeInTheDocument();
    });
    
    const completeButton = screen.getByRole('button', { name: /mark complete/i });
    await user.click(completeButton);
    
    // Should show error notification
    await waitFor(() => {
      expect(screen.getByText(/something went wrong/i)).toBeInTheDocument();
    });
  });
});
```

### Multi-Household Context Switching

```javascript
// tests/integration/MultiHousehold.test.jsx
import { render, screen, waitFor } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import userEvent from '@testing-library/user-event';
import { FamilyContext } from '@/contexts/FamilyContext';
import App from '@/App';

describe('Multi-Household Context Switching', () => {
  it('switches between mom and dad households', async () => {
    const user = userEvent.setup();
    
    const families = [
      { id: 'mom-family', name: "Mom's House" },
      { id: 'dad-family', name: "Dad's House" }
    ];
    
    const switchFamily = vi.fn();
    
    render(
      <FamilyContext.Provider value={{ 
        currentFamily: families[0],
        families,
        switchFamily
      }}>
        <App />
      </FamilyContext.Provider>
    );
    
    // Should show Mom's House initially
    expect(screen.getByText("Mom's House")).toBeInTheDocument();
    
    // Open family switcher
    await user.click(screen.getByRole('button', { name: /switch family/i }));
    
    // Select Dad's House
    await user.click(screen.getByText("Dad's House"));
    
    // Should call switchFamily
    await waitFor(() => {
      expect(switchFamily).toHaveBeenCalledWith('dad-family');
    });
  });
});
```

## End-to-End Tests (Playwright)

### Configuration

```javascript
// playwright.config.js
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  
  use: {
    baseURL: 'http://localhost:5173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure'
  },
  
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
    {
      name: 'Mobile Chrome',
      use: { ...devices['Pixel 5'] },
    },
    {
      name: 'Mobile Safari',
      use: { ...devices['iPhone 12'] },
    },
  ],
  
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Complete E2E Test

```javascript
// tests/e2e/chore-workflow.spec.js
import { test, expect } from '@playwright/test';

test.describe('Complete Chore Workflow', () => {
  test('parent creates chore and child completes it', async ({ browser }) => {
    // Create two browser contexts for parent and child
    const parentContext = await browser.newContext();
    const childContext = await browser.newContext();
    
    const parentPage = await parentContext.newPage();
    const childPage = await childContext.newPage();
    
    // PARENT: Login
    await parentPage.goto('http://localhost:5173/login');
    await parentPage.fill('[name="email"]', 'parent@test.com');
    await parentPage.fill('[name="password"]', 'password123');
    await parentPage.click('button[type="submit"]');
    
    // Wait for dashboard
    await expect(parentPage.locator('h1')).toContainText('Dashboard');
    
    // PARENT: Create chore
    await parentPage.click('text=Create Chore');
    await parentPage.fill('[name="title"]', 'E2E Test Chore');
    await parentPage.fill('[name="description"]', 'This is a test chore');
    await parentPage.fill('[name="reward_amount"]', '15.50');
    await parentPage.selectOption('[name="assigned_to"]', 'Emma');
    await parentPage.click('button:has-text("Create")');
    
    // Should see success message
    await expect(parentPage.locator('text=Chore created')).toBeVisible();
    
    // CHILD: Login
    await childPage.goto('http://localhost:5173/login');
    await childPage.fill('[name="email"]', 'emma@test.com');
    await childPage.fill('[name="password"]', 'password123');
    await childPage.click('button[type="submit"]');
    
    // CHILD: See chore on dashboard
    await expect(childPage.locator('text=E2E Test Chore')).toBeVisible();
    await expect(childPage.locator('text=$15.50')).toBeVisible();
    
    // CHILD: Complete chore
    await childPage.click('text=E2E Test Chore');
    await childPage.click('button:has-text("Mark Complete")');
    
    // Should show waiting for approval
    await expect(childPage.locator('text=Waiting for approval')).toBeVisible();
    
    // PARENT: Approve chore
    await parentPage.click('text=Pending Approvals');
    await expect(parentPage.locator('text=E2E Test Chore')).toBeVisible();
    await parentPage.click('text=E2E Test Chore');
    await parentPage.fill('[name="feedback"]', 'Great job!');
    await parentPage.click('button:has-text("Approve")');
    
    // Should see success
    await expect(parentPage.locator('text=Chore approved')).toBeVisible();
    
    // CHILD: Verify reward received
    await childPage.reload();
    await childPage.click('text=Accounts');
    await expect(childPage.locator('text=$15.50')).toBeVisible();
    
    // CHILD: Check transaction history
    await childPage.click('text=Transactions');
    await expect(childPage.locator('text=Chore Reward: E2E Test Chore')).toBeVisible();
    
    // Cleanup
    await parentContext.close();
    await childContext.close();
  });
  
  test('first-dibs chore can be claimed by any child', async ({ browser }) => {
    const parent = await browser.newContext();
    const child1 = await browser.newContext();
    const child2 = await browser.newContext();
    
    const parentPage = await parent.newPage();
    const child1Page = await child1.newPage();
    const child2Page = await child2.newPage();
    
    // Parent creates first-dibs chore
    await parentPage.goto('http://localhost:5173/login');
    await parentPage.fill('[name="email"]', 'parent@test.com');
    await parentPage.fill('[name="password"]', 'password123');
    await parentPage.click('button[type="submit"]');
    
    await parentPage.click('text=Create Chore');
    await parentPage.fill('[name="title"]', 'First Dibs Chore');
    await parentPage.fill('[name="reward_amount"]', '10');
    await parentPage.selectOption('[name="assignment_type"]', 'FIRST_DIBS');
    await parentPage.click('button:has-text("Create")');
    
    // Both children login
    await child1Page.goto('http://localhost:5173/login');
    await child1Page.fill('[name="email"]', 'emma@test.com');
    await child1Page.fill('[name="password"]', 'password123');
    await child1Page.click('button[type="submit"]');
    
    await child2Page.goto('http://localhost:5173/login');
    await child2Page.fill('[name="email"]', 'lucas@test.com');
    await child2Page.fill('[name="password"]', 'password123');
    await child2Page.click('button[type="submit"]');
    
    // Both should see chore
    await expect(child1Page.locator('text=First Dibs Chore')).toBeVisible();
    await expect(child2Page.locator('text=First Dibs Chore')).toBeVisible();
    
    // First child claims it
    await child1Page.click('text=First Dibs Chore');
    await child1Page.click('button:has-text("Claim")');
    
    await expect(child1Page.locator('text=Claimed')).toBeVisible();
    
    // Second child should see it's already claimed
    await child2Page.reload();
    await expect(child2Page.locator('text=First Dibs Chore')).not.toBeVisible();
    
    // Cleanup
    await parent.close();
    await child1.close();
    await child2.close();
  });
});
```

### PWA Testing

```javascript
// tests/e2e/pwa.spec.js
import { test, expect } from '@playwright/test';

test.describe('PWA Features', () => {
  test('can be installed as PWA', async ({ page }) => {
    await page.goto('http://localhost:5173');
    
    // Check for service worker registration
    const swRegistered = await page.evaluate(() => {
      return 'serviceWorker' in navigator;
    });
    expect(swRegistered).toBe(true);
    
    // Check for web manifest
    const manifestLink = await page.locator('link[rel="manifest"]');
    await expect(manifestLink).toHaveAttribute('href', '/manifest.json');
  });
  
  test('works offline', async ({ page, context }) => {
    // Go online first
    await page.goto('http://localhost:5173/login');
    
    // Login
    await page.fill('[name="email"]', 'emma@test.com');
    await page.fill('[name="password"]', 'password123');
    await page.click('button[type="submit"]');
    
    // Wait for dashboard
    await expect(page.locator('h1')).toContainText('Dashboard');
    
    // Go offline
    await context.setOffline(true);
    
    // Navigate to chores page
    await page.click('text=Chores');
    
    // Should show cached content
    await expect(page.locator('h1')).toContainText('Chores');
    
    // Should show offline indicator
    await expect(page.locator('text=Offline')).toBeVisible();
  });
});
```

## Running Tests

```bash
# Run unit and component tests
npm run test

# Run tests in watch mode (during development)
npm run test:watch

# Run tests with coverage
npm run test:coverage

# Run tests in UI mode (interactive)
npm run test:ui

# Run E2E tests
npm run test:e2e

# Run E2E tests in UI mode
npm run test:e2e:ui

# Run E2E tests in headed mode (see browser)
npm run test:e2e:headed

# Run specific E2E test file
npx playwright test chore-workflow

# Run tests in specific browser
npx playwright test --project=firefox

# Generate Playwright report
npx playwright show-report

# View coverage report
open coverage/index.html
```

## Test Utilities

```javascript
// tests/utils.jsx
import { render } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { AuthContext } from '@/contexts/AuthContext';
import { FamilyContext } from '@/contexts/FamilyContext';
import { UserContext } from '@/contexts/UserContext';

// Render with all providers
export function renderWithProviders(
  component,
  {
    auth = { user: null, token: null, login: vi.fn(), logout: vi.fn() },
    family = { currentFamily: null, families: [], switchFamily: vi.fn() },
    user = { user: null, updateUser: vi.fn() },
    route = '/'
  } = {}
) {
  window.history.pushState({}, 'Test page', route);
  
  return render(
    <BrowserRouter>
      <AuthContext.Provider value={auth}>
        <FamilyContext.Provider value={family}>
          <UserContext.Provider value={user}>
            {component}
          </UserContext.Provider>
        </FamilyContext.Provider>
      </AuthContext.Provider>
    </BrowserRouter>
  );
}

// Create mock user
export function createMockUser(overrides = {}) {
  return {
    id: 'user-1',
    email: 'test@example.com',
    name: 'Test User',
    role: 'CHILD',
    age_bracket: 'TWEEN',
    birthdate: '2014-01-01',
    families: [
      { id: 'family-1', name: 'Test Family' }
    ],
    ...overrides
  };
}

// Create mock chore
export function createMockChore(overrides = {}) {
  return {
    id: 'chore-1',
    title: 'Test Chore',
    description: 'Test description',
    reward_amount: 5.00,
    status: 'PENDING',
    assignment_type: 'ASSIGNED',
    ...overrides
  };
}
```

## Continuous Integration

```yaml
# .github/workflows/frontend-tests.yml
name: Frontend Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:coverage
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
      
      - name: Install Playwright
        run: npx playwright install --with-deps
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload Playwright report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/
```

## Best Practices

1. **Test User Behavior**: Test what users see and do, not implementation details
2. **Accessible Queries**: Use `getByRole`, `getByLabelText` over `getByTestId`
3. **Realistic Interactions**: Use `userEvent` for realistic user interactions
4. **Async Handling**: Always use `waitFor` for async operations
5. **Clean Mocks**: Reset mocks after each test
6. **Visual Regression**: Consider adding visual regression tests for UI components
7. **Mobile Testing**: Test responsive behavior and mobile-specific features
8. **Age-Adaptive Testing**: Test all three age brackets for adaptive components

## Next Steps

- Review [Component Architecture](COMPONENT_ARCHITECTURE.md) for component design
- Check [Age-Adaptive UI Guide](AGE_ADAPTIVE_UI_GUIDE.md) for UI testing patterns
- Read [Local Development Setup](LOCAL_DEVELOPMENT_SETUP.md) for test environment setup
- Explore [PWA Implementation](PWA_IMPLEMENTATION.md) for offline testing strategies
