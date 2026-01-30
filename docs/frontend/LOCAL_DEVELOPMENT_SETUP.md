# Frontend Local Development Setup

## Prerequisites
- macOS (or Linux/Windows with adaptations)
- Git
- Node.js 20+
- npm or yarn

## Node.js Installation

```bash
# Install Node.js via Homebrew
brew install node@20

# Verify installation
node --version  # Should be 20.x
npm --version   # Should be 10.x
```

## Frontend Setup

```bash
# Navigate to frontend directory
cd chore-app-frontend

# Install dependencies
npm install

# Copy environment template
cp .env.example .env

# Edit .env with your settings
```

### Environment Variables

Edit `.env` with the following variables:

```bash
# API Configuration
VITE_API_BASE_URL=http://localhost:8000/api/v1

# OAuth Client IDs (Frontend needs these for OAuth flows)
VITE_GOOGLE_CLIENT_ID=your_google_client_id
VITE_APPLE_CLIENT_ID=your_apple_client_id
VITE_FACEBOOK_CLIENT_ID=your_facebook_client_id

# Feature Flags (Development)
VITE_ENABLE_PWA=true
VITE_ENABLE_OFFLINE_MODE=true
VITE_ENABLE_PUSH_NOTIFICATIONS=false

# Environment
VITE_ENV=development
```

> **Note**: All Vite environment variables must be prefixed with `VITE_` to be exposed to the client-side code.

## Run Development Server

```bash
# Start development server with hot reload
npm run dev

# Server will start at http://localhost:5173
# API proxy will forward /api requests to backend
```

### Development Server Features

- **Hot Module Replacement (HMR)**: Changes reflect instantly without full reload
- **API Proxy**: `/api/*` requests are proxied to `http://localhost:8000`
- **Fast Refresh**: React components hot-reload preserving state
- **Error Overlay**: Build and runtime errors displayed in browser

## Build for Production

```bash
# Create optimized production build
npm run build

# Preview production build locally
npm run preview

# Production build will be in dist/ directory
```

## Docker Setup (Optional)

```bash
# From frontend directory
docker build -t picklesapp-frontend .

# Run container
docker run -d \
  --name picklesapp-frontend \
  -p 5173:5173 \
  --env-file .env \
  picklesapp-frontend

# View logs
docker logs -f picklesapp-frontend

# Stop container
docker stop picklesapp-frontend
docker rm picklesapp-frontend
```

## Testing

### Unit and Component Tests (Vitest)

```bash
# Run all tests
npm run test

# Run tests in watch mode
npm run test:watch

# Run tests with coverage
npm run test:coverage

# View coverage report
open coverage/index.html
```

### End-to-End Tests (Playwright)

```bash
# Install Playwright browsers (first time only)
npx playwright install

# Run E2E tests
npm run test:e2e

# Run E2E tests in UI mode (interactive)
npm run test:e2e:ui

# Run E2E tests in headed mode (see browser)
npm run test:e2e:headed

# Generate Playwright report
npx playwright show-report
```

## Common Development Tasks

### Install New Package

```bash
# Install production dependency
npm install package-name

# Install dev dependency
npm install -D package-name

# Update package.json
npm install
```

### Update Dependencies

```bash
# Check for outdated packages
npm outdated

# Update all packages to latest (within semver range)
npm update

# Update specific package
npm update package-name

# Audit for security vulnerabilities
npm audit
npm audit fix
```

### Linting and Formatting

```bash
# Lint code with ESLint
npm run lint

# Fix auto-fixable issues
npm run lint:fix

# Format code with Prettier (if configured)
npm run format
```

### Clear Cache

```bash
# Clear npm cache
npm cache clean --force

# Remove node_modules and reinstall
rm -rf node_modules package-lock.json
npm install

# Clear Vite cache
rm -rf node_modules/.vite
```

## Project Structure

```
chore-app-frontend/
├── public/              # Static assets
│   ├── favicon.ico
│   └── manifest.json   # PWA manifest
├── src/
│   ├── components/      # Reusable UI components
│   │   ├── auth/       # Authentication components
│   │   ├── chores/     # Chore management components
│   │   ├── layout/     # Layout components
│   │   └── shared/     # Shared/common components
│   ├── pages/          # Page components (routes)
│   ├── contexts/       # React Context providers
│   ├── hooks/          # Custom React hooks
│   ├── utils/          # Utility functions
│   ├── styles/         # Global styles (Tailwind)
│   ├── assets/         # Images, icons, fonts
│   ├── App.jsx         # Root component
│   └── main.jsx        # Entry point
├── tests/
│   ├── components/     # Component tests
│   ├── integration/    # Integration tests
│   └── e2e/           # Playwright E2E tests
├── .env.example        # Environment template
├── index.html         # HTML entry point
├── package.json       # Dependencies and scripts
├── vite.config.js     # Vite configuration
├── tailwind.config.js # Tailwind CSS configuration
├── vitest.config.js   # Vitest configuration
└── playwright.config.js # Playwright configuration
```

## Vite Configuration

The `vite.config.js` includes:
- **API Proxy**: Forwards `/api` requests to backend
- **PWA Plugin**: Service worker generation
- **React Plugin**: Fast Refresh and JSX support
- **Path Aliases**: `@/` maps to `src/`
- **Build Optimization**: Code splitting, tree shaking

## Tailwind CSS

```bash
# Tailwind classes are processed automatically
# Configuration in tailwind.config.js

# Age-adaptive themes are configured:
# - Young (5-9): Large text, bright colors
# - Tween (10-13): Medium text, balanced design
# - Teen (14-17): Compact text, mature design
```

## PWA Development

```bash
# PWA manifest is in public/manifest.json
# Service worker is auto-generated during build

# Test PWA locally:
npm run build
npm run preview

# Use Lighthouse in Chrome DevTools to audit PWA
```

## Browser DevTools

### React Developer Tools
```bash
# Install React DevTools extension for:
# - Chrome
# - Firefox
# - Edge

# Features:
# - Inspect component hierarchy
# - View props and state
# - Profile performance
```

### Vite DevTools
- Network tab shows HMR updates
- Console shows build times and errors
- Source maps enabled for debugging

## Troubleshooting

### Port Already in Use

```bash
# Find process using port 5173
lsof -i :5173

# Kill process
kill -9 <PID>

# Or use different port
npm run dev -- --port 3000
```

### API Connection Issues

```bash
# Verify backend is running
curl http://localhost:8000/api/v1/health

# Check proxy configuration in vite.config.js
# Ensure VITE_API_BASE_URL is correct in .env
```

### Module Not Found Errors

```bash
# Clear node_modules and reinstall
rm -rf node_modules package-lock.json
npm install

# Clear Vite cache
rm -rf node_modules/.vite
```

### Hot Reload Not Working

```bash
# Restart dev server
# Check file watcher limits (Linux/macOS)
# Verify files are saved correctly

# macOS: Increase file watcher limit
# Add to ~/.zshrc or ~/.bashrc:
ulimit -n 10240
```

## Port Configuration

- **Frontend Dev Server**: `5173`
- **Frontend Preview**: `4173`
- **Backend API** (proxied): `8000`

## Integration with Backend

### Authentication Flow
1. Frontend makes auth request to backend
2. Backend returns JWT token
3. Frontend stores token in `localStorage`
4. Frontend includes token in `Authorization` header
5. API proxy forwards authenticated requests

### Family Context
1. User selects family from dropdown
2. Frontend calls `/families/switch` endpoint
3. Backend returns new JWT with `family_id`
4. Frontend updates context and reloads data

## Next Steps

- Review [Component Architecture](COMPONENT_ARCHITECTURE.md)
- Explore [Age-Adaptive UI Guide](AGE_ADAPTIVE_UI_GUIDE.md)
- Read [State Management](STATE_MANAGEMENT.md) strategy
- Check [PWA Implementation](PWA_IMPLEMENTATION.md) guide
- Review [Testing Strategy](TESTING.md)
