# Complete Terraform Deployment for AWS Infrastructure

## Project Structure

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── backend.tf
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── rds/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── elasticache/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── s3/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev/
    │   └── terraform.tfvars
    ├── staging/
    │   └── terraform.tfvars
    └── production/
        └── terraform.tfvars
```

## 1. Backend Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "financial-services-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:ACCOUNT_ID:key/KEY_ID"
    dynamodb_table = "terraform-state-lock"
  }
}

# Create S3 bucket for state (run this first separately)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "financial-services-terraform-state"
  
  lifecycle {
    prevent_destroy = true
  }
  
  tags = {
    Name = "Terraform State Bucket"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_state_lock" {
  name           = "terraform-state-lock"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  tags = {
    Name = "Terraform State Lock Table"
  }
}
```

## 2. Provider Configuration

```hcl
# providers.tf
terraform {
  required_version = ">= 1.6.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = var.common_tags
  }
}

provider "aws" {
  alias  = "secondary"
  region = var.secondary_region
  
  default_tags {
    tags = merge(var.common_tags, {
      Region = "secondary"
    })
  }
}

provider "kubernetes" {
  host                   = module.eks.cluster_endpoint
  cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
  
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "aws"
    args = [
      "eks",
      "get-token",
      "--cluster-name",
      module.eks.cluster_name
    ]
  }
}

provider "helm" {
  kubernetes {
    host                   = module.eks.cluster_endpoint
    cluster_ca_certificate = base64decode(module.eks.cluster_certificate_authority_data)
    
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "aws"
      args = [
        "eks",
        "get-token",
        "--cluster-name",
        module.eks.cluster_name
      ]
    }
  }
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
data "aws_availability_zones" "available" {
  state = "available"
}
```

## 3. Main Configuration

```hcl
# main.tf
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
  
  tags = merge(
    var.common_tags,
    {
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  )
}

# VPC Module
module "vpc" {
  source = "./modules/vpc"
  
  name_prefix = local.name_prefix
  cidr_block  = var.vpc_cidr
  azs         = local.azs
  
  public_subnet_cidrs   = var.public_subnet_cidrs
  private_subnet_cidrs  = var.private_subnet_cidrs
  database_subnet_cidrs = var.database_subnet_cidrs
  cache_subnet_cidrs    = var.cache_subnet_cidrs
  
  enable_nat_gateway   = true
  single_nat_gateway   = var.environment != "production"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = local.tags
}

# EKS Module
module "eks" {
  source = "./modules/eks"
  
  cluster_name    = "${local.name_prefix}-eks"
  cluster_version = var.kubernetes_version
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  
  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true
  
  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent = true
    }
    aws-ebs-csi-driver = {
      most_recent = true
    }
  }
  
  eks_managed_node_groups = {
    system = {
      name           = "system-node-group"
      instance_types = ["t3.large"]
      
      min_size     = 3
      max_size     = 6
      desired_size = 3
      
      labels = {
        role = "system"
      }
      
      taints = []
    }
    
    application = {
      name           = "application-node-group"
      instance_types = ["m5.2xlarge"]
      
      min_size     = var.environment == "production" ? 6 : 2
      max_size     = var.environment == "production" ? 20 : 6
      desired_size = var.environment == "production" ? 6 : 2
      
      labels = {
        role = "application"
      }
      
      taints = []
    }
  }
  
  tags = local.tags
}

# RDS Module
module "rds" {
  source = "./modules/rds"
  
  identifier = "${local.name_prefix}-db"
  
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = var.rds_instance_class
  
  allocated_storage     = var.rds_allocated_storage
  max_allocated_storage = var.rds_max_allocated_storage
  storage_encrypted     = true
  kms_key_id            = aws_kms_key.rds.arn
  
  db_name  = var.db_name
  username = var.db_username
  password = random_password.db_password.result
  
  multi_az               = var.environment == "production"
  db_subnet_group_name   = module.vpc.database_subnet_group_name
  vpc_security_group_ids = [aws_security_group.rds.id]
  
  backup_retention_period = var.environment == "production" ? 35 : 7
  backup_window           = "03:00-04:00"
  maintenance_window      = "sun:04:00-sun:05:00"
  
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  
  performance_insights_enabled    = true
  performance_insights_kms_key_id = aws_kms_key.rds.arn
  
  deletion_protection       = var.environment == "production"
  skip_final_snapshot       = var.environment != "production"
  final_snapshot_identifier = "${local.name_prefix}-final-snapshot"
  
  tags = local.tags
}

# Read Replicas
module "rds_replica_1" {
  source = "./modules/rds"
  count  = var.environment == "production" ? 1 : 0
  
  identifier          = "${local.name_prefix}-replica-1"
  replicate_source_db = module.rds.db_instance_id
  instance_class      = var.rds_instance_class
  
  skip_final_snapshot = true
  
  tags = local.tags
}

module "rds_replica_2" {
  source = "./modules/rds"
  count  = var.environment == "production" ? 1 : 0
  
  identifier          = "${local.name_prefix}-replica-2"
  replicate_source_db = module.rds.db_instance_id
  instance_class      = var.rds_instance_class
  
  skip_final_snapshot = true
  
  tags = local.tags
}

# ElastiCache Module
module "elasticache" {
  source = "./modules/elasticache"
  
  replication_group_id          = "${local.name_prefix}-redis"
  replication_group_description = "Redis cluster for ${local.name_prefix}"
  
  engine_version = "7.0"
  node_type      = var.redis_node_type
  num_cache_clusters = var.environment == "production" ? 3 : 2
  
  subnet_group_name  = module.vpc.cache_subnet_group_name
  security_group_ids = [aws_security_group.redis.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  auth_token_enabled         = true
  auth_token                 = random_password.redis_password.result
  
  automatic_failover_enabled = true
  multi_az_enabled           = var.environment == "production"
  
  snapshot_retention_limit = var.environment == "production" ? 5 : 1
  snapshot_window          = "03:00-05:00"
  maintenance_window       = "sun:05:00-sun:07:00"
  
  tags = local.tags
}

# S3 Buckets
module "s3_frontend" {
  source = "./modules/s3"
  
  bucket_name = "${local.name_prefix}-frontend"
  
  versioning_enabled = true
  
  lifecycle_rules = [
    {
      id      = "delete-old-versions"
      enabled = true
      
      noncurrent_version_expiration = {
        days = 90
      }
    }
  ]
  
  cors_rules = [
    {
      allowed_headers = ["*"]
      allowed_methods = ["GET", "HEAD"]
      allowed_origins = ["https://*.financialservices.com"]
      expose_headers  = ["ETag"]
      max_age_seconds = 3000
    }
  ]
  
  tags = local.tags
}

module "s3_data" {
  source = "./modules/s3"
  
  bucket_name = "${local.name_prefix}-data"
  
  versioning_enabled = true
  
  lifecycle_rules = [
    {
      id      = "archive-old-documents"
      enabled = true
      
      transitions = [
        {
          days          = 90
          storage_class = "STANDARD_IA"
        },
        {
          days          = 365
          storage_class = "GLACIER"
        }
      ]
    },
    {
      id      = "delete-old-logs"
      enabled = true
      
      filter = {
        prefix = "logs/"
      }
      
      expiration = {
        days = 90
      }
    }
  ]
  
  replication_configuration = var.environment == "production" ? {
    role = aws_iam_role.s3_replication[0].arn
    
    rules = [
      {
        id       = "replicate-to-dr"
        status   = "Enabled"
        priority = 1
        
        destination = {
          bucket        = module.s3_data_dr[0].bucket_arn
          storage_class = "STANDARD"
        }
      }
    ]
  } : null
  
  tags = local.tags
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "${local.name_prefix} CDN"
  default_root_object = "index.html"
  price_class         = var.environment == "production" ? "PriceClass_All" : "PriceClass_100"
  
  origin {
    domain_name = module.s3_frontend.bucket_regional_domain_name
    origin_id   = "S3-Frontend"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.main.cloudfront_access_identity_path
    }
  }
  
  origin {
    domain_name = aws_lb.main.dns_name
    origin_id   = "ALB-Backend"
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }
  
  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-Frontend"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    min_ttl     = 0
    default_ttl = 3600
    max_ttl     = 86400
  }
  
  ordered_cache_behavior {
    path_pattern           = "/api/*"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "ALB-Backend"
    viewer_protocol_policy = "https-only"
    compress               = true
    
    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Host"]
      cookies {
        forward = "all"
      }
    }
    
    min_ttl     = 0
    default_ttl = 0
    max_ttl     = 0
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
  
  web_acl_id = aws_wafv2_web_acl.main.arn
  
  tags = local.tags
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "${local.name_prefix}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnet_ids
  
  enable_deletion_protection       = var.environment == "production"
  enable_http2                     = true
  enable_cross_zone_load_balancing = true
  
  access_logs {
    bucket  = module.s3_logs.bucket_id
    prefix  = "alb"
    enabled = true
  }
  
  tags = local.tags
}

# Monitoring Module
module "monitoring" {
  source = "./modules/monitoring"
  
  name_prefix = local.name_prefix
  
  log_retention_days = var.environment == "production" ? 90 : 30
  
  alarm_email = var.alarm_email
  
  eks_cluster_name = module.eks.cluster_name
  rds_instance_id  = module.rds.db_instance_id
  alb_arn_suffix   = aws_lb.main.arn_suffix
  
  tags = local.tags
}

# Secrets Manager
resource "aws_secretsmanager_secret" "database_credentials" {
  name                    = "${local.name_prefix}/database/credentials"
  description             = "Database credentials"
  recovery_window_in_days = var.environment == "production" ? 30 : 0
  
  kms_key_id = aws_kms_key.secrets.id
  
  tags = local.tags
}

resource "aws_secretsmanager_secret_version" "database_credentials" {
  secret_id = aws_secretsmanager_secret.database_credentials.id
  secret_string = jsonencode({
    username = module.rds.db_instance_username
    password = random_password.db_password.result
    host     = module.rds.db_instance_endpoint
    port     = module.rds.db_instance_port
    dbname   = module.rds.db_instance_name
  })
}

# Random Passwords
resource "random_password" "db_password" {
  length  = 32
  special = true
}

resource "random_password" "redis_password" {
  length  = 32
  special = false
}

resource "random_password" "jwt_secret" {
  length  = 64
  special = false
}

# KMS Keys
resource "aws_kms_key" "eks" {
  description             = "KMS key for EKS cluster encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  tags = merge(local.tags, {
    Name = "${local.name_prefix}-eks-key"
  })
}

resource "aws_kms_key" "rds" {
  description             = "KMS key for RDS encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  tags = merge(local.tags, {
    Name = "${local.name_prefix}-rds-key"
  })
}

resource "aws_kms_key" "secrets" {
  description             = "KMS key for Secrets Manager"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  tags = merge(local.tags, {
    Name = "${local.name_prefix}-secrets-key"
  })
}
```

## 4. Variables

```hcl
# variables.tf
variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "financial-services"
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
  description = "Primary AWS region"
  type        = string
  default     = "us-east-1"
}

variable "secondary_region" {
  description = "Secondary AWS region for DR"
  type        = string
  default     = "us-west-2"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
  default     = ["10.0.10.0/24", "10.0.11.0/24", "10.0.12.0/24"]
}

variable "database_subnet_cidrs" {
  description = "CIDR blocks for database subnets"
  type        = list(string)
  default     = ["10.0.20.0/24", "10.0.21.0/24", "10.0.22.0/24"]
}

variable "cache_subnet_cidrs" {
  description = "CIDR blocks for cache subnets"
  type        = list(string)
  default     = ["10.0.30.0/24", "10.0.31.0/24", "10.0.32.0/24"]
}

variable "kubernetes_version" {
  description = "Kubernetes version"
  type        = string
  default     = "1.28"
}

variable "rds_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.r6g.4xlarge"
}

variable "rds_allocated_storage" {
  description = "RDS allocated storage in GB"
  type        = number
  default     = 2000
}

variable "rds_max_allocated_storage" {
  description = "RDS maximum allocated storage in GB"
  type        = number
  default     = 5000
}

variable "db_name" {
  description = "Database name"
  type        = string
  default     = "financialdb"
}

variable "db_username" {
  description = "Database username"
  type        = string
  default     = "dbadmin"
  sensitive   = true
}

variable "redis_node_type" {
  description = "ElastiCache node type"
  type        = string
  default     = "cache.r6g.xlarge"
}

variable "alarm_email" {
  description = "Email address for CloudWatch alarms"
  type        = string
}

variable "common_tags" {
  description = "Common tags to apply to all resources"
  type        = map(string)
  default = {
    Project     = "Financial Services"
    ManagedBy   = "Terraform"
    CostCenter  = "Engineering"
    Compliance  = "PCI-DSS"
  }
}
```

## 5. Outputs

```hcl
# outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "eks_cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
  sensitive   = true
}

output "eks_cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "rds_endpoint" {
  description = "RDS endpoint"
  value       = module.rds.db_instance_endpoint
  sensitive   = true
}

output "redis_endpoint" {
  description = "Redis endpoint"
  value       = module.elasticache.primary_endpoint_address
  sensitive   = true
}

output "alb_dns_name" {
  description = "ALB DNS name"
  value       = aws_lb.main.dns_name
}

output "cloudfront_domain_name" {
  description = "CloudFront distribution domain name"
  value       = aws_cloudfront_distribution.main.domain_name
}

output "s3_frontend_bucket" {
  description = "S3 frontend bucket name"
  value       = module.s3_frontend.bucket_id
}

output "ecr_repository_urls" {
  description = "ECR repository URLs"
  value = {
    for k, v in aws_ecr_repository.services : k => v.repository_url
  }
}
```

## 6. Production Environment Configuration

```hcl
# environments/production/terraform.tfvars
project_name     = "financial-services"
environment      = "production"
aws_region       = "us-east-1"
secondary_region = "us-west-2"

vpc_cidr = "10.0.0.0/16"

kubernetes_version = "1.28"

rds_instance_class        = "db.r6g.4xlarge"
rds_allocated_storage     = 2000
rds_max_allocated_storage = 5000

redis_node_type = "cache.r6g.xlarge"

alarm_email = "devops@company.com"

common_tags = {
  Project     = "Financial Services"
  Environment = "Production"
  ManagedBy   = "Terraform"
  CostCenter  = "Engineering"
  Compliance  = "PCI-DSS"
  Owner       = "Platform Team"
}
```

## 7. Deployment Commands

```bash
# Initialize Terraform
terraform init

# Select workspace (optional)
terraform workspace new production
terraform workspace select production

# Validate configuration
terraform validate

# Format code
terraform fmt -recursive

# Plan changes
terraform plan -var-file=environments/production/terraform.tfvars -out=tfplan

# Review plan
terraform show tfplan

# Apply changes
terraform apply tfplan

# Destroy resources (use with extreme caution)
terraform destroy -var-file=environments/production/terraform.tfvars
```

## 8. Deployment Script

```bash
#!/bin/bash
# deploy.sh

set -e

ENVIRONMENT=${1:-production}
ACTION=${2:-apply}

echo "Deploying to $ENVIRONMENT environment..."

# Initialize Terraform
terraform init

# Select workspace
terraform workspace select $ENVIRONMENT || terraform workspace new $ENVIRONMENT

# Validate
terraform validate

# Format check
terraform fmt -check -recursive

# Plan
terraform plan \
  -var-file=environments/$ENVIRONMENT/terraform.tfvars \
  -out=tfplan

# Show plan
terraform show tfplan

# Confirm
if [ "$ACTION" = "apply" ]; then
  read -p "Do you want to apply these changes? (yes/no): " CONFIRM
  if [ "$CONFIRM" = "yes" ]; then
    terraform apply tfplan
    echo "Deployment completed successfully!"
  else
    echo "Deployment cancelled."
    exit 1
  fi
fi
```

## 9. Post-Deployment Configuration

```bash
#!/bin/bash
# post-deploy.sh

# Update kubeconfig
aws eks update-kubeconfig \
  --region us-east-1 \
  --name financial-services-production-eks

# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=financial-services-production-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Install Cluster Autoscaler
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=financial-services-production-eks \
  --set awsRegion=us-east-1

# Install Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
istioctl install --set profile=production -y

# Create namespaces
kubectl create namespace financial-services
kubectl label namespace financial-services istio-injection=enabled

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

echo "Post-deployment configuration completed!"
```

## 10. Terraform Module Examples

### VPC Module
```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name_prefix}-vpc"
    }
  )
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.azs[count.index]
  
  map_public_ip_on_launch = true
  
  tags = merge(
    var.tags,
    {
      Name                     = "${var.name_prefix}-public-${count.index + 1}"
      "kubernetes.io/role/elb" = "1"
    }
  )
}

resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.azs[count.index]
  
  tags = merge(
    var.tags,
    {
      Name                              = "${var.name_prefix}-private-${count.index + 1}"
      "kubernetes.io/role/internal-elb" = "1"
    }
  )
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name_prefix}-igw"
    }
  )
}

resource "aws_eip" "nat" {
  count  = var.single_nat_gateway ? 1 : length(var.azs)
  domain = "vpc"
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name_prefix}-nat-eip-${count.index + 1}"
    }
  )
}

resource "aws_nat_gateway" "main" {
  count         = var.single_nat_gateway ? 1 : length(var.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name_prefix}-nat-${count.index + 1}"
    }
  )
  
  depends_on = [aws_internet_gateway.main]
}
```

This completes the Terraform deployment documentation. Would you like me to create the final summary document with cost estimation for AWS?
