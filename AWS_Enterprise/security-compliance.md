# Security Architecture and PCI-DSS Compliance

## 1. PCI-DSS Requirements Mapping

### Requirement 1: Install and maintain a firewall configuration

**Implementation:**
- Azure Firewall for network-level protection
- Network Security Groups (NSGs) for subnet-level filtering
- Application Gateway with WAF for application-level protection
- Azure Front Door with DDoS protection

**Controls:**
```hcl
# Network Security Group for AKS
resource "azurerm_network_security_group" "aks" {
  name                = "aks-nsg"
  location            = var.location
  resource_group_name = var.resource_group_name
  
  # Deny all inbound by default
  security_rule {
    name                       = "DenyAllInbound"
    priority                   = 4096
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
  
  # Allow only from Application Gateway
  security_rule {
    name                       = "AllowAppGateway"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_ranges    = ["80", "443"]
    source_address_prefix      = "10.0.16.0/24"
    destination_address_prefix = "*"
  }
}
```

### Requirement 2: Do not use vendor-supplied defaults

**Implementation:**
- Custom passwords for all services
- Disabled default admin accounts
- Azure Policy enforcement
- Configuration management via Terraform

**Controls:**
```yaml
# Azure Policy - Enforce custom passwords
apiVersion: policy.azure.com/v1
kind: PolicyDefinition
metadata:
  name: enforce-strong-passwords
spec:
  displayName: "Enforce Strong Password Policy"
  policyRule:
    if:
      allOf:
        - field: "type"
          equals: "Microsoft.DBforPostgreSQL/flexibleServers"
    then:
      effect: "audit"
      details:
        type: "Microsoft.DBforPostgreSQL/flexibleServers/configurations"
        name: "password_encryption"
        existenceCondition:
          field: "Microsoft.DBforPostgreSQL/flexibleServers/configurations/value"
          equals: "scram-sha-256"
```

### Requirement 3: Protect stored cardholder data

**Implementation:**
- Encryption at rest using Azure-managed or customer-managed keys
- Tokenization for sensitive data
- Data masking for non-production environments
- Secure key management with Azure Key Vault

**Controls:**
```java
// Data Encryption Service
@Service
public class EncryptionService {
    
    @Autowired
    private SecretClient keyVaultClient;
    
    public String encryptCardNumber(String cardNumber) {
        // Tokenize card number
        String token = generateToken();
        
        // Store encrypted mapping in secure vault
        KeyVaultSecret secret = new KeyVaultSecret(token, cardNumber);
        keyVaultClient.setSecret(secret);
        
        return token;
    }
    
    public String decryptCardNumber(String token) {
        KeyVaultSecret secret = keyVaultClient.getSecret(token);
        return secret.getValue();
    }
    
    private String generateToken() {
        return "TKN-" + UUID.randomUUID().toString();
    }
}

// Database Column Encryption
@Entity
@Table(name = "payment_methods")
public class PaymentMethod {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Convert(converter = CardNumberConverter.class)
    @Column(name = "card_number_encrypted")
    private String cardNumber;
    
    @Convert(converter = CVVConverter.class)
    @Column(name = "cvv_encrypted")
    private String cvv;
}

@Converter
public class CardNumberConverter implements AttributeConverter<String, String> {
    
    @Autowired
    private EncryptionService encryptionService;
    
    @Override
    public String convertToDatabaseColumn(String attribute) {
        return encryptionService.encryptCardNumber(attribute);
    }
    
    @Override
    public String convertToEntityAttribute(String dbData) {
        return encryptionService.decryptCardNumber(dbData);
    }
}
```

### Requirement 4: Encrypt transmission of cardholder data

**Implementation:**
- TLS 1.3 for all external communications
- Mutual TLS (mTLS) between microservices
- Certificate management with Azure Key Vault
- Cipher suite restrictions

**Controls:**
```yaml
# Istio mTLS Configuration
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: financial-services
spec:
  mtls:
    mode: STRICT

---
# Ingress TLS Configuration
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: financial-services-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "TLS_AES_256_GCM_SHA384:TLS_AES_128_GCM_SHA256"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
spec:
  tls:
    - hosts:
        - api.financialservices.com
      secretName: tls-certificate
  rules:
    - host: api.financialservices.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: financial-services
                port:
                  number: 443
```

### Requirement 5: Protect all systems against malware

**Implementation:**
- Azure Defender for Containers
- Container image scanning with Trivy
- Runtime protection with Falco
- Regular vulnerability assessments

**Controls:**
```yaml
# Trivy Scan in CI/CD
- task: Trivy@1
  displayName: 'Scan Container Image'
  inputs:
    image: 'financial-services:$(Build.BuildId)'
    severities: 'CRITICAL,HIGH'
    exitCode: '1'
    ignoreUnfixed: false

# Falco Runtime Security
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-config
  namespace: falco
data:
  falco.yaml: |
    rules_file:
      - /etc/falco/falco_rules.yaml
      - /etc/falco/k8s_audit_rules.yaml
      - /etc/falco/custom_rules.yaml
    
    json_output: true
    json_include_output_property: true
    
    priority: warning
    
    outputs:
      rate: 1
      max_burst: 1000
    
    syslog_output:
      enabled: true
    
    file_output:
      enabled: true
      keep_alive: false
      filename: /var/log/falco/events.txt
```

### Requirement 6: Develop and maintain secure systems

**Implementation:**
- Secure SDLC with security gates
- Static code analysis (SonarQube)
- Dependency scanning (WhiteSource, Snyk)
- Security testing in CI/CD

**Controls:**
```yaml
# SonarQube Quality Gate
- task: SonarQubePrepare@5
  inputs:
    SonarQube: 'SonarQube-Connection'
    scannerMode: 'Maven'
    extraProperties: |
      sonar.coverage.jacoco.xmlReportPaths=**/jacoco.xml
      sonar.qualitygate.wait=true
      sonar.qualitygate.timeout=300

# OWASP Dependency Check
- task: dependency-check-build-task@6
  inputs:
    projectName: 'Financial-Services'
    scanPath: '**/*.jar'
    format: 'HTML,JSON'
    failOnCVSS: '7'
```

### Requirement 7: Restrict access to cardholder data

**Implementation:**
- Role-Based Access Control (RBAC)
- Least privilege principle
- Azure AD integration
- Just-in-time access

**Controls:**
```java
// Spring Security Configuration
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/api/public/**").permitAll()
                .antMatchers("/api/customer/**").hasRole("CUSTOMER")
                .antMatchers("/api/teller/**").hasRole("TELLER")
                .antMatchers("/api/manager/**").hasRole("MANAGER")
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .antMatchers("/api/audit/**").hasRole("AUDITOR")
                .anyRequest().authenticated()
            .and()
            .oauth2ResourceServer()
                .jwt()
                .jwtAuthenticationConverter(jwtAuthenticationConverter());
    }
    
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = 
            new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        
        JwtAuthenticationConverter jwtAuthenticationConverter = 
            new JwtAuthenticationConverter();
        jwtAuthenticationConverter.setJwtGrantedAuthoritiesConverter(
            grantedAuthoritiesConverter);
        
        return jwtAuthenticationConverter;
    }
}

// Method-level Security
@Service
public class AccountService {
    
    @PreAuthorize("hasRole('CUSTOMER') and #userId == authentication.principal.userId")
    public Account getAccount(Long userId, Long accountId) {
        return accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
    }
    
    @PreAuthorize("hasRole('TELLER')")
    public Account createAccount(AccountRequest request) {
        Account account = new Account();
        // Create account logic
        return accountRepository.save(account);
    }
    
    @PreAuthorize("hasRole('MANAGER')")
    public void approveTransaction(Long transactionId) {
        // Approval logic
    }
}
```

### Requirement 8: Identify and authenticate access

**Implementation:**
- Multi-factor authentication (MFA)
- Strong password policies
- Azure AD integration
- Session management

**Controls:**
```java
// MFA Configuration
@Configuration
public class MFAConfig {
    
    @Bean
    public AuthenticationProvider mfaAuthenticationProvider() {
        return new MFAAuthenticationProvider();
    }
}

public class MFAAuthenticationProvider implements AuthenticationProvider {
    
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Autowired
    private TOTPService totpService;
    
    @Override
    public Authentication authenticate(Authentication authentication) 
            throws AuthenticationException {
        
        String username = authentication.getName();
        String password = authentication.getCredentials().toString();
        String totpCode = (String) authentication.getDetails();
        
        UserDetails user = userDetailsService.loadUserByUsername(username);
        
        // Validate password
        if (!passwordEncoder.matches(password, user.getPassword())) {
            throw new BadCredentialsException("Invalid credentials");
        }
        
        // Validate TOTP
        if (!totpService.validateTOTP(username, totpCode)) {
            throw new BadCredentialsException("Invalid MFA code");
        }
        
        return new UsernamePasswordAuthenticationToken(
            user, password, user.getAuthorities());
    }
    
    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class
            .isAssignableFrom(authentication);
    }
}

// Password Policy
@Component
public class PasswordPolicyValidator {
    
    private static final int MIN_LENGTH = 12;
    private static final int MAX_AGE_DAYS = 90;
    private static final int HISTORY_COUNT = 12;
    
    public void validatePassword(String password, String username) {
        if (password.length() < MIN_LENGTH) {
            throw new WeakPasswordException("Password must be at least 12 characters");
        }
        
        if (!password.matches(".*[A-Z].*")) {
            throw new WeakPasswordException("Password must contain uppercase letter");
        }
        
        if (!password.matches(".*[a-z].*")) {
            throw new WeakPasswordException("Password must contain lowercase letter");
        }
        
        if (!password.matches(".*[0-9].*")) {
            throw new WeakPasswordException("Password must contain number");
        }
        
        if (!password.matches(".*[!@#$%^&*].*")) {
            throw new WeakPasswordException("Password must contain special character");
        }
        
        if (password.toLowerCase().contains(username.toLowerCase())) {
            throw new WeakPasswordException("Password cannot contain username");
        }
    }
}
```

### Requirement 9: Restrict physical access to cardholder data

**Implementation:**
- Azure datacenter physical security (Microsoft-managed)
- Compliance certifications (SOC 2, ISO 27001)
- Regular audits

**Documentation:**
- Azure datacenter security documentation
- Compliance reports from Microsoft
- Third-party audit reports

### Requirement 10: Track and monitor all access

**Implementation:**
- Comprehensive logging with Azure Monitor
- Audit trails for all operations
- Immutable log storage
- Log retention policies

**Controls:**
```java
// Audit Logging Aspect
@Aspect
@Component
public class AuditLoggingAspect {
    
    @Autowired
    private AuditLogRepository auditLogRepository;
    
    @Autowired
    private TelemetryClient telemetryClient;
    
    @Around("@annotation(Audited)")
    public Object logAuditEvent(ProceedingJoinPoint joinPoint) throws Throwable {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        String username = auth != null ? auth.getName() : "anonymous";
        
        String methodName = joinPoint.getSignature().getName();
        String className = joinPoint.getTarget().getClass().getSimpleName();
        Object[] args = joinPoint.getArgs();
        
        AuditLog auditLog = new AuditLog();
        auditLog.setUsername(username);
        auditLog.setAction(className + "." + methodName);
        auditLog.setTimestamp(Instant.now());
        auditLog.setIpAddress(getClientIpAddress());
        auditLog.setParameters(serializeParameters(args));
        
        Object result = null;
        try {
            result = joinPoint.proceed();
            auditLog.setStatus("SUCCESS");
            return result;
        } catch (Exception e) {
            auditLog.setStatus("FAILURE");
            auditLog.setErrorMessage(e.getMessage());
            throw e;
        } finally {
            auditLogRepository.save(auditLog);
            
            // Send to Application Insights
            Map<String, String> properties = new HashMap<>();
            properties.put("username", username);
            properties.put("action", auditLog.getAction());
            properties.put("status", auditLog.getStatus());
            
            telemetryClient.trackEvent("AuditEvent", properties, null);
        }
    }
}

// Usage
@Service
public class TransactionService {
    
    @Audited
    @Transactional
    public Transaction processTransaction(TransactionRequest request) {
        // Transaction processing logic
        return transaction;
    }
}
```

### Requirement 11: Regularly test security systems

**Implementation:**
- Quarterly vulnerability scans
- Annual penetration testing
- Automated security testing in CI/CD
- Bug bounty program

**Controls:**
```yaml
# Scheduled Security Scan Pipeline
schedules:
  - cron: "0 2 * * 0"  # Every Sunday at 2 AM
    displayName: Weekly Security Scan
    branches:
      include:
        - main
    always: true

stages:
  - stage: SecurityScan
    jobs:
      - job: VulnerabilityScan
        steps:
          - task: Nessus@1
            inputs:
              scanType: 'full'
              targets: '$(production-endpoints)'
          
          - task: OWASP-ZAP@1
            inputs:
              scanType: 'full'
              target: 'https://api.financialservices.com'
          
          - task: PublishSecurityReport@1
            inputs:
              reportPath: '$(Build.ArtifactStagingDirectory)/security-report.html'
```

### Requirement 12: Maintain an information security policy

**Implementation:**
- Documented security policies
- Regular security training
- Incident response procedures
- Compliance documentation

## 2. Data Classification

| Classification | Examples | Encryption | Access Control | Retention |
|----------------|----------|------------|----------------|-----------|
| Public | Marketing materials | Not required | Public | 1 year |
| Internal | Internal docs | At rest | Authenticated users | 3 years |
| Confidential | Customer PII | At rest + transit | Role-based | 7 years |
| Restricted | Card data, passwords | At rest + transit + tokenization | Strict RBAC + MFA | 7 years |

## 3. Secrets Management

```java
// Azure Key Vault Integration
@Configuration
public class KeyVaultConfig {
    
    @Bean
    public SecretClient secretClient() {
        String keyVaultUrl = "https://financial-kv.vault.azure.net/";
        return new SecretClientBuilder()
            .vaultUrl(keyVaultUrl)
            .credential(new DefaultAzureCredentialBuilder().build())
            .buildClient();
    }
}

@Service
public class SecretsService {
    
    @Autowired
    private SecretClient secretClient;
    
    @Cacheable("secrets")
    public String getSecret(String secretName) {
        KeyVaultSecret secret = secretClient.getSecret(secretName);
        return secret.getValue();
    }
    
    public void rotateSecret(String secretName, String newValue) {
        secretClient.setSecret(secretName, newValue);
        // Trigger application restart or reload
    }
}

// Kubernetes Secret Management
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault
  namespace: financial-services
spec:
  provider: azure
  parameters:
    usePodIdentity: "true"
    keyvaultName: "financial-kv"
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: redis-password
          objectType: secret
          objectVersion: ""
        - |
          objectName: jwt-secret
          objectType: secret
          objectVersion: ""
  secretObjects:
    - secretName: app-secrets
      type: Opaque
      data:
        - objectName: db-password
          key: database-password
        - objectName: redis-password
          key: redis-password
        - objectName: jwt-secret
          key: jwt-secret
```

## 4. Network Security

```hcl
# Azure Firewall
resource "azurerm_firewall" "main" {
  name                = "financial-app-firewall"
  location            = var.location
  resource_group_name = var.resource_group_name
  sku_name            = "AZFW_VNet"
  sku_tier            = "Premium"
  
  ip_configuration {
    name                 = "configuration"
    subnet_id            = azurerm_subnet.firewall.id
    public_ip_address_id = azurerm_public_ip.firewall.id
  }
  
  threat_intel_mode = "Alert"
}

# Firewall Rules
resource "azurerm_firewall_application_rule_collection" "main" {
  name                = "app-rules"
  azure_firewall_name = azurerm_firewall.main.name
  resource_group_name = var.resource_group_name
  priority            = 100
  action              = "Allow"
  
  rule {
    name = "allow-azure-services"
    source_addresses = ["10.0.0.0/16"]
    target_fqdns = [
      "*.azure.com",
      "*.microsoft.com",
      "*.windows.net"
    ]
    protocol {
      port = "443"
      type = "Https"
    }
  }
}

# DDoS Protection
resource "azurerm_network_ddos_protection_plan" "main" {
  name                = "financial-app-ddos"
  location            = var.location
  resource_group_name = var.resource_group_name
}
```

## 5. Compliance Monitoring

```yaml
# Azure Policy for Compliance
apiVersion: policy.azure.com/v1
kind: PolicyAssignment
metadata:
  name: pci-dss-compliance
spec:
  displayName: "PCI-DSS Compliance Policy"
  policyDefinitionId: "/providers/Microsoft.Authorization/policySetDefinitions/pci-dss-3.2.1"
  scope: "/subscriptions/{subscription-id}/resourceGroups/financial-app-prod-rg"
  parameters:
    effect:
      value: "Audit"
  enforcementMode: "Default"
```

## 6. Incident Response Plan

### Detection
- Automated alerts from Azure Security Center
- SIEM integration with Azure Sentinel
- Anomaly detection with Application Insights

### Response
1. Isolate affected systems
2. Preserve evidence
3. Contain the incident
4. Eradicate the threat
5. Recover systems
6. Post-incident review

### Communication
- Internal: Security team, management, legal
- External: Customers, regulators, law enforcement (if required)
- Timeline: Within 72 hours for data breaches (GDPR)

## 7. Compliance Certifications

- **PCI-DSS Level 1**: Annual assessment by QSA
- **SOC 2 Type II**: Annual audit
- **ISO 27001**: Certification and annual surveillance
- **GDPR**: Data protection compliance
- **SOX**: Financial controls compliance