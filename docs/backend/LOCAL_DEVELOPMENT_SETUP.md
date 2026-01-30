# Backend Local Development Setup

## Prerequisites
- macOS (or Linux/Windows with adaptations)
- Homebrew (macOS package manager)
- Git
- Python 3.11+
- PostgreSQL 15+

## PostgreSQL Installation (Native)

> **Note**: We use native PostgreSQL installation, not Docker, for better performance and developer experience.

```bash
# Install PostgreSQL via Homebrew
brew install postgresql@15

# Start PostgreSQL service
brew services start postgresql@15

# Create database
createdb picklesapp_dev

# Create user (optional, or use default user)
createuser -s picklesapp
psql -d postgres -c "ALTER USER picklesapp WITH PASSWORD 'dev_password';"

# Verify connection
psql -U picklesapp -d picklesapp_dev -c "SELECT version();"

# Install pgAdmin (optional GUI)
brew install --cask pgadmin4
```

### Database Extensions

```sql
-- Connect to database
psql -U picklesapp -d picklesapp_dev

-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS "pgcrypto";    -- UUID generation
CREATE EXTENSION IF NOT EXISTS "pg_trgm";     -- Fuzzy search
```

## Backend Setup

```bash
# Navigate to backend directory
cd chore-app-backend

# Create virtual environment
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Upgrade pip
pip install --upgrade pip

# Install dependencies
pip install -r requirements.txt

# Copy environment template
cp .env.example .env

# Edit .env with your settings
```

### Environment Variables

Edit `.env` with the following required variables:

```bash
# Database
DATABASE_URL=postgresql://picklesapp:dev_password@localhost:5432/picklesapp_dev

# Security
SECRET_KEY=<generate-strong-key-min-32-chars>
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=30

# CORS (for local frontend)
CORS_ORIGINS=http://localhost:5173,http://127.0.0.1:5173

# OAuth (Development - replace with real credentials)
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
APPLE_CLIENT_ID=your_apple_client_id
APPLE_CLIENT_SECRET=your_apple_client_secret
FACEBOOK_CLIENT_ID=your_facebook_client_id
FACEBOOK_CLIENT_SECRET=your_facebook_client_secret

# AWS (Development - use localstack or leave blank)
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_access_key
AWS_SECRET_ACCESS_KEY=your_secret_key
S3_BUCKET_NAME=picklesapp-dev
SES_SENDER_EMAIL=noreply@picklesapp.com

# Scheduler
SCHEDULER_ENABLED=true
```

### Generate Secret Key

```bash
# Generate a secure secret key
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

## Database Migrations

```bash
# Run all migrations to create schema
alembic upgrade head

# Create a new migration (after model changes)
alembic revision --autogenerate -m "Description of changes"

# Rollback last migration
alembic downgrade -1

# View migration history
alembic history

# View current version
alembic current
```

## Seed Test Data

```bash
# Seed test families and users
python scripts/seed_data.py

# This creates 4 test families with various scenarios:
# 1. Single parent (1 child)
# 2. Two-parent household (2 children)
# 3. Divorced parents (shared child)
# 4. Blended family (multiple households)
```

## Run Development Server

```bash
# Activate virtual environment (if not already active)
source venv/bin/activate

# Run server with hot reload
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Server will start at http://localhost:8000
# Interactive API docs at http://localhost:8000/docs
# Alternative docs at http://localhost:8000/redoc
```

### Background Jobs

The development server automatically starts background jobs if `SCHEDULER_ENABLED=true`:
- Chore instance generation (hourly)
- Overdue chore tracking (daily midnight)
- Allowance processing (varies by schedule)
- Interest calculations (varies by frequency)

To disable background jobs during development:
```bash
SCHEDULER_ENABLED=false uvicorn app.main:app --reload
```

## Docker Setup (Optional)

```bash
# From backend directory
docker build -t picklesapp-backend .

# Run container
docker run -d \
  --name picklesapp-backend \
  -p 8000:8000 \
  --env-file .env \
  picklesapp-backend

# View logs
docker logs -f picklesapp-backend

# Stop container
docker stop picklesapp-backend
docker rm picklesapp-backend
```

## Testing

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=app --cov-report=html

# Run specific test file
pytest tests/test_auth.py

# Run tests matching pattern
pytest -k "test_multi_household"

# View coverage report
open htmlcov/index.html
```

## Common Development Tasks

### Reset Database

```bash
# Drop and recreate database
dropdb picklesapp_dev
createdb picklesapp_dev

# Run migrations
alembic upgrade head

# Reseed data
python scripts/seed_data.py
```

### Update Dependencies

```bash
# Update all packages
pip install --upgrade -r requirements.txt

# Update specific package
pip install --upgrade fastapi

# Freeze updated dependencies
pip freeze > requirements.txt
```

### Linting and Formatting

```bash
# Format code with black
black .

# Lint with flake8
flake8 .

# Type checking with mypy
mypy .
```

## Troubleshooting

### PostgreSQL Connection Issues

```bash
# Check if PostgreSQL is running
brew services list

# Restart PostgreSQL
brew services restart postgresql@15

# Check PostgreSQL logs
tail -f /usr/local/var/log/postgresql@15.log
```

### Migration Issues

```bash
# If migrations fail, check current state
alembic current

# Stamp database at specific revision
alembic stamp head

# Generate migration manually
alembic revision -m "manual migration"
# Edit the generated file in alembic/versions/
```

### Import Errors

```bash
# Ensure virtual environment is activated
which python  # Should point to venv/bin/python

# Reinstall dependencies
pip install -r requirements.txt

# Clear Python cache
find . -type d -name "__pycache__" -exec rm -rf {} +
find . -type f -name "*.pyc" -delete
```

## Port Configuration

- **Backend API**: `8000`
- **PostgreSQL**: `5432`
- **pgAdmin**: `5050` (if using Docker)

Ensure these ports are not in use by other applications.

## Next Steps

- Review [Database Schema](DATABASE_SCHEMA.md) documentation
- Explore [API Specifications](../shared/API_SPECIFICATIONS.md)
- Read [Testing Strategy](TESTING.md)
- Check [Development Plan](../shared/DEVELOPMENT_PLAN.md) for feature roadmap
