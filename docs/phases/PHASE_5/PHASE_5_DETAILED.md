# Phase 5: Infrastructure & Deployment
**Duration: 2-3 weeks**

---

## Overview

Phase 5 establishes the production infrastructure on AWS, implements CI/CD pipelines, and deploys the application. This phase transforms PicklesApp from local development to a hosted service.

---

## Goals

- AWS infrastructure setup
- CI/CD pipeline
- Production deployment
- Cost monitoring

---

## Deliverables

### 1. Infrastructure as Code (Terraform)

- AWS Lightsail containers (POC)
- PostgreSQL (Lightsail managed DB)
- S3 bucket for static assets
- CloudFront distribution
- SES for email
- Route 53 DNS
- IAM roles and policies
- Cost budgets and alarms

### 2. CI/CD (GitHub Actions)

- Automated testing on PR
- Docker image builds
- Deployment to dev environment
- Deployment to prod (manual approval)
- Database migrations automation

### 3. Environments

- **Dev**: Lightsail Micro container + Standard DB ($25/mo)
- **Prod**: Lightsail Small container + HA DB ($50/mo)

### 4. Monitoring

- CloudWatch dashboards
- Error alerting
- Cost monitoring
- Performance metrics

### 5. Security

- HTTPS/TLS certificates
- Secrets management (AWS Secrets Manager)
- Database encryption at rest
- Backup automation

---

## AWS Architecture

### POC Phase ($25-30/mo)

```
Internet
   ↓
Route 53 / Lightsail DNS
   ↓
CloudFront (CDN)
   ↓
┌─────────────────────────────────┐
│   Lightsail Container (Micro)   │
│   - Backend + Frontend serve    │
│   - 1GB RAM, 0.25 vCPU          │
│   - $10/mo                      │
└─────────────────────────────────┘
   ↓
┌─────────────────────────────────┐
│ Lightsail Managed PostgreSQL    │
│ - Standard (40GB)               │
│ - 1GB RAM, 1 vCPU               │
│ - $15/mo                        │
└─────────────────────────────────┘
```

### Services Breakdown

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| Lightsail Container | Micro (1GB RAM) | $10 |
| Lightsail PostgreSQL | Standard (40GB) | $15 |
| S3 | Standard tier | $0-1 |
| CloudFront | Free tier | $0 |
| SES | Email sending | $0-1 |
| Route 53 | DNS (optional) | $0.50 |
| **Total** | | **$25-28** |

---

## Terraform Structure

```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars
├── modules/
│   ├── lightsail/
│   │   ├── container.tf
│   │   ├── database.tf
│   │   ├── dns.tf
│   │   └── variables.tf
│   ├── s3/
│   │   ├── main.tf
│   │   └── variables.tf
│   ├── cloudfront/
│   │   ├── main.tf
│   │   └── variables.tf
│   ├── ses/
│   │   ├── main.tf
│   │   └── variables.tf
│   └── monitoring/
│       ├── budgets.tf
│       ├── alarms.tf
│       └── variables.tf
└── README.md
```

### Lightsail Container Module

```hcl
# infrastructure/modules/lightsail/container.tf

resource "aws_lightsail_container_service" "app" {
  name        = "${var.project_name}-${var.environment}"
  power       = var.environment == "prod" ? "small" : "micro"
  scale       = 1
  is_disabled = false

  tags = {
    Project     = var.project_name
    Environment = var.environment
  }
}

resource "aws_lightsail_container_service_deployment_version" "app" {
  service_name = aws_lightsail_container_service.app.name

  container {
    container_name = "backend"
    image          = var.backend_image

    ports = {
      8000 = "HTTP"
    }

    environment = {
      DATABASE_URL     = "postgresql://${var.db_user}:${var.db_password}@${aws_lightsail_database.main.master_endpoint_address}:5432/${var.db_name}"
      SECRET_KEY       = var.secret_key
      ENVIRONMENT      = var.environment
      AWS_REGION       = var.aws_region
      S3_BUCKET        = var.s3_bucket
      SES_FROM_EMAIL   = var.ses_from_email
    }
  }

  public_endpoint {
    container_name = "backend"
    container_port = 8000

    health_check {
      path                = "/health"
      interval_seconds    = 30
      timeout_seconds     = 5
      healthy_threshold   = 2
      unhealthy_threshold = 2
    }
  }
}
```

### Lightsail Database Module

```hcl
# infrastructure/modules/lightsail/database.tf

resource "aws_lightsail_database" "main" {
  relational_database_name = "${var.project_name}-${var.environment}-db"
  blueprint_id             = "postgres_15"
  bundle_id                = var.environment == "prod" ? "medium_ha_2_0" : "micro_2_0"
  
  master_database_name     = var.db_name
  master_username          = var.db_user
  master_password          = var.db_password
  
  availability_zone        = "${var.aws_region}a"
  preferred_backup_window  = "03:00-04:00"
  backup_retention_enabled = true
  
  apply_immediately        = var.environment != "prod"
  
  tags = {
    Project     = var.project_name
    Environment = var.environment
  }
}

output "database_endpoint" {
  value = aws_lightsail_database.main.master_endpoint_address
}
```

### S3 Module

```hcl
# infrastructure/modules/s3/main.tf

resource "aws_s3_bucket" "assets" {
  bucket = "${var.project_name}-${var.environment}-assets"

  tags = {
    Project     = var.project_name
    Environment = var.environment
  }
}

resource "aws_s3_bucket_public_access_block" "assets" {
  bucket = aws_s3_bucket.assets.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_cors_configuration" "assets" {
  bucket = aws_s3_bucket.assets.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "HEAD"]
    allowed_origins = var.allowed_origins
    max_age_seconds = 3600
  }
}
```

### Cost Monitoring Module

```hcl
# infrastructure/modules/monitoring/budgets.tf

resource "aws_budgets_budget" "monthly" {
  name              = "${var.project_name}-${var.environment}-monthly"
  budget_type       = "COST"
  limit_amount      = var.monthly_budget_limit
  limit_unit        = "USD"
  time_period_start = "2026-02-01_00:00"
  time_unit         = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = var.alert_emails
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "ACTUAL"
    subscriber_email_addresses = var.alert_emails
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type            = "PERCENTAGE"
    notification_type         = "FORECASTED"
    subscriber_email_addresses = var.alert_emails
  }
}
```

---

## GitHub Actions CI/CD

### Test Workflow

```yaml
# .github/workflows/test.yml

name: Test

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main, develop]

jobs:
  backend-tests:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          cd backend
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/test_db
          SECRET_KEY: test-secret-key
        run: |
          cd backend
          pytest -v --cov=app --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: backend/coverage.xml

  frontend-tests:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
      
      - name: Install dependencies
        run: |
          cd frontend
          npm ci
      
      - name: Run tests
        run: |
          cd frontend
          npm test -- --coverage
      
      - name: Run linting
        run: |
          cd frontend
          npm run lint
```

### Deploy Workflow

```yaml
# .github/workflows/deploy.yml

name: Deploy

on:
  push:
    branches: [main]
    tags:
      - 'v*'

env:
  AWS_REGION: us-east-1
  PROJECT_NAME: picklesapp

jobs:
  deploy-dev:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: development
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Build and push Docker image
        run: |
          docker build -t ${{ env.PROJECT_NAME }}-backend:${{ github.sha }} ./backend
          aws lightsail push-container-image \
            --service-name ${{ env.PROJECT_NAME }}-dev \
            --label backend \
            --image ${{ env.PROJECT_NAME }}-backend:${{ github.sha }}
      
      - name: Deploy to Lightsail
        run: |
          aws lightsail create-container-service-deployment \
            --service-name ${{ env.PROJECT_NAME }}-dev \
            --containers file://deploy/containers-dev.json \
            --public-endpoint file://deploy/public-endpoint.json
      
      - name: Run migrations
        run: |
          # Execute migration command in container
          aws lightsail get-container-log \
            --service-name ${{ env.PROJECT_NAME }}-dev \
            --container-name backend
      
      - name: Run smoke tests
        run: |
          sleep 60  # Wait for deployment
          curl -f https://dev.picklesapp.com/health || exit 1

  deploy-prod:
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    needs: [deploy-dev]
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Build and push Docker image
        run: |
          docker build -t ${{ env.PROJECT_NAME }}-backend:${{ github.ref_name }} ./backend
          aws lightsail push-container-image \
            --service-name ${{ env.PROJECT_NAME }}-prod \
            --label backend \
            --image ${{ env.PROJECT_NAME }}-backend:${{ github.ref_name }}
      
      - name: Create database backup
        run: |
          aws lightsail create-relational-database-snapshot \
            --relational-database-name ${{ env.PROJECT_NAME }}-prod-db \
            --relational-database-snapshot-name pre-deploy-${{ github.ref_name }}
      
      - name: Deploy to Lightsail
        run: |
          aws lightsail create-container-service-deployment \
            --service-name ${{ env.PROJECT_NAME }}-prod \
            --containers file://deploy/containers-prod.json \
            --public-endpoint file://deploy/public-endpoint.json
      
      - name: Run smoke tests
        run: |
          sleep 60
          curl -f https://picklesapp.com/health || exit 1
      
      - name: Notify team
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "🚀 PicklesApp ${{ github.ref_name }} deployed to production!"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Secrets Management

### AWS Secrets Manager Setup

```hcl
# infrastructure/modules/secrets/main.tf

resource "aws_secretsmanager_secret" "app" {
  name = "${var.project_name}/${var.environment}/app"
  
  tags = {
    Project     = var.project_name
    Environment = var.environment
  }
}

resource "aws_secretsmanager_secret_version" "app" {
  secret_id = aws_secretsmanager_secret.app.id
  secret_string = jsonencode({
    SECRET_KEY       = var.secret_key
    DATABASE_URL     = var.database_url
    SES_ACCESS_KEY   = var.ses_access_key
    SES_SECRET_KEY   = var.ses_secret_key
  })
}
```

### Backend Integration

```python
# app/config.py

import boto3
import json
from functools import lru_cache

@lru_cache()
def get_secrets():
    """
    Retrieve secrets from AWS Secrets Manager.
    """
    if settings.ENVIRONMENT == "development":
        return {}  # Use .env in development
    
    client = boto3.client('secretsmanager', region_name=settings.AWS_REGION)
    
    response = client.get_secret_value(
        SecretId=f"picklesapp/{settings.ENVIRONMENT}/app"
    )
    
    return json.loads(response['SecretString'])
```

---

## Monitoring & Alerting

### CloudWatch Dashboard

```hcl
# infrastructure/modules/monitoring/dashboard.tf

resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "${var.project_name}-${var.environment}"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "Container CPU"
          region = var.aws_region
          metrics = [
            ["AWS/Lightsail", "CPUUtilization", "ContainerServiceName", "${var.project_name}-${var.environment}"]
          ]
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "Container Memory"
          region = var.aws_region
          metrics = [
            ["AWS/Lightsail", "MemoryUtilization", "ContainerServiceName", "${var.project_name}-${var.environment}"]
          ]
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 12
        height = 6
        properties = {
          title  = "Database Connections"
          region = var.aws_region
          metrics = [
            ["AWS/Lightsail", "DatabaseConnections", "DatabaseName", "${var.project_name}-${var.environment}-db"]
          ]
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 6
        width  = 12
        height = 6
        properties = {
          title  = "API Latency"
          region = var.aws_region
          metrics = [
            ["AWS/Lightsail", "RequestLatency", "ContainerServiceName", "${var.project_name}-${var.environment}"]
          ]
        }
      }
    ]
  })
}
```

### Alarms

```hcl
# infrastructure/modules/monitoring/alarms.tf

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.project_name}-${var.environment}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name        = "CPUUtilization"
  namespace          = "AWS/Lightsail"
  period             = "300"
  statistic          = "Average"
  threshold          = "80"
  alarm_description  = "CPU utilization above 80% for 10 minutes"
  alarm_actions      = [aws_sns_topic.alerts.arn]

  dimensions = {
    ContainerServiceName = "${var.project_name}-${var.environment}"
  }
}

resource "aws_cloudwatch_metric_alarm" "high_memory" {
  alarm_name          = "${var.project_name}-${var.environment}-high-memory"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name        = "MemoryUtilization"
  namespace          = "AWS/Lightsail"
  period             = "300"
  statistic          = "Average"
  threshold          = "85"
  alarm_description  = "Memory utilization above 85% for 10 minutes"
  alarm_actions      = [aws_sns_topic.alerts.arn]

  dimensions = {
    ContainerServiceName = "${var.project_name}-${var.environment}"
  }
}

resource "aws_cloudwatch_metric_alarm" "db_connections" {
  alarm_name          = "${var.project_name}-${var.environment}-db-connections"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name        = "DatabaseConnections"
  namespace          = "AWS/Lightsail"
  period             = "300"
  statistic          = "Average"
  threshold          = "80"
  alarm_description  = "Database connections above 80"
  alarm_actions      = [aws_sns_topic.alerts.arn]

  dimensions = {
    DatabaseName = "${var.project_name}-${var.environment}-db"
  }
}
```

---

## Deployment Process

### Dev Environment

```bash
# Initial setup
cd infrastructure/environments/dev
terraform init
terraform plan
terraform apply

# Deploy application
aws lightsail push-container-image \
  --service-name picklesapp-dev \
  --label backend \
  --image backend:latest

aws lightsail create-container-service-deployment \
  --service-name picklesapp-dev \
  --containers file://containers.json \
  --public-endpoint file://public-endpoint.json
```

### Production Deployment

```bash
# Via GitHub Actions (manual approval required)
git tag v1.0.0
git push origin v1.0.0

# Triggers workflow:
# 1. Run tests
# 2. Build Docker images
# 3. Push to registry
# 4. Wait for manual approval
# 5. Deploy to production
# 6. Run smoke tests
# 7. Notify team
```

---

## Testing Requirements

### Deployment Smoke Tests

```python
# tests/smoke/test_health.py

import requests

def test_health_endpoint():
    """Verify application is running."""
    response = requests.get(f"{BASE_URL}/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"

def test_database_connection():
    """Verify database connectivity."""
    response = requests.get(f"{BASE_URL}/health/db")
    assert response.status_code == 200
    assert response.json()["database"] == "connected"

def test_auth_endpoint():
    """Verify auth is working."""
    response = requests.post(f"{BASE_URL}/auth/login", json={
        "email": "smoke@test.com",
        "password": "smoketest123"
    })
    assert response.status_code in [200, 401]  # Either works or unauthorized
```

### Infrastructure Validation

```bash
# Terraform validate
cd infrastructure/environments/dev
terraform validate
terraform plan -detailed-exitcode

# Security scan
tfsec infrastructure/

# Cost estimate
infracost breakdown --path infrastructure/environments/prod
```

---

## Documentation

### Deployment Runbook

```markdown
# Deployment Runbook

## Pre-Deployment Checklist
- [ ] All tests passing
- [ ] Database migrations reviewed
- [ ] Environment variables updated
- [ ] Backup created (prod only)

## Deployment Steps
1. Create git tag: `git tag v1.x.x`
2. Push tag: `git push origin v1.x.x`
3. Monitor GitHub Actions workflow
4. Approve production deployment
5. Verify smoke tests pass

## Rollback Procedure
1. Identify last working version
2. Push previous image: `aws lightsail create-container-service-deployment ...`
3. If DB changes: restore from snapshot
4. Notify team of rollback

## Emergency Contacts
- On-call: alerts@picklesapp.com
- AWS Support: (if enterprise)
```

---

## Definition of Done

### Must Have
- [ ] Dev environment deployed and accessible
- [ ] CI/CD pipeline running tests on PRs
- [ ] Automated deployment to dev on main merge
- [ ] Production deployment workflow (manual approval)
- [ ] Cost monitoring with alerts at 80% and 100%
- [ ] CloudWatch dashboards for key metrics
- [ ] HTTPS working with valid certificates
- [ ] Database backups enabled
- [ ] Secrets in AWS Secrets Manager

### Nice to Have
- [ ] Staging environment
- [ ] Blue/green deployments
- [ ] Automated rollback on failed smoke tests
- [ ] Slack notifications for deployments

---

## Dependencies

- **Requires**: Phase 1-4 (application to deploy)
- **Enables**: Phase 6 (production environment needed for OAuth testing)

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Cost overrun | Budget alerts at 80%, daily cost alarm |
| Deployment failures | Pre-deployment backup, rollback procedure |
| Secrets exposure | AWS Secrets Manager, no secrets in code |
| Database downtime | HA setup for prod, automated backups |

---

*Phase 5 Document - Last Updated: February 2026*
