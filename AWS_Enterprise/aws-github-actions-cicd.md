# GitHub Actions CI/CD Pipeline for AWS

## Overview

This document provides complete GitHub Actions workflows for building, testing, and deploying the financial services application to AWS EKS with blue-green deployment strategy.

## Repository Structure

```
.github/
├── workflows/
│   ├── backend-ci.yml
│   ├── backend-cd.yml
│   ├── frontend-ci.yml
│   ├── frontend-cd.yml
│   ├── infrastructure.yml
│   └── security-scan.yml
├── actions/
│   ├── setup-aws/
│   │   └── action.yml
│   └── deploy-to-eks/
│       └── action.yml
└── scripts/
    ├── blue-green-deploy.sh
    └── rollback.sh
```

## 1. Backend CI Pipeline

```yaml
# .github/workflows/backend-ci.yml
name: Backend CI

on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'
    paths:
      - 'backend/**'
      - '.github/workflows/backend-ci.yml'
  pull_request:
    branches:
      - main
      - develop
    paths:
      - 'backend/**'

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: financial-services
  JAVA_VERSION: '17'

jobs:
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Cache Maven packages
        uses: actions/cache@v3
        with:
          path: ~/.m2
          key: ${{ runner.os }}-m2-${{ hashFiles('**/pom.xml') }}
          restore-keys: ${{ runner.os }}-m2
      
      - name: Build with Maven
        run: |
          cd backend
          mvn clean compile -DskipTests
      
      - name: Run unit tests
        run: |
          cd backend
          mvn test
      
      - name: Run integration tests
        run: |
          cd backend
          mvn verify -DskipUTs=true
      
      - name: Generate test coverage report
        run: |
          cd backend
          mvn jacoco:report
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: backend/target/surefire-reports/
      
      - name: Upload coverage report
        uses: actions/upload-artifact@v3
        with:
          name: coverage-report
          path: backend/target/site/jacoco/
  
  code-quality:
    name: Code Quality Analysis
    runs-on: ubuntu-latest
    needs: build-and-test
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: SonarQube Scan
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
        run: |
          cd backend
          mvn sonar:sonar \
            -Dsonar.projectKey=financial-services \
            -Dsonar.host.url=$SONAR_HOST_URL \
            -Dsonar.login=$SONAR_TOKEN \
            -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
      
      - name: Check Quality Gate
        uses: sonarsource/sonarqube-quality-gate-action@master
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
  
  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest
    needs: build-and-test
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
      
      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'financial-services'
          path: 'backend'
          format: 'HTML'
          args: >
            --failOnCVSS 7
            --enableRetired
      
      - name: Upload OWASP report
        uses: actions/upload-artifact@v3
        with:
          name: owasp-report
          path: reports/
      
      - name: Snyk Security Scan
        uses: snyk/actions/maven@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --fail-on=all
      
      - name: Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: 'backend'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
  
  build-docker-image:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [code-quality, security-scan]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Build application
        run: |
          cd backend
          mvn clean package -DskipTests
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ github.sha }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
      
      - name: Scan Docker image with Trivy
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-image-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-image-results.sarif'
      
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json
      
      - name: Upload SBOM
        uses: actions/upload-artifact@v3
        with:
          name: sbom
          path: sbom.spdx.json
```

## 2. Backend CD Pipeline (Blue-Green Deployment)

```yaml
# .github/workflows/backend-cd.yml
name: Backend CD

on:
  workflow_run:
    workflows: ["Backend CI"]
    types:
      - completed
    branches:
      - main

env:
  AWS_REGION: us-east-1
  EKS_CLUSTER_NAME: financial-services-eks
  NAMESPACE: financial-services

jobs:
  deploy-to-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    environment:
      name: staging
      url: https://staging-api.financialservices.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --region ${{ env.AWS_REGION }} \
            --name ${{ env.EKS_CLUSTER_NAME }}-staging
      
      - name: Install Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.13.0'
      
      - name: Deploy to staging
        run: |
          helm upgrade --install financial-services ./helm/financial-services \
            --namespace ${{ env.NAMESPACE }}-staging \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set environment=staging \
            --set replicaCount=2 \
            --wait \
            --timeout 10m
      
      - name: Run smoke tests
        run: |
          kubectl run smoke-test \
            --image=curlimages/curl:latest \
            --rm -i --restart=Never \
            --namespace=${{ env.NAMESPACE }}-staging \
            -- curl -f http://financial-services/actuator/health
  
  integration-tests:
    name: Run Integration Tests
    runs-on: ubuntu-latest
    needs: deploy-to-staging
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Run integration tests
        env:
          TEST_URL: https://staging-api.financialservices.com
        run: |
          cd backend
          mvn verify -DskipUTs=true -Dtest.url=$TEST_URL
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: integration-test-results
          path: backend/target/failsafe-reports/
  
  deploy-to-production:
    name: Deploy to Production (Blue-Green)
    runs-on: ubuntu-latest
    needs: integration-tests
    environment:
      name: production
      url: https://api.financialservices.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --region ${{ env.AWS_REGION }} \
            --name ${{ env.EKS_CLUSTER_NAME }}
      
      - name: Install Helm
        uses: azure/setup-helm@v3
        with:
          version: '3.13.0'
      
      - name: Determine current deployment slot
        id: current-slot
        run: |
          CURRENT=$(kubectl get service financial-services \
            -n ${{ env.NAMESPACE }} \
            -o jsonpath='{.spec.selector.slot}' 2>/dev/null || echo "blue")
          
          if [ "$CURRENT" = "blue" ]; then
            echo "current=blue" >> $GITHUB_OUTPUT
            echo "new=green" >> $GITHUB_OUTPUT
          else
            echo "current=green" >> $GITHUB_OUTPUT
            echo "new=blue" >> $GITHUB_OUTPUT
          fi
          
          echo "Current slot: $CURRENT"
          echo "New slot: $([ "$CURRENT" = "blue" ] && echo "green" || echo "blue")"
      
      - name: Deploy to new slot (${{ steps.current-slot.outputs.new }})
        run: |
          helm upgrade --install financial-services-${{ steps.current-slot.outputs.new }} \
            ./helm/financial-services \
            --namespace ${{ env.NAMESPACE }} \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set environment=production \
            --set slot=${{ steps.current-slot.outputs.new }} \
            --set replicaCount=10 \
            --set resources.requests.cpu=1000m \
            --set resources.requests.memory=2Gi \
            --set resources.limits.cpu=2000m \
            --set resources.limits.memory=4Gi \
            --wait \
            --timeout 15m
      
      - name: Wait for new deployment to be ready
        run: |
          kubectl wait --for=condition=available \
            --timeout=600s \
            deployment/financial-services-${{ steps.current-slot.outputs.new }} \
            -n ${{ env.NAMESPACE }}
      
      - name: Run smoke tests on new deployment
        run: |
          NEW_SERVICE_IP=$(kubectl get service financial-services-${{ steps.current-slot.outputs.new }} \
            -n ${{ env.NAMESPACE }} \
            -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          
          for i in {1..10}; do
            if curl -f http://$NEW_SERVICE_IP/actuator/health; then
              echo "Health check passed"
              break
            fi
            echo "Health check failed, retrying..."
            sleep 10
          done
      
      - name: Gradual traffic shift - 10%
        run: |
          kubectl apply -f - <<EOF
          apiVersion: v1
          kind: Service
          metadata:
            name: financial-services
            namespace: ${{ env.NAMESPACE }}
          spec:
            selector:
              app: financial-services
            sessionAffinity: ClientIP
            ports:
              - protocol: TCP
                port: 80
                targetPort: 8080
          ---
          apiVersion: networking.istio.io/v1beta1
          kind: VirtualService
          metadata:
            name: financial-services
            namespace: ${{ env.NAMESPACE }}
          spec:
            hosts:
              - financial-services
            http:
              - match:
                  - headers:
                      x-version:
                        exact: canary
                route:
                  - destination:
                      host: financial-services-${{ steps.current-slot.outputs.new }}
                    weight: 100
              - route:
                  - destination:
                      host: financial-services-${{ steps.current-slot.outputs.current }}
                    weight: 90
                  - destination:
                      host: financial-services-${{ steps.current-slot.outputs.new }}
                    weight: 10
          EOF
      
      - name: Monitor for 5 minutes
        run: |
          echo "Monitoring deployment with 10% traffic..."
          sleep 300
      
      - name: Check error rates
        id: check-errors-10
        run: |
          ERROR_COUNT=$(aws cloudwatch get-metric-statistics \
            --namespace AWS/ApplicationELB \
            --metric-name HTTPCode_Target_5XX_Count \
            --dimensions Name=LoadBalancer,Value=${{ env.ALB_NAME }} \
            --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 300 \
            --statistics Sum \
            --query 'Datapoints[0].Sum' \
            --output text)
          
          echo "Error count: $ERROR_COUNT"
          
          if [ "$ERROR_COUNT" -gt 10 ]; then
            echo "Error rate too high, aborting deployment"
            exit 1
          fi
      
      - name: Gradual traffic shift - 50%
        run: |
          kubectl patch virtualservice financial-services \
            -n ${{ env.NAMESPACE }} \
            --type merge \
            -p '{
              "spec": {
                "http": [{
                  "route": [
                    {"destination": {"host": "financial-services-${{ steps.current-slot.outputs.current }}"}, "weight": 50},
                    {"destination": {"host": "financial-services-${{ steps.current-slot.outputs.new }}"}, "weight": 50}
                  ]
                }]
              }
            }'
      
      - name: Monitor for 5 minutes
        run: |
          echo "Monitoring deployment with 50% traffic..."
          sleep 300
      
      - name: Check error rates
        id: check-errors-50
        run: |
          ERROR_COUNT=$(aws cloudwatch get-metric-statistics \
            --namespace AWS/ApplicationELB \
            --metric-name HTTPCode_Target_5XX_Count \
            --dimensions Name=LoadBalancer,Value=${{ env.ALB_NAME }} \
            --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 300 \
            --statistics Sum \
            --query 'Datapoints[0].Sum' \
            --output text)
          
          echo "Error count: $ERROR_COUNT"
          
          if [ "$ERROR_COUNT" -gt 10 ]; then
            echo "Error rate too high, aborting deployment"
            exit 1
          fi
      
      - name: Complete traffic shift - 100%
        run: |
          kubectl patch virtualservice financial-services \
            -n ${{ env.NAMESPACE }} \
            --type merge \
            -p '{
              "spec": {
                "http": [{
                  "route": [
                    {"destination": {"host": "financial-services-${{ steps.current-slot.outputs.new }}"}, "weight": 100}
                  ]
                }]
              }
            }'
      
      - name: Final monitoring period
        run: |
          echo "Final monitoring with 100% traffic..."
          sleep 600
      
      - name: Validate deployment
        run: |
          # Check error rates
          ERROR_RATE=$(aws cloudwatch get-metric-statistics \
            --namespace AWS/ApplicationELB \
            --metric-name HTTPCode_Target_5XX_Count \
            --dimensions Name=LoadBalancer,Value=${{ env.ALB_NAME }} \
            --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 600 \
            --statistics Sum \
            --query 'Datapoints[0].Sum' \
            --output text)
          
          # Check response times
          P95_LATENCY=$(aws cloudwatch get-metric-statistics \
            --namespace AWS/ApplicationELB \
            --metric-name TargetResponseTime \
            --dimensions Name=LoadBalancer,Value=${{ env.ALB_NAME }} \
            --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
            --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
            --period 600 \
            --statistics p95 \
            --query 'Datapoints[0].ExtendedStatistics.p95' \
            --output text)
          
          echo "Error rate: $ERROR_RATE"
          echo "P95 latency: $P95_LATENCY"
          
          if [ "$ERROR_RATE" -gt 20 ] || (( $(echo "$P95_LATENCY > 2.0" | bc -l) )); then
            echo "Deployment validation failed"
            exit 1
          fi
          
          echo "Deployment validation successful"
      
      - name: Cleanup old deployment
        if: success()
        run: |
          helm uninstall financial-services-${{ steps.current-slot.outputs.current }} \
            -n ${{ env.NAMESPACE }} || true
      
      - name: Rollback on failure
        if: failure()
        run: |
          echo "Rolling back to ${{ steps.current-slot.outputs.current }} slot"
          
          kubectl patch virtualservice financial-services \
            -n ${{ env.NAMESPACE }} \
            --type merge \
            -p '{
              "spec": {
                "http": [{
                  "route": [
                    {"destination": {"host": "financial-services-${{ steps.current-slot.outputs.current }}"}, "weight": 100}
                  ]
                }]
              }
            }'
          
          helm uninstall financial-services-${{ steps.current-slot.outputs.new }} \
            -n ${{ env.NAMESPACE }} || true
      
      - name: Send deployment notification
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: |
            Deployment to production ${{ job.status }}
            Version: ${{ github.sha }}
            Slot: ${{ steps.current-slot.outputs.new }}
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

## 3. Frontend CI/CD Pipeline

```yaml
# .github/workflows/frontend-ci.yml
name: Frontend CI

on:
  push:
    branches:
      - main
      - develop
    paths:
      - 'frontend/**'
      - '.github/workflows/frontend-ci.yml'

env:
  NODE_VERSION: '18'
  AWS_REGION: us-east-1

jobs:
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest
    
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
        run: |
          cd frontend
          npm ci
      
      - name: Lint
        run: |
          cd frontend
          npm run lint
      
      - name: Run unit tests
        run: |
          cd frontend
          npm run test:ci
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: frontend/coverage/
      
      - name: Build application
        run: |
          cd frontend
          npm run build:prod
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: frontend-build
          path: frontend/dist/
  
  deploy-to-s3:
    name: Deploy to S3 and CloudFront
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: frontend-build
          path: dist/
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Deploy to S3
        run: |
          aws s3 sync dist/ s3://financial-services-frontend/ \
            --delete \
            --cache-control "public, max-age=31536000, immutable" \
            --exclude "index.html" \
            --exclude "*.map"
          
          aws s3 cp dist/index.html s3://financial-services-frontend/index.html \
            --cache-control "public, max-age=0, must-revalidate"
      
      - name: Invalidate CloudFront cache
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths "/*"
```

## 4. Infrastructure Pipeline

```yaml
# .github/workflows/infrastructure.yml
name: Infrastructure Deployment

on:
  push:
    branches:
      - main
    paths:
      - 'terraform/**'
      - '.github/workflows/infrastructure.yml'
  pull_request:
    branches:
      - main
    paths:
      - 'terraform/**'

env:
  AWS_REGION: us-east-1
  TF_VERSION: '1.6.0'

jobs:
  terraform-validate:
    name: Terraform Validate
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Format Check
        run: |
          cd terraform
          terraform fmt -check -recursive
      
      - name: Terraform Init
        run: |
          cd terraform
          terraform init -backend=false
      
      - name: Terraform Validate
        run: |
          cd terraform
          terraform validate
  
  terraform-security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: terraform-validate
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: terraform
          format: sarif
          soft_fail: false
      
      - name: Upload tfsec results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: tfsec.sarif
      
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: terraform
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif
      
      - name: Upload Checkov results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: checkov.sarif
  
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    needs: terraform-security-scan
    if: github.event_name == 'pull_request'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Init
        run: |
          cd terraform
          terraform init
      
      - name: Terraform Plan
        id: plan
        run: |
          cd terraform
          terraform plan -no-color -out=tfplan
        continue-on-error: true
      
      - name: Comment PR with plan
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const output = `#### Terraform Plan 📖
            
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            
            *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })
  
  terraform-apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    needs: terraform-security-scan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment:
      name: production-infrastructure
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Init
        run: |
          cd terraform
          terraform init
      
      - name: Terraform Apply
        run: |
          cd terraform
          terraform apply -auto-approve
```

## 5. Database Migration Workflow

```yaml
# .github/workflows/database-migration.yml
name: Database Migration

on:
  push:
    branches:
      - main
    paths:
      - 'database/migrations/**'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to run migrations'
        required: true
        type: choice
        options:
          - staging
          - production

jobs:
  migrate-staging:
    name: Migrate Staging Database
    runs-on: ubuntu-latest
    if: github.event_name == 'push' || github.event.inputs.environment == 'staging'
    environment: staging-database
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Get database credentials
        id: db-creds
        run: |
          SECRET=$(aws secretsmanager get-secret-value \
            --secret-id financial-services/database/credentials-staging \
            --query SecretString \
            --output text)
          
          echo "DB_HOST=$(echo $SECRET | jq -r .host)" >> $GITHUB_OUTPUT
          echo "DB_PORT=$(echo $SECRET | jq -r .port)" >> $GITHUB_OUTPUT
          echo "DB_NAME=$(echo $SECRET | jq -r .dbname)" >> $GITHUB_OUTPUT
          echo "DB_USER=$(echo $SECRET | jq -r .username)" >> $GITHUB_OUTPUT
          echo "::add-mask::$(echo $SECRET | jq -r .password)"
          echo "DB_PASSWORD=$(echo $SECRET | jq -r .password)" >> $GITHUB_OUTPUT
      
      - name: Run Flyway migrations
        run: |
          cd database
          mvn flyway:migrate \
            -Dflyway.url=jdbc:postgresql://${{ steps.db-creds.outputs.DB_HOST }}:${{ steps.db-creds.outputs.DB_PORT }}/${{ steps.db-creds.outputs.DB_NAME }} \
            -Dflyway.user=${{ steps.db-creds.outputs.DB_USER }} \
            -Dflyway.password=${{ steps.db-creds.outputs.DB_PASSWORD }} \
            -Dflyway.locations=filesystem:migrations
  
  migrate-production:
    name: Migrate Production Database
    runs-on: ubuntu-latest
    needs: migrate-staging
    if: github.event.inputs.environment == 'production'
    environment: production-database
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Create RDS snapshot
        run: |
          SNAPSHOT_ID="financial-db-pre-migration-$(date +%Y%m%d-%H%M%S)"
          aws rds create-db-snapshot \
            --db-instance-identifier financial-services-primary \
            --db-snapshot-identifier $SNAPSHOT_ID
          
          echo "Created snapshot: $SNAPSHOT_ID"
      
      - name: Get database credentials
        id: db-creds
        run: |
          SECRET=$(aws secretsmanager get-secret-value \
            --secret-id financial-services/database/credentials \
            --query SecretString \
            --output text)
          
          echo "DB_HOST=$(echo $SECRET | jq -r .host)" >> $GITHUB_OUTPUT
          echo "DB_PORT=$(echo $SECRET | jq -r .port)" >> $GITHUB_OUTPUT
          echo "DB_NAME=$(echo $SECRET | jq -r .dbname)" >> $GITHUB_OUTPUT
          echo "DB_USER=$(echo $SECRET | jq -r .username)" >> $GITHUB_OUTPUT
          echo "::add-mask::$(echo $SECRET | jq -r .password)"
          echo "DB_PASSWORD=$(echo $SECRET | jq -r .password)" >> $GITHUB_OUTPUT
      
      - name: Run Flyway migrations
        run: |
          cd database
          mvn flyway:migrate \
            -Dflyway.url=jdbc:postgresql://${{ steps.db-creds.outputs.DB_HOST }}:${{ steps.db-creds.outputs.DB_PORT }}/${{ steps.db-creds.outputs.DB_NAME }} \
            -Dflyway.user=${{ steps.db-creds.outputs.DB_USER }} \
            -Dflyway.password=${{ steps.db-creds.outputs.DB_PASSWORD }} \
            -Dflyway.locations=filesystem:migrations
```

## 6. Secrets Configuration

Required GitHub Secrets:
```yaml
# AWS Credentials
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY

# SonarQube
SONAR_TOKEN
SONAR_HOST_URL

# Snyk
SNYK_TOKEN

# CloudFront
CLOUDFRONT_DISTRIBUTION_ID

# Notifications
SLACK_WEBHOOK

# Database (if needed for local testing)
DB_PASSWORD
REDIS_PASSWORD
JWT_SECRET
```

## 7. Reusable Actions

### Setup AWS Action
```yaml
# .github/actions/setup-aws/action.yml
name: 'Setup AWS'
description: 'Configure AWS credentials and tools'

inputs:
  aws-region:
    description: 'AWS Region'
    required: true
    default: 'us-east-1'

runs:
  using: 'composite'
  steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ env.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ env.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ inputs.aws-region }}
    
    - name: Install AWS CLI
      shell: bash
      run: |
        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
        unzip awscliv2.zip
        sudo ./aws/install --update
    
    - name: Install kubectl
      shell: bash
      run: |
        curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
        chmod +x kubectl
        sudo mv kubectl /usr/local/bin/
```

This completes the GitHub Actions CI/CD pipeline documentation. Would you like me to continue with the complete Terraform deployment scripts and cost estimation for AWS?
