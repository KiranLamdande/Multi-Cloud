# CI/CD Pipeline Architecture

## 1. Azure DevOps Setup

### 1.1 Project Structure
```
financial-services-app/
├── Repos/
│   ├── frontend-angular
│   ├── backend-microservices
│   ├── infrastructure-terraform
│   └── helm-charts
├── Pipelines/
│   ├── frontend-ci-cd.yml
│   ├── backend-ci-cd.yml
│   ├── infrastructure-pipeline.yml
│   └── release-pipeline.yml
├── Artifacts/
│   ├── Docker Images
│   ├── Helm Charts
│   └── Terraform Modules
└── Test Plans/
    ├── Unit Tests
    ├── Integration Tests
    └── Performance Tests
```

## 2. Backend CI Pipeline

```yaml
# backend-ci-cd.yml
trigger:
  branches:
    include:
      - main
      - develop
      - feature/*
  paths:
    include:
      - backend-microservices/*

stages:
  - stage: Build
    jobs:
      - job: CompileAndTest
        steps:
          - task: Maven@3
            displayName: 'Maven Build and Test'
            inputs:
              goals: 'clean verify'
              options: '-DskipITs=false'
              javaHomeOption: 'JDKVersion'
              jdkVersionOption: '17'
          
          - task: SonarQubePrepare@5
            displayName: 'Prepare SonarQube Analysis'
            inputs:
              SonarQube: 'SonarQube-Connection'
              scannerMode: 'Maven'
              extraProperties: |
                sonar.coverage.jacoco.xmlReportPaths=**/jacoco.xml
          
          - task: Maven@3
            displayName: 'Run SonarQube Analysis'
            inputs:
              goals: 'sonar:sonar'
          
          - task: PublishTestResults@2
            displayName: 'Publish Test Results'
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/TEST-*.xml'
              failTaskOnFailedTests: true
          
          - task: PublishCodeCoverageResults@1
            displayName: 'Publish Code Coverage'
            inputs:
              codeCoverageTool: 'JaCoCo'
              summaryFileLocation: '**/jacoco.xml'
              failIfCoverageEmpty: true
  
  - stage: SecurityScan
    dependsOn: Build
    jobs:
      - job: DependencyCheck
        steps:
          - task: dependency-check-build-task@6
            displayName: 'OWASP Dependency Check'
            inputs:
              projectName: 'Financial-Services'
              scanPath: '**/*.jar'
              format: 'HTML,JSON'
          
          - task: WhiteSource@21
            displayName: 'WhiteSource Security Scan'
            inputs:
              cwd: '$(System.DefaultWorkingDirectory)'
              projectName: 'Financial-Services'
          
          - task: SnykSecurityScan@1
            displayName: 'Snyk Security Scan'
            inputs:
              serviceConnectionEndpoint: 'Snyk-Connection'
              testType: 'app'
              severityThreshold: 'high'
              failOnIssues: true
  
  - stage: ContainerBuild
    dependsOn: SecurityScan
    jobs:
      - job: DockerBuild
        steps:
          - task: Docker@2
            displayName: 'Build Docker Image'
            inputs:
              command: 'build'
              repository: 'financial-services'
              dockerfile: '**/Dockerfile'
              tags: |
                $(Build.BuildId)
                latest
              arguments: '--build-arg VERSION=$(Build.BuildId)'
          
          - task: Trivy@1
            displayName: 'Trivy Container Scan'
            inputs:
              image: 'financial-services:$(Build.BuildId)'
              severities: 'CRITICAL,HIGH'
              exitCode: '1'
          
          - task: Docker@2
            displayName: 'Push to ACR'
            inputs:
              command: 'push'
              containerRegistry: 'ACR-Connection'
              repository: 'financial-services'
              tags: |
                $(Build.BuildId)
                latest
          
          - task: HelmDeploy@0
            displayName: 'Package Helm Chart'
            inputs:
              command: 'package'
              chartPath: 'helm-charts/financial-services'
              chartVersion: '$(Build.BuildId)'
              destination: '$(Build.ArtifactStagingDirectory)'
          
          - task: PublishBuildArtifacts@1
            displayName: 'Publish Artifacts'
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'helm-charts'
```

## 3. Frontend CI Pipeline

```yaml
# frontend-ci-cd.yml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - frontend-angular/*

stages:
  - stage: Build
    jobs:
      - job: BuildAngular
        steps:
          - task: NodeTool@0
            displayName: 'Install Node.js'
            inputs:
              versionSpec: '18.x'
          
          - task: Npm@1
            displayName: 'npm install'
            inputs:
              command: 'install'
              workingDir: 'frontend-angular'
          
          - task: Npm@1
            displayName: 'npm run lint'
            inputs:
              command: 'custom'
              customCommand: 'run lint'
              workingDir: 'frontend-angular'
          
          - task: Npm@1
            displayName: 'npm run test'
            inputs:
              command: 'custom'
              customCommand: 'run test:ci'
              workingDir: 'frontend-angular'
          
          - task: PublishTestResults@2
            displayName: 'Publish Test Results'
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/TESTS-*.xml'
          
          - task: PublishCodeCoverageResults@1
            displayName: 'Publish Code Coverage'
            inputs:
              codeCoverageTool: 'Cobertura'
              summaryFileLocation: '**/coverage/cobertura-coverage.xml'
          
          - task: Npm@1
            displayName: 'npm run build'
            inputs:
              command: 'custom'
              customCommand: 'run build:prod'
              workingDir: 'frontend-angular'
          
          - task: ArchiveFiles@2
            displayName: 'Archive Build Output'
            inputs:
              rootFolderOrFile: 'frontend-angular/dist'
              includeRootFolder: false
              archiveType: 'zip'
              archiveFile: '$(Build.ArtifactStagingDirectory)/frontend-$(Build.BuildId).zip'
          
          - task: PublishBuildArtifacts@1
            displayName: 'Publish Artifacts'
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)'
              ArtifactName: 'frontend'
  
  - stage: DeployToCDN
    dependsOn: Build
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployStatic
        environment: 'production-cdn'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: 'Deploy to Azure CDN'
                  inputs:
                    azureSubscription: 'Azure-Subscription'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az storage blob upload-batch \
                        --account-name financialcdn \
                        --destination '$web' \
                        --source $(Pipeline.Workspace)/frontend/frontend-$(Build.BuildId).zip \
                        --overwrite
                      
                      az cdn endpoint purge \
                        --resource-group financial-app-prod-rg \
                        --profile-name financial-cdn-profile \
                        --name financial-cdn-endpoint \
                        --content-paths '/*'
```

## 4. Release Pipeline with Blue-Green Deployment

```yaml
# release-pipeline.yml
trigger: none

resources:
  pipelines:
    - pipeline: backend-build
      source: backend-ci-cd
      trigger:
        branches:
          include:
            - main

variables:
  - group: production-secrets
  - name: namespace
    value: 'financial-services'

stages:
  - stage: DeployToStaging
    jobs:
      - deployment: DeployStaging
        environment: 'staging'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: KubernetesManifest@0
                  displayName: 'Create Namespace'
                  inputs:
                    action: 'deploy'
                    kubernetesServiceConnection: 'AKS-Staging'
                    namespace: '$(namespace)-staging'
                    manifests: |
                      apiVersion: v1
                      kind: Namespace
                      metadata:
                        name: $(namespace)-staging
                
                - task: HelmDeploy@0
                  displayName: 'Deploy with Helm'
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscription: 'Azure-Subscription'
                    azureResourceGroup: 'financial-app-staging-rg'
                    kubernetesCluster: 'aks-staging'
                    namespace: '$(namespace)-staging'
                    command: 'upgrade'
                    chartType: 'FilePath'
                    chartPath: '$(Pipeline.Workspace)/backend-build/helm-charts'
                    releaseName: 'financial-services'
                    install: true
                    waitForExecution: true
                    overrideValues: |
                      image.tag=$(resources.pipeline.backend-build.runID)
                      replicaCount=2
                      environment=staging
                      database.host=$(staging-db-host)
                      redis.host=$(staging-redis-host)
                
                - task: Kubernetes@1
                  displayName: 'Wait for Rollout'
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscription: 'Azure-Subscription'
                    azureResourceGroup: 'financial-app-staging-rg'
                    kubernetesCluster: 'aks-staging'
                    namespace: '$(namespace)-staging'
                    command: 'rollout'
                    arguments: 'status deployment/financial-services --timeout=5m'
  
  - stage: IntegrationTests
    dependsOn: DeployToStaging
    jobs:
      - job: RunIntegrationTests
        steps:
          - task: Maven@3
            displayName: 'Run Integration Tests'
            inputs:
              goals: 'verify'
              options: '-DskipUTs=true -Dtest.environment=staging -Dtest.url=https://staging-api.financialservices.com'
          
          - task: PublishTestResults@2
            displayName: 'Publish Test Results'
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: '**/TEST-*.xml'
              failTaskOnFailedTests: true
      
      - job: PerformanceTests
        steps:
          - task: JMeterInstaller@0
            displayName: 'Install JMeter'
            inputs:
              jmeterVersion: '5.5'
          
          - task: TaurusRunner@0
            displayName: 'Run Performance Tests'
            inputs:
              taurusConfig: 'performance-tests/taurus.yml'
              outputDir: '$(Build.ArtifactStagingDirectory)/performance'
          
          - task: PublishBuildArtifacts@1
            displayName: 'Publish Performance Results'
            inputs:
              PathtoPublish: '$(Build.ArtifactStagingDirectory)/performance'
              ArtifactName: 'performance-results'
  
  - stage: DeployToProduction
    dependsOn: IntegrationTests
    condition: succeeded()
    jobs:
      - deployment: BlueGreenDeployment
        environment: 'production'
        strategy:
          runOnce:
            preDeploy:
              steps:
                - task: Kubernetes@1
                  displayName: 'Get Current Deployment'
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscription: 'Azure-Subscription'
                    azureResourceGroup: 'financial-app-prod-rg'
                    kubernetesCluster: 'aks-production'
                    namespace: '$(namespace)'
                    command: 'get'
                    arguments: 'deployments -l app=financial-services,slot=blue -o json'
                    outputFormat: 'json'
                
                - task: PowerShell@2
                  displayName: 'Determine Deployment Slot'
                  inputs:
                    targetType: 'inline'
                    script: |
                      $currentSlot = "blue"
                      $newSlot = "green"
                      Write-Host "##vso[task.setvariable variable=currentSlot]$currentSlot"
                      Write-Host "##vso[task.setvariable variable=newSlot]$newSlot"
            
            deploy:
              steps:
                - task: HelmDeploy@0
                  displayName: 'Deploy to Green Slot'
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscription: 'Azure-Subscription'
                    azureResourceGroup: 'financial-app-prod-rg'
                    kubernetesCluster: 'aks-production'
                    namespace: '$(namespace)-$(newSlot)'
                    command: 'upgrade'
                    chartType: 'FilePath'
                    chartPath: '$(Pipeline.Workspace)/backend-build/helm-charts'
                    releaseName: 'financial-services-$(newSlot)'
                    install: true
                    waitForExecution: true
                    overrideValues: |
                      image.tag=$(resources.pipeline.backend-build.runID)
                      replicaCount=10
                      environment=production
                      slot=$(newSlot)
                      database.host=$(prod-db-host)
                      redis.host=$(prod-redis-host)
                      resources.requests.cpu=1000m
                      resources.requests.memory=2Gi
                      resources.limits.cpu=2000m
                      resources.limits.memory=4Gi
                
                - task: Kubernetes@1
                  displayName: 'Wait for Green Deployment'
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscription: 'Azure-Subscription'
                    azureResourceGroup: 'financial-app-prod-rg'
                    kubernetesCluster: 'aks-production'
                    namespace: '$(namespace)-$(newSlot)'
                    command: 'rollout'
                    arguments: 'status deployment/financial-services-$(newSlot) --timeout=10m'
                
                - task: Bash@3
                  displayName: 'Smoke Tests on Green'
                  inputs:
                    targetType: 'inline'
                    script: |
                      echo "Running smoke tests on green deployment"
                      GREEN_URL="http://financial-services-$(newSlot).$(namespace)-$(newSlot).svc.cluster.local"
                      
                      # Health check
                      curl -f $GREEN_URL/actuator/health || exit 1
                      
                      # Basic API test
                      curl -f $GREEN_URL/api/v1/health || exit 1
                      
                      echo "Smoke tests passed"
            
            routeTraffic:
              steps:
                - task: Kubernetes@1
                  displayName: 'Route 10% Traffic to Green'
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscription: 'Azure-Subscription'
                    azureResourceGroup: 'financial-app-prod-rg'
                    kubernetesCluster: 'aks-production'
                    namespace: '$(namespace)'
                    command: 'apply'
                    arguments: '-f $(Pipeline.Workspace)/k8s/service-split-10.yaml'
                
                - task: Bash@3
                  displayName: 'Monitor for 5 minutes'
                  inputs:
                    targetType: 'inline'
                    script: |
                      echo "Monitoring green deployment with 10% traffic"
                      sleep 300
                
                - task: AzureCLI@2
                  displayName: 'Check Error Rates'
                  inputs:
                    azureSubscription: 'Azure-Subscription'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      ERROR_COUNT=$(az monitor app-insights query \
                        --app financial-services-insights \
                        --analytics-query "exceptions | where timestamp > ago(5m) and cloud_RoleName == 'financial-services-$(newSlot)' | count" \
                        --query 'tables[0].rows[0][0]' -o tsv)
                      
                      if [ $ERROR_COUNT -gt 10 ]; then
                        echo "Error count too high: $ERROR_COUNT"
                        exit 1
                      fi
                
                - task: Kubernetes@1
                  displayName: 'Route 50% Traffic to Green'
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscription: 'Azure-Subscription'
                    azureResourceGroup: 'financial-app-prod-rg'
                    kubernetesCluster: 'aks-production'
                    namespace: '$(namespace)'
                    command: 'apply'
                    arguments: '-f $(Pipeline.Workspace)/k8s/service-split-50.yaml'
                
                - task: Bash@3
                  displayName: 'Monitor for 5 minutes'
                  inputs:
                    targetType: 'inline'
                    script: |
                      echo "Monitoring green deployment with 50% traffic"
                      sleep 300
                
                - task: Kubernetes@1
                  displayName: 'Route 100% Traffic to Green'
                  inputs:
                    connectionType: 'Azure Resource Manager'
                    azureSubscription: 'Azure-Subscription'
                    azureResourceGroup: 'financial-app-prod-rg'
                    kubernetesCluster: 'aks-production'
                    namespace: '$(namespace)'
                    command: 'apply'
                    arguments: '-f $(Pipeline.Workspace)/k8s/service-green.yaml'
                
                - task: Bash@3
                  displayName: 'Final Monitoring Period'
                  inputs:
                    targetType: 'inline'
                    script: |
                      echo "Final monitoring with 100% traffic on green"
                      sleep 600
            
            postRouteTraffic:
              steps:
                - task: AzureCLI@2
                  displayName: 'Validate Deployment Success'
                  inputs:
                    azureSubscription: 'Azure-Subscription'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      # Check error rates
                      ERROR_RATE=$(az monitor app-insights query \
                        --app financial-services-insights \
                        --analytics-query "requests | where timestamp > ago(10m) | summarize errorRate = todouble(countif(success == false)) / count() * 100" \
                        --query 'tables[0].rows[0][0]' -o tsv)
                      
                      echo "Error rate: $ERROR_RATE%"
                      
                      if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
                        echo "Error rate too high"
                        exit 1
                      fi
                      
                      # Check response times
                      P95_LATENCY=$(az monitor app-insights query \
                        --app financial-services-insights \
                        --analytics-query "requests | where timestamp > ago(10m) | summarize p95=percentile(duration, 95)" \
                        --query 'tables[0].rows[0][0]' -o tsv)
                      
                      echo "P95 latency: $P95_LATENCY ms"
                      
                      if (( $(echo "$P95_LATENCY > 2000" | bc -l) )); then
                        echo "Latency too high"
                        exit 1
                      fi
            
            on:
              failure:
                steps:
                  - task: Kubernetes@1
                    displayName: 'Rollback to Blue'
                    inputs:
                      connectionType: 'Azure Resource Manager'
                      azureSubscription: 'Azure-Subscription'
                      azureResourceGroup: 'financial-app-prod-rg'
                      kubernetesCluster: 'aks-production'
                      namespace: '$(namespace)'
                      command: 'apply'
                      arguments: '-f $(Pipeline.Workspace)/k8s/service-blue.yaml'
                  
                  - task: Kubernetes@1
                    displayName: 'Delete Green Deployment'
                    inputs:
                      connectionType: 'Azure Resource Manager'
                      azureSubscription: 'Azure-Subscription'
                      azureResourceGroup: 'financial-app-prod-rg'
                      kubernetesCluster: 'aks-production'
                      namespace: '$(namespace)-$(newSlot)'
                      command: 'delete'
                      arguments: 'namespace $(namespace)-$(newSlot)'
                  
                  - task: AzureCLI@2
                    displayName: 'Send Failure Notification'
                    inputs:
                      azureSubscription: 'Azure-Subscription'
                      scriptType: 'bash'
                      scriptLocation: 'inlineScript'
                      inlineScript: |
                        az monitor action-group trigger \
                          --resource-group financial-app-prod-rg \
                          --action-group-name deployment-failure-group
              
              success:
                steps:
                  - task: Kubernetes@1
                    displayName: 'Delete Blue Deployment'
                    inputs:
                      connectionType: 'Azure Resource Manager'
                      azureSubscription: 'Azure-Subscription'
                      azureResourceGroup: 'financial-app-prod-rg'
                      kubernetesCluster: 'aks-production'
                      namespace: '$(namespace)-$(currentSlot)'
                      command: 'delete'
                      arguments: 'namespace $(namespace)-$(currentSlot)'
                  
                  - task: Bash@3
                    displayName: 'Rename Green to Blue'
                    inputs:
                      targetType: 'inline'
                      script: |
                        echo "Green deployment successful, now becomes the new blue"
                        # Update labels to mark green as new blue
                        kubectl label namespace $(namespace)-$(newSlot) slot=blue --overwrite
                  
                  - task: AzureCLI@2
                    displayName: 'Send Success Notification'
                    inputs:
                      azureSubscription: 'Azure-Subscription'
                      scriptType: 'bash'
                      scriptLocation: 'inlineScript'
                      inlineScript: |
                        az monitor action-group trigger \
                          --resource-group financial-app-prod-rg \
                          --action-group-name deployment-success-group
```

## 5. Infrastructure Pipeline

```yaml
# infrastructure-pipeline.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure-terraform/*

pool:
  vmImage: 'ubuntu-latest'

variables:
  - group: terraform-backend
  - name: terraformVersion
    value: '1.6.0'

stages:
  - stage: Validate
    jobs:
      - job: TerraformValidate
        steps:
          - task: TerraformInstaller@0
            displayName: 'Install Terraform'
            inputs:
              terraformVersion: '$(terraformVersion)'
          
          - task: TerraformTaskV4@4
            displayName: 'Terraform Init'
            inputs:
              provider: 'azurerm'
              command: 'init'
              workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure-terraform'
              backendServiceArm: 'Azure-Subscription'
              backendAzureRmResourceGroupName: '$(tf-backend-rg)'
              backendAzureRmStorageAccountName: '$(tf-backend-sa)'
              backendAzureRmContainerName: 'tfstate'
              backendAzureRmKey: 'financial-app.tfstate'
          
          - task: TerraformTaskV4@4
            displayName: 'Terraform Format Check'
            inputs:
              provider: 'azurerm'
              command: 'custom'
              customCommand: 'fmt'
              commandOptions: '-check -recursive'
              workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure-terraform'
          
          - task: TerraformTaskV4@4
            displayName: 'Terraform Validate'
            inputs:
              provider: 'azurerm'
              command: 'validate'
              workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure-terraform'
          
          - task: TerraformTaskV4@4
            displayName: 'Terraform Plan'
            inputs:
              provider: 'azurerm'
              command: 'plan'
              workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure-terraform'
              environmentServiceNameAzureRM: 'Azure-Subscription'
              commandOptions: '-out=tfplan'
          
          - task: PublishBuildArtifacts@1
            displayName: 'Publish Terraform Plan'
            inputs:
              PathtoPublish: '$(System.DefaultWorkingDirectory)/infrastructure-terraform/tfplan'
              ArtifactName: 'terraform-plan'
  
  - stage: SecurityScan
    dependsOn: Validate
    jobs:
      - job: TerraformSecurity
        steps:
          - task: TerraformInstaller@0
            inputs:
              terraformVersion: '$(terraformVersion)'
          
          - task: Bash@3
            displayName: 'Install tfsec'
            inputs:
              targetType: 'inline'
              script: |
                wget -q https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64
                chmod +x tfsec-linux-amd64
                sudo mv tfsec-linux-amd64 /usr/local/bin/tfsec
          
          - task: Bash@3
            displayName: 'Run tfsec'
            inputs:
              targetType: 'inline'
              script: |
                tfsec $(System.DefaultWorkingDirectory)/infrastructure-terraform \
                  --format junit \
                  --out tfsec-results.xml
          
          - task: PublishTestResults@2
            displayName: 'Publish tfsec Results'
            inputs:
              testResultsFormat: 'JUnit'
              testResultsFiles: 'tfsec-results.xml'
  
  - stage: ApplyInfrastructure
    dependsOn: SecurityScan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: ApplyTerraform
        environment: 'infrastructure-production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: TerraformInstaller@0
                  inputs:
                    terraformVersion: '$(terraformVersion)'
                
                - task: TerraformTaskV4@4
                  displayName: 'Terraform Init'
                  inputs:
                    provider: 'azurerm'
                    command: 'init'
                    workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure-terraform'
                    backendServiceArm: 'Azure-Subscription'
                    backendAzureRmResourceGroupName: '$(tf-backend-rg)'
                    backendAzureRmStorageAccountName: '$(tf-backend-sa)'
                    backendAzureRmContainerName: 'tfstate'
                    backendAzureRmKey: 'financial-app.tfstate'
                
                - task: TerraformTaskV4@4
                  displayName: 'Terraform Apply'
                  inputs:
                    provider: 'azurerm'
                    command: 'apply'
                    workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure-terraform'
                    environmentServiceNameAzureRM: 'Azure-Subscription'
                    commandOptions: '-auto-approve'
```

## 6. Database Migration Pipeline

```yaml
# database-migration-pipeline.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - database/migrations/*

stages:
  - stage: ValidateMigrations
    jobs:
      - job: FlywayValidate
        steps:
          - task: FlywayTask@0
            displayName: 'Flyway Validate'
            inputs:
              flywayCommand: 'validate'
              targetType: 'inline'
              locations: 'filesystem:database/migrations'
              url: '$(staging-db-url)'
              user: '$(staging-db-user)'
              password: '$(staging-db-password)'
  
  - stage: ApplyToStaging
    dependsOn: ValidateMigrations
    jobs:
      - deployment: MigrateStaging
        environment: 'staging-database'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: FlywayTask@0
                  displayName: 'Flyway Migrate Staging'
                  inputs:
                    flywayCommand: 'migrate'
                    targetType: 'inline'
                    locations: 'filesystem:database/migrations'
                    url: '$(staging-db-url)'
                    user: '$(staging-db-user)'
                    password: '$(staging-db-password)'
                    baselineOnMigrate: true
  
  - stage: ApplyToProduction
    dependsOn: ApplyToStaging
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: MigrateProduction
        environment: 'production-database'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  displayName: 'Backup Database'
                  inputs:
                    azureSubscription: 'Azure-Subscription'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az postgres flexible-server backup create \
                        --resource-group financial-app-prod-rg \
                        --name financial-db-primary \
                        --backup-name pre-migration-$(Build.BuildId)
                
                - task: FlywayTask@0
                  displayName: 'Flyway Migrate Production'
                  inputs:
                    flywayCommand: 'migrate'
                    targetType: 'inline'
                    locations: 'filesystem:database/migrations'
                    url: '$(prod-db-url)'
                    user: '$(prod-db-user)'
                    password: '$(prod-db-password)'
                    baselineOnMigrate: true
                    outOfOrder: false
```

## 7. GitOps with ArgoCD (Alternative Approach)

```yaml
# argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: financial-services
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: financial-services
  
  source:
    repoURL: https://dev.azure.com/org/financial-services/_git/helm-charts
    targetRevision: HEAD
    path: financial-services
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "latest"
        - name: replicaCount
          value: "10"
  
  destination:
    server: https://kubernetes.default.svc
    namespace: financial-services
  
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  
  revisionHistoryLimit: 10
  
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas

---
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: financial-services
  namespace: argocd
spec:
  description: Financial Services Application
  
  sourceRepos:
    - https://dev.azure.com/org/financial-services/_git/*
  
  destinations:
    - namespace: financial-services*
      server: https://kubernetes.default.svc
  
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
  
  roles:
    - name: developer
      description: Developer role
      policies:
        - p, proj:financial-services:developer, applications, get, financial-services/*, allow
        - p, proj:financial-services:developer, applications, sync, financial-services/*, allow
      groups:
        - financial-services-developers
    
    - name: admin
      description: Admin role
      policies:
        - p, proj:financial-services:admin, applications, *, financial-services/*, allow
      groups:
        - financial-services-admins
```

## 8. Deployment Approval Gates

```yaml
# approval-gates.yml
stages:
  - stage: ProductionApproval
    dependsOn: IntegrationTests
    jobs:
      - job: waitForValidation
        displayName: 'Wait for Manual Approval'
        pool: server
        timeoutInMinutes: 4320 # 3 days
        steps:
          - task: ManualValidation@0
            timeoutInMinutes: 1440 # 1 day
            inputs:
              notifyUsers: |
                release-managers@company.com
                devops-team@company.com
              instructions: 'Please review the staging deployment and approve for production release'
              onTimeout: 'reject'
```

## 9. Rollback Procedures

```bash
#!/bin/bash
# rollback-deployment.sh

NAMESPACE="financial-services"
PREVIOUS_VERSION=$1

if [ -z "$PREVIOUS_VERSION" ]; then
    echo "Usage: ./rollback-deployment.sh <previous-version>"
    exit 1
fi

echo "Rolling back to version: $PREVIOUS_VERSION"

# Rollback using Helm
helm rollback financial-services $PREVIOUS_VERSION \
    --namespace $NAMESPACE \
    --wait \
    --timeout 10m

# Verify rollback
kubectl rollout status deployment/financial-services \
    --namespace $NAMESPACE \
    --timeout=10m

# Check pod health
kubectl get pods -n $NAMESPACE -l app=financial-services

echo "Rollback completed successfully"
```

## 10. Continuous Monitoring Integration

```yaml
# monitoring-integration.yml
- task: AzureCLI@2
  displayName: 'Create Deployment Annotation'
  inputs:
    azureSubscription: 'Azure-Subscription'
    scriptType: 'bash'
    scriptLocation: 'inlineScript'
    inlineScript: |
      az monitor app-insights component create-annotation \
        --resource-group financial-app-prod-rg \
        --app financial-services-insights \
        --annotation-name "Deployment $(Build.BuildId)" \
        --category "Deployment" \
        --properties "version=$(Build.BuildId)" "environment=production"