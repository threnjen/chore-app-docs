# Local Development Setup Guide

> Complete setup instructions for **macOS** and **Windows (WSL)** developers

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [macOS Setup](#macos-setup)
3. [Windows (WSL) Setup](#windows-wsl-setup)
4. [Backend Setup](#backend-setup)
5. [Frontend Setup](#frontend-setup)
6. [Running the Application](#running-the-application)
7. [Seed Data](#seed-data)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Software

| Software | Version | Purpose |
|----------|---------|---------|
| Docker Desktop | Latest | PostgreSQL + Container runtime |
| Git | 2.x+ | Version control |
| Node.js | 20.x LTS | Frontend development |
| Python | 3.11+ | Backend development (optional for Docker-only) |

### Architecture Decision

**PostgreSQL runs in Docker** for all developers to ensure:
- Cross-platform consistency (macOS, Windows, Linux)
- No native installation conflicts
- Easy cleanup and reset
- Matches production environment

---

## macOS Setup

### 1. Install Homebrew (if not installed)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### 2. Install Docker Desktop

```bash
brew install --cask docker

# Launch Docker Desktop
open -a Docker
```

Verify Docker is running:
```bash
docker --version
docker-compose --version
```

### 3. Install Node.js

```bash
brew install node@20

# Add to PATH (if needed)
echo 'export PATH="/opt/homebrew/opt/node@20/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Verify
node --version  # Should be 20.x
npm --version
```

### 4. Install Python (optional, for local development without Docker)

```bash
brew install python@3.11

# Verify
python3 --version  # Should be 3.11.x
```

### 5. Clone Repositories

```bash
mkdir -p ~/github_repos && cd ~/github_repos

git clone https://github.com/your-org/chore-app-backend.git
git clone https://github.com/your-org/chore-app-frontend.git
git clone https://github.com/your-org/chore-app-docs.git
```

---

## Windows (WSL) Setup

> **Important**: Windows developers must use WSL 2 (Windows Subsystem for Linux) for the best development experience.

### 1. Install WSL 2

Open PowerShell as Administrator:

```powershell
# Install WSL with Ubuntu
wsl --install -d Ubuntu

# Restart your computer when prompted
```

After restart, open Ubuntu from the Start menu and complete initial setup (username/password).

### 2. Install Docker Desktop for Windows

1. Download from: https://www.docker.com/products/docker-desktop/
2. Run installer and enable "Use WSL 2 based engine"
3. In Docker Desktop Settings:
   - Go to **Resources** → **WSL Integration**
   - Enable integration with your Ubuntu distro
   - Click **Apply & Restart**

Verify in WSL terminal:
```bash
docker --version
docker-compose --version
```

### 3. Install Node.js in WSL

```bash
# Install nvm (Node Version Manager)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash

# Reload shell
source ~/.bashrc

# Install Node.js
nvm install 20
nvm use 20

# Verify
node --version
npm --version
```

### 4. Install Python in WSL (optional)

```bash
sudo apt update
sudo apt install python3.11 python3.11-venv python3-pip

# Verify
python3.11 --version
```

### 5. Clone Repositories in WSL

```bash
mkdir -p ~/github_repos && cd ~/github_repos

git clone https://github.com/your-org/chore-app-backend.git
git clone https://github.com/your-org/chore-app-frontend.git
git clone https://github.com/your-org/chore-app-docs.git
```

> **Tip**: Store code in the WSL filesystem (`~/github_repos`) not Windows (`/mnt/c/...`) for much better performance.

### 6. VS Code Remote Development

1. Install VS Code on Windows
2. Install the "Remote - WSL" extension
3. In WSL terminal, navigate to project and run: `code .`

This opens VS Code connected to WSL with full Linux tooling support.

---

## Backend Setup

### 1. Navigate to Backend Directory

```bash
cd ~/github_repos/chore-app-backend
```

### 2. Create Environment File

```bash
cp .env.example .env
```

Edit `.env` with your editor:

```bash
# Database (Docker PostgreSQL)
DATABASE_URL=postgresql+asyncpg://picklesapp:dev_password@localhost:5432/picklesapp

# Security (generate a strong key)
SECRET_KEY=your-super-secret-key-at-least-32-characters
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60
REFRESH_TOKEN_EXPIRE_DAYS=30

# CORS
CORS_ORIGINS=http://localhost:3000,http://localhost:5173

# OAuth (stub values for Phase 1)
GOOGLE_CLIENT_ID=not-configured
GOOGLE_CLIENT_SECRET=not-configured
APPLE_CLIENT_ID=not-configured
APPLE_CLIENT_SECRET=not-configured
FACEBOOK_CLIENT_ID=not-configured
FACEBOOK_CLIENT_SECRET=not-configured
```

### 3. Start Backend with Docker Compose

```bash
# Start PostgreSQL and API containers
docker-compose up -d

# View logs
docker-compose logs -f

# Check containers are running
docker-compose ps
```

This starts:
- **PostgreSQL 15** on port 5432
- **FastAPI** on port 8000

### 4. Run Database Migrations

```bash
# Run Alembic migrations inside the API container
docker-compose exec api alembic upgrade head
```

### 5. Verify API is Running

```bash
# Health check
curl http://localhost:8000/health

# Should return: {"status":"healthy"}

# API docs
open http://localhost:8000/docs  # macOS
# or visit in browser: http://localhost:8000/docs
```

---

## Frontend Setup

### 1. Navigate to Frontend Directory

```bash
cd ~/github_repos/chore-app-frontend
```

### 2. Install Dependencies

```bash
npm install
```

### 3. Create Environment File

```bash
cp .env.example .env
```

Edit `.env`:

```bash
# API Configuration (points to local backend)
VITE_API_BASE_URL=http://localhost:8000

# OAuth (not functional in Phase 1)
VITE_GOOGLE_CLIENT_ID=not-configured
VITE_APPLE_CLIENT_ID=not-configured
VITE_FACEBOOK_CLIENT_ID=not-configured

# Feature Flags
VITE_ENABLE_PWA=true
VITE_ENABLE_OFFLINE_MODE=true
VITE_ENABLE_PUSH_NOTIFICATIONS=false

# Environment
VITE_ENV=development
```

### 4. Start Development Server

```bash
npm run dev
```

Frontend runs at: http://localhost:3000

---

## Running the Application

### Quick Start (Both Services)

**Terminal 1 - Backend:**
```bash
cd ~/github_repos/chore-app-backend
docker-compose up
```

**Terminal 2 - Frontend:**
```bash
cd ~/github_repos/chore-app-frontend
npm run dev
```

### Stopping Services

```bash
# Stop backend
cd ~/github_repos/chore-app-backend
docker-compose down

# Stop and remove volumes (⚠️ deletes database data)
docker-compose down -v
```

### Viewing Logs

```bash
# All logs
docker-compose logs -f

# Only API logs
docker-compose logs -f api

# Only database logs
docker-compose logs -f db
```

---

## Seed Data

Load test data for development:

```bash
cd ~/github_repos/chore-app-backend

# Run seed script inside container
docker-compose exec api python scripts/seed_dev_data.py
```

### Seed Data Contents

| User | Email | Password | Role | Family |
|------|-------|----------|------|--------|
| Parent 1 | parent1@test.com | TestPass123 | PARENT | Family A (Owner) |
| Parent 2 | parent2@test.com | TestPass123 | PARENT | Family B (Owner), Family A (Member) |
| Child 1 | child1@test.com | TestPass123 | CHILD | Family A (age 8) |
| Child 2 | child2@test.com | TestPass123 | CHILD | Family A (age 12) |
| Child 3 | child3@test.com | TestPass123 | CHILD | Family B, Family A (age 15) |

Each child has SPEND, SAVE, and GIVE accounts with sample transactions.

---

## Troubleshooting

### Docker Issues

**Problem**: `docker-compose: command not found`
```bash
# Modern Docker uses `docker compose` (no hyphen)
docker compose up -d
```

**Problem**: Port 5432 already in use
```bash
# Find what's using the port
lsof -i :5432

# Kill the process or change port in docker-compose.yml
```

**Problem**: Permission denied on Docker socket
```bash
# Add user to docker group (Linux/WSL)
sudo usermod -aG docker $USER
newgrp docker
```

### Database Issues

**Problem**: Migration fails
```bash
# Reset database (⚠️ deletes all data)
docker-compose down -v
docker-compose up -d
docker-compose exec api alembic upgrade head
```

**Problem**: Can't connect to database
```bash
# Check if PostgreSQL container is running
docker-compose ps

# Check database logs
docker-compose logs db
```

### Frontend Issues

**Problem**: CORS errors
```bash
# Ensure backend .env has correct CORS_ORIGINS
CORS_ORIGINS=http://localhost:3000,http://localhost:5173

# Restart backend
docker-compose restart api
```

**Problem**: API calls return 404
```bash
# Check VITE_API_BASE_URL in frontend .env
# Should be: http://localhost:8000 (no /api/v1 suffix)
```

### WSL-Specific Issues

**Problem**: Docker not accessible in WSL
1. Open Docker Desktop
2. Go to Settings → Resources → WSL Integration
3. Ensure your Ubuntu distro is enabled
4. Click Apply & Restart

**Problem**: Slow filesystem performance
```bash
# Move project to WSL filesystem
mv /mnt/c/Users/you/projects/chore-app-backend ~/github_repos/
```

**Problem**: Line ending issues
```bash
# Configure Git for LF line endings
git config --global core.autocrlf input
```

---

## Next Steps

1. ✅ Backend running on http://localhost:8000
2. ✅ Frontend running on http://localhost:3000
3. ✅ Database migrations applied
4. ⬜ Load seed data (optional)
5. ⬜ Register a test user
6. ⬜ Create a family
7. ⬜ Start developing!

For testing procedures, see [PHASE_1_TESTING.md](../phases/PHASE_1_TESTING.md).
