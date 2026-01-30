## Local Development Setup

### Prerequisites
- macOS (or Linux/Windows with adaptations)
- Homebrew (macOS package manager)
- Git
- Docker Desktop
- Node.js 20+
- Python 3.11+

### PostgreSQL Installation (Native)

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

### Backend Setup

```bash
# Navigate to backend directory
cd backend

# Create virtual environment
python3 -m venv venv

# Activate virtual environment
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Copy environment template
cp .env.example .env

# Edit .env with your settings
# DATABASE_URL=postgresql://picklesapp:dev_password@localhost:5432/picklesapp_dev
# SECRET_KEY=<generate strong key>
# etc.

# Run database migrations
alembic upgrade head

# Seed test data (optional)
python scripts/seed_data.py

# Run development server (with hot reload)
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Backend available at http://localhost:8000
# API docs at http://localhost:8000/docs
```

### Frontend Setup

```bash
# Navigate to frontend directory
cd frontend

# Install dependencies
npm install

# Copy environment template
cp .env.example .env

# Edit .env
# VITE_API_URL=http://localhost:8000/api/v1

# Run development server (with hot reload)
npm run dev

# Frontend available at http://localhost:5173
```

### Docker Setup (Alternative to Native)

```bash
# From project root
docker-compose up -d

# Services:
# - Backend: http://localhost:8000
# - Frontend: http://localhost:5173
# - PostgreSQL: localhost:5432
# - pgAdmin: http://localhost:5050

# View logs
docker-compose logs -f backend
docker-compose logs -f frontend

# Stop services
docker-compose down

# Rebuild after code changes
docker-compose up -d --build
```