# PicklesApp Documentation

**Family Chore & Allowance Management Platform**

*"Your parenting, your way"*

---

## Overview

This repository contains comprehensive documentation for PicklesApp, a family financial management platform that combines chore management, allowance automation, and financial education for children.

## Quick Links

- **[Backend Repository](https://github.com/threnjen/chore-app-backend)** - FastAPI backend service
- **[Frontend Repository](https://github.com/threnjen/chore-app-frontend)** - React PWA frontend

## Documentation Structure

### 📘 Backend Documentation

Located in `docs/backend/`:

| Document | Description |
|----------|-------------|
| **[Database Schema](docs/backend/DATABASE_SCHEMA.md)** | Complete database structure with DDL, indexes, and Row-Level Security policies |
| **[Local Development Setup](docs/backend/LOCAL_DEVELOPMENT_SETUP.md)** | PostgreSQL installation, Python environment, and backend development setup |
| **[Testing Strategy](docs/backend/TESTING.md)** | Backend testing with pytest, factories, integration tests, and load testing |

### 🎨 Frontend Documentation

Located in `docs/frontend/`:

| Document | Description |
|----------|-------------|
| **[Component Architecture](docs/frontend/COMPONENT_ARCHITECTURE.md)** | Component organization, patterns, composition strategies, and code splitting |
| **[State Management](docs/frontend/STATE_MANAGEMENT.md)** | React Context architecture, custom hooks, and API integration patterns |
| **[Age-Adaptive UI Guide](docs/frontend/AGE_ADAPTIVE_UI_GUIDE.md)** | Implementing age-adaptive interfaces for three age brackets (5-9, 10-13, 14-17) |
| **[PWA Implementation](docs/frontend/PWA_IMPLEMENTATION.md)** | Service worker setup, offline support, background sync, and push notifications |
| **[Styling Guide](docs/frontend/STYLING_GUIDE.md)** | Tailwind CSS conventions, design system, and responsive patterns |
| **[Local Development Setup](docs/frontend/LOCAL_DEVELOPMENT_SETUP.md)** | Node/npm installation, Vite configuration, and frontend development setup |
| **[Testing Strategy](docs/frontend/TESTING.md)** | Component testing with Vitest/RTL and E2E testing with Playwright |

### 🔗 Shared Documentation

Located in `docs/shared/`:

| Document | Description |
|----------|-------------|
| **[API Specifications](docs/shared/API_SPECIFICATIONS.md)** | Complete REST API reference with all endpoints, request/response examples |
| **[Development Plan](docs/shared/DEVELOPMENT_PLAN.md)** | Comprehensive project plan with architecture, phases, and deployment strategy |

## Project Features

### Core Capabilities

- **Advanced Chore Scheduling**: RFC 5545 RRULE-based recurrence patterns with custom intervals
- **Multi-Household Support**: Complete data isolation for children in divorced/separated families
- **Age-Adaptive UI**: Interface automatically adapts based on child's age (Young 5-9, Tween 10-13, Teen 14-17)
- **Financial Transaction System**: ACID-compliant transactions with balance tracking and audit trails
- **Automated Allowances**: Scheduled payments with split distribution across multiple accounts
- **Progressive Web App**: Installable, offline-capable mobile experience
- **OAuth 2.0 Authentication**: Google, Apple, and Facebook social login support

### Technology Stack

**Backend**
- Python 3.11+ with FastAPI
- PostgreSQL 15+ with SQLAlchemy 2.0+
- APScheduler for background jobs
- JWT authentication
- Docker containerization

**Frontend**
- React 18+ with Vite 5+
- Tailwind CSS 3+ with Headless UI
- React Router v6
- React Context + Hooks for state
- PWA with offline support

## Getting Started

### For Backend Developers

1. Review [Database Schema](docs/backend/DATABASE_SCHEMA.md) to understand data models
2. Follow [Backend Setup Guide](docs/backend/LOCAL_DEVELOPMENT_SETUP.md) to install PostgreSQL and Python dependencies
3. Check [API Specifications](docs/shared/API_SPECIFICATIONS.md) for endpoint details
4. Read [Testing Strategy](docs/backend/TESTING.md) for writing tests
5. Clone [chore-app-backend](https://github.com/threnjen/chore-app-backend) and start coding

### For Frontend Developers

1. Review [Component Architecture](docs/frontend/COMPONENT_ARCHITECTURE.md) for component organization
2. Follow [Frontend Setup Guide](docs/frontend/LOCAL_DEVELOPMENT_SETUP.md) to install Node and dependencies
3. Read [Age-Adaptive UI Guide](docs/frontend/AGE_ADAPTIVE_UI_GUIDE.md) for UI implementation patterns
4. Check [State Management](docs/frontend/STATE_MANAGEMENT.md) for Context and hooks usage
5. Review [API Specifications](docs/shared/API_SPECIFICATIONS.md) for backend integration
6. Clone [chore-app-frontend](https://github.com/threnjen/chore-app-frontend) and start building

### For Project Managers / Product

1. Start with [Development Plan](docs/shared/DEVELOPMENT_PLAN.md) for full project overview
2. Review architecture diagrams and phase breakdown
3. Understand multi-household architecture and age-adaptive UI system
4. Check deployment strategy and cost management sections

## Architecture Highlights

### Multi-Tenant Architecture
- Family-based multi-tenancy with complete data isolation
- Middleware-based family scoping for API requests
- PostgreSQL Row-Level Security as safety net
- Support for children belonging to multiple families

### Chore Scheduling Engine
- RFC 5545 iCalendar RRULE-based recurrence patterns
- Custom patterns: M-W-F, bi-weekly, monthly, etc.
- Absolute vs relative timing modes
- Single-instance overdue handling (no backlog accumulation)

### Age-Adaptive UI System
- **Young (5-9)**: Large icons, emoji navigation, simplified interface
- **Tween (10-13)**: Balanced text/icons, educational focus, moderate features
- **Teen (14-17)**: Full feature set, compact layout, advanced financial tools

### Financial Transaction System
- ACID-compliant transactions with automatic balance updates
- Multiple account types: SPENDING, SAVING, GIVING, INVESTMENT, LOAN
- Automated allowance processing with split distributions
- Complete audit trail with balance snapshots

## Development Phases

1. **Phase 1** - Core Infrastructure (3 weeks)
2. **Phase 2** - Chore Management (2 weeks)
3. **Phase 3** - Financial Management (2 weeks)
4. **Phase 4** - Multi-Household Support (2 weeks)
5. **Phase 5** - Advanced Features (3 weeks)
6. **Phase 6** - Notifications & Polish (2 weeks)
7. **Phase 7** - Testing & Documentation (2 weeks)
8. **Phase 8** - Production Deployment (2 weeks)
9. **Phase 9** - Monitoring & Optimization (2 weeks)

**Total Timeline**: ~20 weeks to production-ready

## Deployment Architecture

### POC (Proof of Concept)
- AWS Lightsail Containers ($10/mo micro instance)
- Lightsail Managed Database (PostgreSQL)
- Lightsail CDN + S3 for static assets
- **Cost**: $25-30/month

### Production Scale
- AWS ECS Fargate with auto-scaling
- AWS RDS Multi-AZ PostgreSQL
- CloudFront CDN + S3
- SES for email, CloudWatch monitoring
- **Cost**: $300-800/month (scales with usage)

## Business Model

**Freemium Approach**:
- **Free Tier**: Single family, basic chores, manual allowances, 2 children
- **Premium ($5/month)**: Multi-household, advanced scheduling, automated allowances, unlimited children, savings goals, budgets, priority support

## Contributing

This project follows the coding guidelines in [chore-app-backend/AGENTS.md](https://github.com/threnjen/chore-app-backend/blob/main/AGENTS.md):
- Use public and trusted libraries
- Prefer caching strategies with `lru_cache`
- Prioritize object-oriented programming
- Always use type hints and docstrings
- Use Pydantic for data modeling

## Repository Links

| Repository | Description | Link |
|------------|-------------|------|
| **chore-app-backend** | FastAPI backend service | [View Repo](https://github.com/threnjen/chore-app-backend) |
| **chore-app-frontend** | React PWA frontend | [View Repo](https://github.com/threnjen/chore-app-frontend) |
| **chore-app-docs** | Documentation (this repo) | [View Repo](https://github.com/threnjen/chore-app-docs) |

## License

TBD

## Contact

For questions or contributions, please open an issue in the relevant repository.

---

**Project Status**: Documentation and planning phase. Implementation to begin with Phase 1 (Core Infrastructure).

**Last Updated**: January 29, 2026