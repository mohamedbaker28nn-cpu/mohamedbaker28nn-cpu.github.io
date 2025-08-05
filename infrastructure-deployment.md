# Infrastructure as Code & Deployment Guide

## Terraform Infrastructure Configuration

### Root Module (main.tf)
```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
  
  backend "s3" {
    bucket = "course-platform-terraform-state"
    key    = "infrastructure/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Project     = "course-platform"
      Environment = var.environment
      ManagedBy   = "terraform"
    }
  }
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}

# Local variables
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_caller_identity" "current" {}

# Module calls
module "vpc" {
  source = "./modules/vpc"
  
  name_prefix         = local.name_prefix
  cidr_block         = var.vpc_cidr
  availability_zones = data.aws_availability_zones.available.names
  
  tags = local.common_tags
}

module "security" {
  source = "./modules/security"
  
  name_prefix = local.name_prefix
  vpc_id      = module.vpc.vpc_id
  
  tags = local.common_tags
}

module "database" {
  source = "./modules/database"
  
  name_prefix       = local.name_prefix
  vpc_id           = module.vpc.vpc_id
  subnet_ids       = module.vpc.private_subnet_ids
  security_group_id = module.security.database_security_group_id
  
  tags = local.common_tags
}

module "cache" {
  source = "./modules/cache"
  
  name_prefix       = local.name_prefix
  vpc_id           = module.vpc.vpc_id
  subnet_ids       = module.vpc.private_subnet_ids
  security_group_id = module.security.cache_security_group_id
  
  tags = local.common_tags
}

module "ecs" {
  source = "./modules/ecs"
  
  name_prefix            = local.name_prefix
  vpc_id                = module.vpc.vpc_id
  public_subnet_ids     = module.vpc.public_subnet_ids
  private_subnet_ids    = module.vpc.private_subnet_ids
  security_group_id     = module.security.ecs_security_group_id
  database_url          = module.database.connection_string
  redis_url             = module.cache.redis_url
  
  tags = local.common_tags
}

module "storage" {
  source = "./modules/storage"
  
  name_prefix = local.name_prefix
  
  tags = local.common_tags
}

module "video_processing" {
  source = "./modules/video-processing"
  
  name_prefix              = local.name_prefix
  vpc_id                  = module.vpc.vpc_id
  private_subnet_ids      = module.vpc.private_subnet_ids
  temp_storage_bucket     = module.storage.temp_bucket_name
  database_url            = module.database.connection_string
  cloudflare_r2_endpoint  = var.cloudflare_r2_endpoint
  cloudflare_r2_bucket    = var.cloudflare_r2_bucket
  
  tags = local.common_tags
}

module "monitoring" {
  source = "./modules/monitoring"
  
  name_prefix = local.name_prefix
  
  tags = local.common_tags
}
```

### Variables (variables.tf)
```hcl
variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "course-platform"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "cloudflare_api_token" {
  description = "Cloudflare API token"
  type        = string
  sensitive   = true
}

variable "cloudflare_r2_endpoint" {
  description = "Cloudflare R2 endpoint"
  type        = string
}

variable "cloudflare_r2_bucket" {
  description = "Cloudflare R2 bucket name"
  type        = string
}

variable "domain_name" {
  description = "Primary domain name"
  type        = string
}

variable "database_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}
```

### VPC Module (modules/vpc/main.tf)
```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-vpc"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-igw"
  })
}

resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-public-subnet-${count.index + 1}"
    Type = "public"
  })
}

resource "aws_subnet" "private" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-private-subnet-${count.index + 1}"
    Type = "private"
  })
}

resource "aws_eip" "nat" {
  count = length(aws_subnet.public)
  
  domain = "vpc"
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-nat-eip-${count.index + 1}"
  })
  
  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = length(aws_subnet.public)
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-nat-gateway-${count.index + 1}"
  })
  
  depends_on = [aws_internet_gateway.main]
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-public-rt"
  })
}

resource "aws_route_table" "private" {
  count = length(aws_nat_gateway.main)
  
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-private-rt-${count.index + 1}"
  })
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

### ECS Module (modules/ecs/main.tf)
```hcl
# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "${var.name_prefix}-cluster"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
  
  tags = var.tags
}

resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name
  
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
  
  default_capacity_provider_strategy {
    base              = 1
    weight            = 100
    capacity_provider = "FARGATE"
  }
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "${var.name_prefix}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [var.security_group_id]
  subnets            = var.public_subnet_ids
  
  enable_deletion_protection = var.environment == "production"
  
  tags = var.tags
}

resource "aws_lb_target_group" "frontend" {
  name     = "${var.name_prefix}-frontend-tg"
  port     = 3000
  protocol = "HTTP"
  vpc_id   = var.vpc_id
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
  
  tags = var.tags
}

resource "aws_lb_target_group" "backend" {
  name     = "${var.name_prefix}-backend-tg"
  port     = 4000
  protocol = "HTTP"
  vpc_id   = var.vpc_id
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/api/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
  
  tags = var.tags
}

resource "aws_lb_listener" "main" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.frontend.arn
  }
}

resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.main.arn
  priority     = 100
  
  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.backend.arn
  }
  
  condition {
    path_pattern {
      values = ["/api/*"]
    }
  }
}

# Task Definitions
resource "aws_ecs_task_definition" "frontend" {
  family                   = "${var.name_prefix}-frontend"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 512
  memory                   = 1024
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn           = aws_iam_role.ecs_task.arn
  
  container_definitions = jsonencode([
    {
      name  = "frontend"
      image = "${var.ecr_repository_url}:frontend-latest"
      
      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]
      
      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment
        },
        {
          name  = "NEXT_PUBLIC_API_URL"
          value = "https://${var.domain_name}/api"
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.frontend.name
          "awslogs-region"        = data.aws_region.current.name
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
  
  tags = var.tags
}

resource "aws_ecs_task_definition" "backend" {
  family                   = "${var.name_prefix}-backend"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = 1024
  memory                   = 2048
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn           = aws_iam_role.ecs_task.arn
  
  container_definitions = jsonencode([
    {
      name  = "backend"
      image = "${var.ecr_repository_url}:backend-latest"
      
      portMappings = [
        {
          containerPort = 4000
          protocol      = "tcp"
        }
      ]
      
      environment = [
        {
          name  = "NODE_ENV"
          value = var.environment
        },
        {
          name  = "DATABASE_URL"
          value = var.database_url
        },
        {
          name  = "REDIS_URL"
          value = var.redis_url
        }
      ]
      
      secrets = [
        {
          name      = "JWT_SECRET"
          valueFrom = aws_secretsmanager_secret.jwt.arn
        }
      ]
      
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.backend.name
          "awslogs-region"        = data.aws_region.current.name
          "awslogs-stream-prefix" = "ecs"
        }
      }
    }
  ])
  
  tags = var.tags
}

# ECS Services
resource "aws_ecs_service" "frontend" {
  name            = "${var.name_prefix}-frontend"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.frontend.arn
  desired_count   = var.environment == "production" ? 2 : 1
  
  network_configuration {
    security_groups  = [var.security_group_id]
    subnets          = var.private_subnet_ids
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.frontend.arn
    container_name   = "frontend"
    container_port   = 3000
  }
  
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight           = 100
  }
  
  depends_on = [aws_lb_listener.main]
  
  tags = var.tags
}

resource "aws_ecs_service" "backend" {
  name            = "${var.name_prefix}-backend"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.backend.arn
  desired_count   = var.environment == "production" ? 2 : 1
  
  network_configuration {
    security_groups  = [var.security_group_id]
    subnets          = var.private_subnet_ids
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = aws_lb_target_group.backend.arn
    container_name   = "backend"
    container_port   = 4000
  }
  
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight           = 100
  }
  
  depends_on = [aws_lb_listener_rule.api]
  
  tags = var.tags
}

# Auto Scaling
resource "aws_appautoscaling_target" "frontend" {
  max_capacity       = 10
  min_capacity       = 1
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.frontend.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "frontend_cpu" {
  name               = "${var.name_prefix}-frontend-cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.frontend.resource_id
  scalable_dimension = aws_appautoscaling_target.frontend.scalable_dimension
  service_namespace  = aws_appautoscaling_target.frontend.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}

# CloudWatch Log Groups
resource "aws_cloudwatch_log_group" "frontend" {
  name              = "/ecs/${var.name_prefix}-frontend"
  retention_in_days = 30
  
  tags = var.tags
}

resource "aws_cloudwatch_log_group" "backend" {
  name              = "/ecs/${var.name_prefix}-backend"
  retention_in_days = 30
  
  tags = var.tags
}
```

### Database Module (modules/database/main.tf)
```hcl
resource "aws_db_subnet_group" "main" {
  name       = "${var.name_prefix}-db-subnet-group"
  subnet_ids = var.subnet_ids
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-db-subnet-group"
  })
}

resource "aws_rds_cluster_parameter_group" "main" {
  family = "aurora-postgresql15"
  name   = "${var.name_prefix}-cluster-pg"
  
  parameter {
    name  = "shared_preload_libraries"
    value = "pg_stat_statements"
  }
  
  parameter {
    name  = "log_statement"
    value = "all"
  }
  
  tags = var.tags
}

resource "aws_rds_cluster" "main" {
  cluster_identifier      = "${var.name_prefix}-aurora-cluster"
  engine                 = "aurora-postgresql"
  engine_mode            = "provisioned"
  engine_version         = "15.4"
  database_name          = "courseplatform"
  master_username        = "postgres"
  master_password        = var.master_password
  
  serverlessv2_scaling_configuration {
    max_capacity = 16
    min_capacity = 0.5
  }
  
  vpc_security_group_ids   = [var.security_group_id]
  db_subnet_group_name     = aws_db_subnet_group.main.name
  db_cluster_parameter_group_name = aws_rds_cluster_parameter_group.main.name
  
  backup_retention_period  = var.environment == "production" ? 7 : 1
  preferred_backup_window  = "03:00-04:00"
  preferred_maintenance_window = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = var.environment != "production"
  deletion_protection = var.environment == "production"
  
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-aurora-cluster"
  })
}

resource "aws_rds_cluster_instance" "cluster_instances" {
  count              = var.environment == "production" ? 2 : 1
  identifier         = "${var.name_prefix}-aurora-${count.index}"
  cluster_identifier = aws_rds_cluster.main.id
  instance_class     = "db.serverless"
  engine             = aws_rds_cluster.main.engine
  engine_version     = aws_rds_cluster.main.engine_version
  
  performance_insights_enabled = true
  monitoring_interval          = 60
  monitoring_role_arn          = aws_iam_role.rds_enhanced_monitoring.arn
  
  tags = merge(var.tags, {
    Name = "${var.name_prefix}-aurora-instance-${count.index}"
  })
}

# Enhanced Monitoring Role
resource "aws_iam_role" "rds_enhanced_monitoring" {
  name = "${var.name_prefix}-rds-enhanced-monitoring"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "monitoring.rds.amazonaws.com"
        }
      }
    ]
  })
  
  tags = var.tags
}

resource "aws_iam_role_policy_attachment" "rds_enhanced_monitoring" {
  role       = aws_iam_role.rds_enhanced_monitoring.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole"
}
```

## Docker Configuration

### Frontend Dockerfile
```dockerfile
# Frontend Dockerfile
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install pnpm
RUN npm install -g pnpm

# Copy package files
COPY package.json pnpm-lock.yaml* ./
COPY apps/frontend/package.json ./apps/frontend/
COPY libs/*/package.json ./libs/*/

# Install dependencies
RUN pnpm install --frozen-lockfile

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Set environment variables for build
ENV NEXT_TELEMETRY_DISABLED 1
ENV NODE_ENV production

# Build the frontend application
RUN npm run build:frontend

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built application
COPY --from=builder /app/dist/apps/frontend/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/dist/apps/frontend/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/dist/apps/frontend/.next/static ./.next/static

USER nextjs

EXPOSE 3000

ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

### Backend Dockerfile
```dockerfile
# Backend Dockerfile
FROM node:18-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app

# Install pnpm
RUN npm install -g pnpm

# Copy package files
COPY package.json pnpm-lock.yaml* ./
COPY apps/backend/package.json ./apps/backend/
COPY libs/*/package.json ./libs/*/

# Install dependencies
RUN pnpm install --frozen-lockfile

# Build the source code
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .

# Build the backend application
RUN npm run build:backend

# Production image
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production

RUN addgroup --system --gid 1001 nestjs
RUN adduser --system --uid 1001 nestjs

# Copy built application
COPY --from=builder --chown=nestjs:nestjs /app/dist/apps/backend ./
COPY --from=builder --chown=nestjs:nestjs /app/node_modules ./node_modules

USER nestjs

EXPOSE 4000

CMD ["node", "main.js"]
```

## GitHub Actions CI/CD Pipeline

### Main Pipeline (.github/workflows/deploy.yml)
```yaml
name: Deploy to AWS

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY_FRONTEND: course-platform-frontend
  ECR_REPOSITORY_BACKEND: course-platform-backend

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install pnpm
        run: npm install -g pnpm
      
      - name: Install dependencies
        run: pnpm install --frozen-lockfile
      
      - name: Run linting
        run: |
          npm run lint:frontend
          npm run lint:backend
      
      - name: Run type checking
        run: |
          npm run type-check:frontend
          npm run type-check:backend
      
      - name: Run unit tests
        run: |
          npm run test:frontend
          npm run test:backend
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379
      
      - name: Run E2E tests
        run: npm run test:e2e
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379

  security-scan:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    outputs:
      frontend-image: ${{ steps.frontend-image.outputs.image }}
      backend-image: ${{ steps.backend-image.outputs.image }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push frontend image
        id: frontend-image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./apps/frontend/Dockerfile
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY_FRONTEND }}:${{ github.sha }}
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY_FRONTEND }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Build and push backend image
        id: backend-image
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./apps/backend/Dockerfile
          push: true
          tags: |
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY_BACKEND }}:${{ github.sha }}
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY_BACKEND }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy to staging
        run: |
          aws ecs update-service \
            --cluster course-platform-staging-cluster \
            --service course-platform-staging-frontend \
            --force-new-deployment \
            --task-definition course-platform-staging-frontend
          
          aws ecs update-service \
            --cluster course-platform-staging-cluster \
            --service course-platform-staging-backend \
            --force-new-deployment \
            --task-definition course-platform-staging-backend
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster course-platform-staging-cluster \
            --services course-platform-staging-frontend course-platform-staging-backend

  deploy-production:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy database migrations
        run: |
          # Run database migrations before deploying new application version
          aws ecs run-task \
            --cluster course-platform-production-cluster \
            --task-definition course-platform-production-migration \
            --launch-type FARGATE \
            --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx,subnet-yyy],securityGroups=[sg-xxx],assignPublicIp=ENABLED}"
      
      - name: Deploy to production
        run: |
          aws ecs update-service \
            --cluster course-platform-production-cluster \
            --service course-platform-production-frontend \
            --force-new-deployment \
            --task-definition course-platform-production-frontend
          
          aws ecs update-service \
            --cluster course-platform-production-cluster \
            --service course-platform-production-backend \
            --force-new-deployment \
            --task-definition course-platform-production-backend
      
      - name: Wait for deployment
        run: |
          aws ecs wait services-stable \
            --cluster course-platform-production-cluster \
            --services course-platform-production-frontend course-platform-production-backend
      
      - name: Run health checks
        run: |
          curl -f https://courseplatform.com/health || exit 1
          curl -f https://courseplatform.com/api/health || exit 1
      
      - name: Notify deployment success
        uses: 8398a7/action-slack@v3
        with:
          status: success
          channel: '#deployments'
          text: 'Production deployment successful! :rocket:'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### Database Migration Job
```yaml
name: Database Migration

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to run migration'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  migrate:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Run database migration
        run: |
          aws ecs run-task \
            --cluster course-platform-${{ github.event.inputs.environment }}-cluster \
            --task-definition course-platform-${{ github.event.inputs.environment }}-migration \
            --launch-type FARGATE \
            --network-configuration "awsvpcConfiguration={subnets=[subnet-xxx],securityGroups=[sg-xxx],assignPublicIp=ENABLED}" \
            --overrides '{
              "containerOverrides": [
                {
                  "name": "migration",
                  "command": ["npx", "prisma", "migrate", "deploy"]
                }
              ]
            }'
```

## Monitoring and Observability

### CloudWatch Dashboard Configuration
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/ECS", "CPUUtilization", "ServiceName", "course-platform-frontend"],
          [".", "MemoryUtilization", ".", "."],
          [".", "CPUUtilization", "ServiceName", "course-platform-backend"],
          [".", "MemoryUtilization", ".", "."]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "ECS Service Metrics"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/RDS", "CPUUtilization", "DBClusterIdentifier", "course-platform-aurora-cluster"],
          [".", "DatabaseConnections", ".", "."],
          [".", "ServerlessDatabaseCapacity", ".", "."]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Database Metrics"
      }
    },
    {
      "type": "log",
      "properties": {
        "query": "SOURCE '/ecs/course-platform-backend'\n| fields @timestamp, @message\n| filter @message like /ERROR/\n| sort @timestamp desc\n| limit 100",
        "region": "us-east-1",
        "title": "Application Errors"
      }
    }
  ]
}
```

### Alerts Configuration
```yaml
# CloudWatch Alarms (alerts.tf)
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.name_prefix}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors ECS CPU utilization"
  
  dimensions = {
    ServiceName = "${var.name_prefix}-backend"
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "database_connections" {
  alarm_name          = "${var.name_prefix}-database-connections"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors RDS connection count"
  
  dimensions = {
    DBClusterIdentifier = aws_rds_cluster.main.cluster_identifier
  }
  
  alarm_actions = [aws_sns_topic.alerts.arn]
}

resource "aws_sns_topic" "alerts" {
  name = "${var.name_prefix}-alerts"
}

resource "aws_sns_topic_subscription" "alerts_email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}
```

This comprehensive infrastructure configuration provides a complete deployment solution for the multi-tenant online course platform, including:

1. **Infrastructure as Code**: Complete Terraform modules for all AWS resources
2. **Container Orchestration**: ECS Fargate with auto-scaling and load balancing
3. **CI/CD Pipeline**: GitHub Actions with automated testing, security scanning, and deployment
4. **Database Management**: Aurora Serverless with automated backups and monitoring
5. **Monitoring & Alerting**: CloudWatch dashboards, alarms, and SNS notifications
6. **Security**: IAM roles, security groups, and secret management

The infrastructure is designed to be cost-effective, scalable, and maintainable with proper separation of concerns across different environments.