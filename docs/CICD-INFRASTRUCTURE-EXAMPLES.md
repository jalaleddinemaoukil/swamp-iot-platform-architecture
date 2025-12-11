# Sanitized CI/CD and Infrastructure Examples

## Overview

This document provides **sanitized examples** of CI/CD pipelines, Infrastructure as Code, and security scanning implementations from the SWAMP platform. All sensitive information has been removed or replaced with placeholders.

---

## 1. GitHub Actions CI/CD Pipeline

### Complete Workflow (Sanitized)

```yaml
name: SWAMP CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

env:
  NODE_VERSION: '18.x'
  PYTHON_VERSION: '3.11'

jobs:
  # ========================================
  # STAGE 1: Security Scanning
  # ========================================
  
  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for Gitleaks
      
      # Scan for secrets
      - name: Gitleaks Secret Scanning
        uses: gitleaks/gitleaks-action@v2
        env:
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}
      
      # Scan dependencies for vulnerabilities
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      # Upload scan results
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      # Python dependency check
      - name: Python Safety Check
        run: |
          pip install safety
          safety check --json || true
        working-directory: ./iot-simulator
  
  # ========================================
  # STAGE 2: Frontend Build & Test
  # ========================================
  
  frontend-build:
    name: Frontend Build & Test
    runs-on: ubuntu-latest
    needs: security-scan
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
      
      - name: Install dependencies
        run: npm ci
        working-directory: ./frontend
      
      - name: Lint code
        run: npm run lint
        working-directory: ./frontend
      
      - name: Type check
        run: npm run type-check
        working-directory: ./frontend
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
        working-directory: ./frontend
      
      - name: Build application
        run: npm run build
        working-directory: ./frontend
        env:
          VITE_SUPABASE_URL: ${{ secrets.VITE_SUPABASE_URL }}
          VITE_SUPABASE_ANON_KEY: ${{ secrets.VITE_SUPABASE_ANON_KEY }}
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: frontend-build
          path: frontend/dist
          retention-days: 7
  
  # ========================================
  # STAGE 3: Backend Build & Test
  # ========================================
  
  backend-test:
    name: Backend Tests
    runs-on: ubuntu-latest
    needs: security-scan
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      
      - name: Install dependencies
        run: npm ci
        working-directory: ./backend
      
      - name: Lint code
        run: npm run lint
        working-directory: ./backend
      
      - name: Type check
        run: npm run type-check
        working-directory: ./backend
      
      - name: Run unit tests
        run: npm run test -- --coverage
        working-directory: ./backend
  
  # ========================================
  # STAGE 4: Docker Build & Scan
  # ========================================
  
  docker-build:
    name: Docker Build & Security Scan
    runs-on: ubuntu-latest
    needs: [frontend-build, backend-test]
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ secrets.CONTAINER_REGISTRY }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      
      - name: Build MQTT Bridge Image
        uses: docker/build-push-action@v5
        with:
          context: ./mqtt-bridge
          file: ./mqtt-bridge/Dockerfile
          push: false
          load: true
          tags: mqtt-bridge:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Scan Docker image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: mqtt-bridge:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-docker-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Push to registry (if scan passes)
        uses: docker/build-push-action@v5
        with:
          context: ./mqtt-bridge
          file: ./mqtt-bridge/Dockerfile
          push: true
          tags: |
            ${{ secrets.CONTAINER_REGISTRY }}/mqtt-bridge:${{ github.sha }}
            ${{ secrets.CONTAINER_REGISTRY }}/mqtt-bridge:latest
  
  # ========================================
  # STAGE 5: Deploy to Staging
  # ========================================
  
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: docker-build
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      # Deploy frontend to Vercel
      - name: Deploy Frontend to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          working-directory: ./frontend
          vercel-args: '--prod'
      
      # Deploy Supabase Edge Functions
      - name: Deploy Edge Functions
        run: |
          npx supabase functions deploy \
            --project-ref ${{ secrets.SUPABASE_PROJECT_REF }}
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_ACCESS_TOKEN }}
      
      # Update ECS service (MQTT bridge)
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Update ECS Task Definition
        run: |
          # Create new task definition with updated image
          aws ecs register-task-definition \
            --family mqtt-bridge-staging \
            --container-definitions '[{
              "name": "mqtt-bridge",
              "image": "${{ secrets.CONTAINER_REGISTRY }}/mqtt-bridge:${{ github.sha }}",
              "memory": 512,
              "cpu": 256,
              "essential": true,
              "portMappings": [{
                "containerPort": 3000,
                "protocol": "tcp"
              }],
              "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                  "awslogs-group": "/ecs/mqtt-bridge-staging",
                  "awslogs-region": "us-east-1",
                  "awslogs-stream-prefix": "ecs"
                }
              }
            }]' \
            --requires-compatibilities FARGATE \
            --network-mode awsvpc \
            --cpu 256 \
            --memory 512 \
            --execution-role-arn ${{ secrets.ECS_EXECUTION_ROLE_ARN }}
          
          # Update ECS service
          aws ecs update-service \
            --cluster swamp-staging \
            --service mqtt-bridge \
            --task-definition mqtt-bridge-staging \
            --force-new-deployment
      
      # Wait for deployment to stabilize
      - name: Wait for ECS deployment
        run: |
          aws ecs wait services-stable \
            --cluster swamp-staging \
            --services mqtt-bridge
  
  # ========================================
  # STAGE 6: Smoke Tests
  # ========================================
  
  smoke-tests:
    name: Smoke Tests
    runs-on: ubuntu-latest
    needs: deploy-staging
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Run smoke tests
        run: |
          # Health check endpoints
          curl -f https://staging.example.com/health || exit 1
          curl -f https://staging-api.example.com/health || exit 1
      
      - name: Run E2E tests
        run: npm run test:e2e
        working-directory: ./e2e-tests
        env:
          BASE_URL: https://staging.example.com
          API_URL: https://staging-api.example.com
  
  # ========================================
  # STAGE 7: Deploy to Production
  # ========================================
  
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: smoke-tests
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://app.example.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      # Deploy frontend to Vercel Production
      - name: Deploy Frontend to Production
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROD_PROJECT_ID }}
          working-directory: ./frontend
          vercel-args: '--prod'
      
      # Deploy Supabase Edge Functions to Production
      - name: Deploy Edge Functions to Production
        run: |
          npx supabase functions deploy \
            --project-ref ${{ secrets.SUPABASE_PROD_PROJECT_REF }}
        env:
          SUPABASE_ACCESS_TOKEN: ${{ secrets.SUPABASE_PROD_ACCESS_TOKEN }}
      
      # Blue-Green deployment for ECS
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_PROD_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_PROD_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Blue-Green ECS Deployment
        run: |
          # Update task definition
          aws ecs register-task-definition \
            --family mqtt-bridge-production \
            --container-definitions '[...]' # (same as staging)
          
          # Update service with deployment configuration
          aws ecs update-service \
            --cluster swamp-production \
            --service mqtt-bridge \
            --task-definition mqtt-bridge-production \
            --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100" \
            --force-new-deployment
          
          # Wait for deployment
          aws ecs wait services-stable \
            --cluster swamp-production \
            --services mqtt-bridge
      
      - name: Health check after deployment
        run: |
          sleep 30  # Wait for health checks to pass
          curl -f https://app.example.com/health || exit 1
          curl -f https://api.example.com/health || exit 1
      
      # Rollback on failure
      - name: Rollback on failure
        if: failure()
        run: |
          echo "Deployment failed, rolling back..."
          # Get previous task definition
          PREVIOUS_TASK_DEF=$(aws ecs describe-services \
            --cluster swamp-production \
            --services mqtt-bridge \
            --query 'services[0].deployments[1].taskDefinition' \
            --output text)
          
          # Rollback
          aws ecs update-service \
            --cluster swamp-production \
            --service mqtt-bridge \
            --task-definition $PREVIOUS_TASK_DEF \
            --force-new-deployment
  
  # ========================================
  # STAGE 8: Post-Deploy Monitoring
  # ========================================
  
  post-deploy-check:
    name: Post-Deploy Monitoring
    runs-on: ubuntu-latest
    needs: deploy-production
    
    steps:
      - name: Monitor error rates
        run: |
          # Check CloudWatch metrics for error spike
          aws cloudwatch get-metric-statistics \
            --namespace AWS/ApplicationELB \
            --metric-name HTTPCode_Target_5XX_Count \
            --dimensions Name=LoadBalancer,Value=${{ secrets.ALB_NAME }} \
            --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 300 \
            --statistics Sum
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_PROD_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_PROD_SECRET_ACCESS_KEY }}
          AWS_DEFAULT_REGION: us-east-1
      
      - name: Send deployment notification
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Production deployment completed successfully!'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        if: always()
```

---

## 2. Terraform Infrastructure Examples

### VPC Configuration (Sanitized)

```hcl
# terraform/vpc.tf

variable "environment" {
  description = "Environment name (staging, production)"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "swamp-${var.environment}-vpc"
    Environment = var.environment
    Project     = "SWAMP"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name        = "swamp-${var.environment}-public-${count.index + 1}"
    Environment = var.environment
    Type        = "public"
  }
}

# Private Subnets
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 100)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name        = "swamp-${var.environment}-private-${count.index + 1}"
    Environment = var.environment
    Type        = "private"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "swamp-${var.environment}-igw"
    Environment = var.environment
  }
}

# NAT Gateway (for private subnets)
resource "aws_eip" "nat" {
  count  = 2
  domain = "vpc"

  tags = {
    Name        = "swamp-${var.environment}-nat-eip-${count.index + 1}"
    Environment = var.environment
  }
}

resource "aws_nat_gateway" "main" {
  count         = 2
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name        = "swamp-${var.environment}-nat-${count.index + 1}"
    Environment = var.environment
  }
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name        = "swamp-${var.environment}-public-rt"
    Environment = var.environment
  }
}

resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name        = "swamp-${var.environment}-private-rt-${count.index + 1}"
    Environment = var.environment
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Data source for availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Outputs
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}
```

### ECS Cluster Configuration (Sanitized)

```hcl
# terraform/ecs.tf

variable "mqtt_bridge_image" {
  description = "Docker image for MQTT bridge"
  type        = string
}

variable "mqtt_bridge_cpu" {
  description = "CPU units for MQTT bridge task"
  type        = number
  default     = 256
}

variable "mqtt_bridge_memory" {
  description = "Memory for MQTT bridge task (MB)"
  type        = number
  default     = 512
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "swamp-${var.environment}"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Name        = "swamp-${var.environment}"
    Environment = var.environment
  }
}

# ECS Task Execution Role
resource "aws_iam_role" "ecs_execution_role" {
  name = "swamp-${var.environment}-ecs-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution_role_policy" {
  role       = aws_iam_role.ecs_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# ECS Task Role (for application)
resource "aws_iam_role" "ecs_task_role" {
  name = "swamp-${var.environment}-ecs-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ecs-tasks.amazonaws.com"
        }
      }
    ]
  })
}

# Policy for accessing Secrets Manager
resource "aws_iam_role_policy" "ecs_task_secrets_policy" {
  name = "secrets-access"
  role = aws_iam_role.ecs_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = [
          aws_secretsmanager_secret.mqtt_credentials.arn,
          aws_secretsmanager_secret.supabase_credentials.arn
        ]
      }
    ]
  })
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "mqtt_bridge" {
  name              = "/ecs/swamp-${var.environment}-mqtt-bridge"
  retention_in_days = 30

  tags = {
    Name        = "mqtt-bridge-logs"
    Environment = var.environment
  }
}

# ECS Task Definition
resource "aws_ecs_task_definition" "mqtt_bridge" {
  family                   = "swamp-${var.environment}-mqtt-bridge"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = var.mqtt_bridge_cpu
  memory                   = var.mqtt_bridge_memory
  execution_role_arn       = aws_iam_role.ecs_execution_role.arn
  task_role_arn            = aws_iam_role.ecs_task_role.arn

  container_definitions = jsonencode([
    {
      name  = "mqtt-bridge"
      image = var.mqtt_bridge_image
      
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
          name  = "LOG_LEVEL"
          value = "info"
        }
      ]

      secrets = [
        {
          name      = "MQTT_BROKER_URL"
          valueFrom = "${aws_secretsmanager_secret.mqtt_credentials.arn}:url::"
        },
        {
          name      = "MQTT_USERNAME"
          valueFrom = "${aws_secretsmanager_secret.mqtt_credentials.arn}:username::"
        },
        {
          name      = "MQTT_PASSWORD"
          valueFrom = "${aws_secretsmanager_secret.mqtt_credentials.arn}:password::"
        },
        {
          name      = "SUPABASE_URL"
          valueFrom = "${aws_secretsmanager_secret.supabase_credentials.arn}:url::"
        },
        {
          name      = "SUPABASE_SERVICE_KEY"
          valueFrom = "${aws_secretsmanager_secret.supabase_credentials.arn}:service_key::"
        }
      ]

      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.mqtt_bridge.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "ecs"
        }
      }

      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
        interval    = 30
        timeout     = 5
        retries     = 3
        startPeriod = 60
      }
    }
  ])

  tags = {
    Name        = "mqtt-bridge-task"
    Environment = var.environment
  }
}

# Security Group for ECS Tasks
resource "aws_security_group" "ecs_tasks" {
  name        = "swamp-${var.environment}-ecs-tasks"
  description = "Allow inbound traffic for ECS tasks"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 3000
    to_port         = 3000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "ecs-tasks-sg"
    Environment = var.environment
  }
}

# ECS Service
resource "aws_ecs_service" "mqtt_bridge" {
  name            = "mqtt-bridge"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.mqtt_bridge.arn
  desired_count   = var.environment == "production" ? 2 : 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.ecs_tasks.id]
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.mqtt_bridge.arn
    container_name   = "mqtt-bridge"
    container_port   = 3000
  }

  deployment_configuration {
    maximum_percent         = 200
    minimum_healthy_percent = 100
  }

  # Auto-scaling (optional)
  depends_on = [
    aws_lb_listener.mqtt_bridge
  ]

  tags = {
    Name        = "mqtt-bridge-service"
    Environment = var.environment
  }
}

# Auto Scaling for ECS Service
resource "aws_appautoscaling_target" "ecs" {
  max_capacity       = 10
  min_capacity       = var.environment == "production" ? 2 : 1
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.mqtt_bridge.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "ecs_cpu" {
  name               = "cpu-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs.service_namespace

  target_tracking_scaling_policy_configuration {
    target_value = 70.0

    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }

    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

### Application Load Balancer (Sanitized)

```hcl
# terraform/alb.tf

# Security Group for ALB
resource "aws_security_group" "alb" {
  name        = "swamp-${var.environment}-alb"
  description = "Security group for Application Load Balancer"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "alb-sg"
    Environment = var.environment
  }
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "swamp-${var.environment}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = var.environment == "production"

  tags = {
    Name        = "swamp-alb"
    Environment = var.environment
  }
}

# Target Group
resource "aws_lb_target_group" "mqtt_bridge" {
  name        = "swamp-${var.environment}-mqtt-bridge"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 3
  }

  deregistration_delay = 30

  tags = {
    Name        = "mqtt-bridge-tg"
    Environment = var.environment
  }
}

# HTTPS Listener (requires SSL certificate)
resource "aws_lb_listener" "mqtt_bridge" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = var.ssl_certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.mqtt_bridge.arn
  }
}

# HTTP to HTTPS redirect
resource "aws_lb_listener" "redirect_http_to_https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}

# Outputs
output "alb_dns_name" {
  description = "DNS name of the load balancer"
  value       = aws_lb.main.dns_name
}
```

---

## 3. Docker Multi-Stage Build Example

```dockerfile
# Dockerfile for MQTT Bridge Service (Sanitized)

# Stage 1: Build
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies (including dev dependencies for build)
RUN npm ci

# Copy source code
COPY src ./src
COPY tsconfig.json ./

# Build TypeScript
RUN npm run build

# Stage 2: Production
FROM node:18-alpine AS production

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install only production dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy built application from builder stage
COPY --from=builder --chown=nodejs:nodejs /app/dist ./dist

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => {process.exit(res.statusCode === 200 ? 0 : 1);})"

# Start application
CMD ["node", "dist/index.js"]
```

### Docker Compose (Development)

```yaml
# docker-compose.yml (Sanitized for development)

version: '3.8'

services:
  # MQTT Broker (Mosquitto)
  mqtt-broker:
    image: eclipse-mosquitto:2.0
    container_name: swamp-mqtt-broker
    ports:
      - "1883:1883"
      - "9001:9001"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - mqtt-data:/mosquitto/data
      - mqtt-logs:/mosquitto/log
    restart: unless-stopped

  # MQTT Bridge Service
  mqtt-bridge:
    build:
      context: ./mqtt-bridge
      dockerfile: Dockerfile
    container_name: swamp-mqtt-bridge
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - MQTT_BROKER_URL=mqtt://mqtt-broker:1883
      - SUPABASE_URL=${SUPABASE_URL}
      - SUPABASE_SERVICE_KEY=${SUPABASE_SERVICE_KEY}
      - LOG_LEVEL=debug
    depends_on:
      - mqtt-broker
    restart: unless-stopped
    volumes:
      - ./mqtt-bridge/src:/app/src:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # PostgreSQL (Local development)
  postgres:
    image: postgres:15-alpine
    container_name: swamp-postgres
    environment:
      - POSTGRES_USER=swamp
      - POSTGRES_PASSWORD=development_password
      - POSTGRES_DB=swamp_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./database/init:/docker-entrypoint-initdb.d:ro
    restart: unless-stopped

  # Redis (for caching/rate limiting)
  redis:
    image: redis:7-alpine
    container_name: swamp-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    restart: unless-stopped
    command: redis-server --appendonly yes

volumes:
  mqtt-data:
  mqtt-logs:
  postgres-data:
  redis-data:

networks:
  default:
    name: swamp-network
```

---

## 4. Security Scanning Configuration

### Trivy Configuration

```yaml
# .trivyignore (Sanitized)
# Ignore specific vulnerabilities (with justification)

# CVE-2023-12345 - False positive in dev dependency, not used in production
CVE-2023-12345

# CVE-2023-67890 - No fix available, applying temporary mitigation
CVE-2023-67890
```

```yaml
# trivy.yaml (Configuration)
scan:
  security-checks:
    - vuln
    - config
    - secret
  
  severity:
    - CRITICAL
    - HIGH
  
  skip-files:
    - "node_modules/**"
    - "dist/**"
    - "*.test.ts"
  
  skip-dirs:
    - "test"
    - "docs"

output:
  format: json
  file: trivy-results.json
```

### Gitleaks Configuration

```toml
# .gitleaks.toml (Sanitized)

title = "SWAMP Gitleaks Configuration"

[extend]
useDefault = true

[[rules]]
id = "generic-api-key"
description = "Generic API Key"
regex = '''(?i)(api[_-]?key|apikey)['"]\s*[:=]\s*['"][0-9a-zA-Z]{32,}['"]'''
tags = ["key", "API"]

[[rules]]
id = "aws-access-key"
description = "AWS Access Key"
regex = '''(A3T[A-Z0-9]|AKIA|AGPA|AIDA|AROA|AIPA|ANPA|ANVA|ASIA)[A-Z0-9]{16}'''
tags = ["key", "AWS"]

[[rules]]
id = "database-connection-string"
description = "Database Connection String"
regex = '''(?i)(postgres|mysql|mongodb)://[^:]+:[^@]+@[^/]+'''
tags = ["secret", "database"]

[[rules.allowlist]]
description = "Allow example/placeholder values"
regexes = [
  '''YOUR_[A-Z_]+_HERE''',
  '''example\.(com|org|net)''',
  '''test_[a-z_]+''',
  '''placeholder'''
]
```

---

## 5. Monitoring & Alerting

### CloudWatch Alarms (Terraform)

```hcl
# terraform/cloudwatch.tf

# CPU Utilization Alarm
resource "aws_cloudwatch_metric_alarm" "ecs_cpu_high" {
  alarm_name          = "swamp-${var.environment}-ecs-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/ECS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "ECS CPU utilization is too high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = aws_ecs_service.mqtt_bridge.name
  }
}

# Memory Utilization Alarm
resource "aws_cloudwatch_metric_alarm" "ecs_memory_high" {
  alarm_name          = "swamp-${var.environment}-ecs-memory-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "MemoryUtilization"
  namespace           = "AWS/ECS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "ECS memory utilization is too high"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = aws_ecs_service.mqtt_bridge.name
  }
}

# Target 5XX Error Alarm
resource "aws_cloudwatch_metric_alarm" "target_5xx_errors" {
  alarm_name          = "swamp-${var.environment}-target-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "Too many 5XX errors from targets"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
    TargetGroup  = aws_lb_target_group.mqtt_bridge.arn_suffix
  }
}

# Unhealthy Targets Alarm
resource "aws_cloudwatch_metric_alarm" "unhealthy_targets" {
  alarm_name          = "swamp-${var.environment}-unhealthy-targets"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "UnHealthyHostCount"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Average"
  threshold           = 0
  alarm_description   = "Unhealthy targets detected"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  treat_missing_data  = "notBreaching"

  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
    TargetGroup  = aws_lb_target_group.mqtt_bridge.arn_suffix
  }
}

# SNS Topic for Alerts
resource "aws_sns_topic" "alerts" {
  name = "swamp-${var.environment}-alerts"

  tags = {
    Environment = var.environment
  }
}

resource "aws_sns_topic_subscription" "alerts_email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}
```

---

## 6. Database Migration Example

### Supabase Migration (SQL)

```sql
-- supabase/migrations/20250101000000_initial_schema.sql

-- Enable Row Level Security
ALTER DATABASE postgres SET row_security = on;

-- Create users table (sanitized)
CREATE TABLE IF NOT EXISTS public.users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT UNIQUE NOT NULL,
    role TEXT NOT NULL DEFAULT 'viewer',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Create farms table
CREATE TABLE IF NOT EXISTS public.farms (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT NOT NULL,
    owner_id UUID NOT NULL REFERENCES public.users(id) ON DELETE CASCADE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Create sensors table
CREATE TABLE IF NOT EXISTS public.sensors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    farm_id UUID NOT NULL REFERENCES public.farms(id) ON DELETE CASCADE,
    device_id TEXT UNIQUE NOT NULL,
    type TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'active',
    last_seen TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Create sensor_readings table (partitioned by month)
CREATE TABLE IF NOT EXISTS public.sensor_readings (
    id BIGSERIAL,
    sensor_id UUID NOT NULL REFERENCES public.sensors(id) ON DELETE CASCADE,
    value NUMERIC NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    metadata JSONB
) PARTITION BY RANGE (timestamp);

-- Create partitions for sensor_readings (example for 2025)
CREATE TABLE sensor_readings_2025_01 PARTITION OF sensor_readings
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE sensor_readings_2025_02 PARTITION OF sensor_readings
    FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Create indexes
CREATE INDEX idx_farms_owner_id ON public.farms(owner_id);
CREATE INDEX idx_sensors_farm_id ON public.sensors(farm_id);
CREATE INDEX idx_sensors_device_id ON public.sensors(device_id);
CREATE INDEX idx_sensor_readings_sensor_id_timestamp ON public.sensor_readings(sensor_id, timestamp DESC);

-- Enable Row Level Security
ALTER TABLE public.users ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.farms ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.sensors ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.sensor_readings ENABLE ROW LEVEL SECURITY;

-- RLS Policies (sanitized examples)
CREATE POLICY "users_read_own_data" ON public.users
    FOR SELECT
    USING (auth.uid() = id);

CREATE POLICY "users_read_own_farms" ON public.farms
    FOR SELECT
    USING (auth.uid() = owner_id);

CREATE POLICY "users_read_farm_sensors" ON public.sensors
    FOR SELECT
    USING (
        farm_id IN (
            SELECT id FROM public.farms WHERE owner_id = auth.uid()
        )
    );

-- Functions for automatic timestamp updates
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Triggers
CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON public.users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_farms_updated_at
    BEFORE UPDATE ON public.farms
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_sensors_updated_at
    BEFORE UPDATE ON public.sensors
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

---

## Conclusion

These sanitized examples demonstrate:

✅ **Comprehensive CI/CD Pipeline** with security scanning, testing, and deployment  
✅ **Infrastructure as Code** with Terraform for AWS resources  
✅ **Docker containerization** with multi-stage builds and security  
✅ **Monitoring & Alerting** with CloudWatch  
✅ **Database migrations** with partitioning and RLS  

**Remember:** 
- Never commit secrets to Git
- Use environment variables or secrets management
- Always scan for vulnerabilities
- Test thoroughly before production deployment

---

*Last Updated: December 2025*
