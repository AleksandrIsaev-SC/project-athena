
## 14e-security-view.md

```markdown
# Представление безопасности архитектуры

## Security-by-Design принципы

### 1. Модель угроз и рисков

#### 1.1. Идентифицированные угрозы
**Высокий риск:**
- Утечка персональных данных пользователей
- Неавторизованный доступ к биометрическим данным
- DDoS атаки на API endpoints
- Injection attacks через пользовательский ввод

**Средний риск:**
- Session hijacking
- Man-in-the-middle атаки
- Insufficient logging and monitoring
- Vulnerabilities in third-party dependencies

**Низкий риск:**
- Physical security breaches
- Social engineering attacks
- Zero-day vulnerabilities

### 2. Identity and Access Management (IAM)

#### 2.1. Аутентификация
**Multi-factor authentication (MFA)**
```yaml
authentication:
  primary_methods:
    - email_password: Для обычных пользователей
    - oauth2_providers: [google, apple, facebook]
    - biometric: Для мобильных приложений
  
  mfa:
    required: true для администраторов
    optional: true для пользователей
    methods: [totp, sms, authenticator_app]

**JWT токены**
```java
public class JwtConfig {
    @Value("${security.jwt.secret}")
    private String secretKey;
    
    @Value("${security.jwt.expiration}")
    private long expirationMs;
    
    public String generateToken(UserDetails user) {
        return Jwts.builder()
            .setSubject(user.getUsername())
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(SignatureAlgorithm.HS512, secretKey)
            .compact();
    }
}

#### 2.2. Авторизация
**Role-Based Access Control (RBAC)**
```yaml
roles:
  - user: Базовые права на чтение/запись своих данных
  - premium_user: Расширенные аналитические возможности
  - moderator: Модерация контента
  - admin: Полный доступ к системе
 
 **Attribute-Based Access Control (ABAC)**
```python
class WorkoutAccessPolicy:
    def can_view_workout(self, user, workout):
        return (
            workout.user_id == user.id or
            workout.privacy == 'public' or
            user.is_friend(workout.user_id) and workout.privacy == 'friends'
        )

### 3. Защита данных

#### 3.1. Шифрование
**Data at rest**
```yaml
encryption:
  databases:
    postgresql: AES-256 (RDS encryption)
    mongodb: AES-256 (at rest encryption)
    redis: AES-256 (transit and at rest)
  
  storage:
    s3: SSE-S3 (server-side encryption)
    ebs: EBS encryption

**Data in transit**
```yaml
tls:
  minimum_version: TLS 1.2
  preferred_version: TLS 1.3
  ciphers: 
    - ECDHE-RSA-AES128-GCM-SHA256
    - ECDHE-RSA-AES256-GCM-SHA384
  
  certificate_management: AWS ACM + automatic rotation

#### 3.2. Маскирование и анонимизация
**PII protection**
```sql
-- Маскирование email в логах
CREATE FUNCTION mask_email(email TEXT) RETURNS TEXT AS $$
BEGIN
    RETURN regexp_replace(email, '(.{2}).*@(.{2}).*', '\1***@\2***');
END;
$$ LANGUAGE plpgsql;

**Анонимизация аналитических данных**
```python
def anonymize_workout_data(workout):
    return {
        'id': hash_with_salt(workout.user_id),
        'sport_type': workout.sport_type,
        'metrics': workout.metrics,
        'location_region': workout.location.country_code,
        # Удалены все PII поля
    }

### 4. Сетевая безопасность

#### 4.1. Network segmentation
```yaml
network_security:
  vpc:
    public_subnets: Only for load balancers
    private_subnets: Application servers
    data_subnets: Databases (no internet access)
  
  security_groups:
    - api_gateway: Allow 443 from 0.0.0.0/0
    - application: Allow from api_gateway only
    - database: Allow from application only

#### 4.2. Web Application Firewall (WAF)
```yaml
waf_rules:
  - sql_injection: Block SQL injection attempts
  - xss: Block cross-site scripting
  - rate_based: Block IPs with >1000 requests/min
  - geo_block: Block known malicious regions
  - size_constraints: Block large payloads
 
### 5. Application Security

#### 5.1. Input validation и sanitization
```java
@Validated
public class WorkoutCreateRequest {
    @NotBlank
    @Size(min = 1, max = 50)
    private String sportType;
    
    @NotNull
    @Future
    private LocalDateTime startTime;
    
    @Valid
    private List<@Valid Metric> metrics;
}

// Sanitization для пользовательского контента
public String sanitizeUserContent(String content) {
    return Jsoup.clean(content, Safelist.basic());
}

#### 5.2. Security headers
```yaml
security_headers:
  - Content-Security-Policy: "default-src 'self'"
  - X-Content-Type-Options: nosniff
  - X-Frame-Options: DENY
  - Strict-Transport-Security: "max-age=31536000"
  - X-XSS-Protection: "1; mode=block"

### 6. API Security

#### 6.1. Rate limiting
```redis
-- Redis-based rate limiting
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('INCR', key)
if current == 1 then
    redis.call('EXPIRE', key, window)
end

return current <= limit

#### 6.2. API versioning и deprecation
```yaml
api_versioning:
  current: v1
  supported:
    - v1: Until 2025-01-01
    - v2: Beta, available for early adopters
  
  deprecation_policy:
    - notice_period: 6 months
    - sunset_period: 3 months

### 7. Mobile App Security

#### 7.1. Certificate pinning
```kotlin
class CertificatePinner {
    companion object {
        val pinning = CertificatePinner.Builder()
            .add("api.athena.fit", "sha256/AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA=")
            .build()
    }
}

#### 7.2. Secure storage
```swift
// iOS Keychain для чувствительных данных
func storeSecureData(_ data: Data, forKey key: String) {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: key,
        kSecValueData as String: data
    ]
    SecItemAdd(query as CFDictionary, nil)
}

### 8. Monitoring и Incident Response

#### 8.1. Security monitoring
```yaml
security_events:
  - failed_logins: Alert after 5 attempts
  - data_access: Alert on unusual patterns
  - configuration_changes: Alert on critical changes
  - new_deployments: Alert for security review
  
siem_integration:
  - aws_cloudtrail: For API calls logging
  - osquery: For host-level monitoring
  - waf_logs: For web traffic analysis

#### 8.2. Incident response plan
```yaml
incident_response:
  classification:
    - severity_1: Critical (response within 15 minutes)
    - severity_2: High (response within 1 hour)
    - severity_3: Medium (response within 4 hours)
    - severity_4: Low (response within 24 hours)
  
  communication:
    - internal: Slack channel #security-incidents
    - external: Legal team for data breaches
    - regulatory: GDPR authorities within 72 hours

### 9. Compliance и аудиты

#### 9.1. Regulatory complianc
```yaml
compliance_frameworks:
  - gdpr: 
      data_protection_officer: required
      user_rights: [access, rectification, erasure]
      data_processing_agreements: with all vendors
  
  - ccpa:
      opt_out_mechanism: required
      data_broker_registration: not required
  
  - soc2:
      annual_audit: required
      controls: [security, availability, confidentiality]

#### 9.2. Security audits
```yaml
audit_schedule:
  - penetration_testing: Quarterly
  - code_review: Every pull request
  - dependency_scanning: Daily
  - infrastructure_scanning: Weekly
  - compliance_audit: Annually
 
### 10. Third-party Security

#### 10.1. Vendor risk managementc
```yaml
third_party_assessments:
  - required: Security questionnaire
  - required: SOC 2 Type II report
  - required: Data processing agreement
  - optional: Penetration test report

#### 10.2. Supply chain security
```yaml
supply_chain_security:
  - code_signing: All artifacts must be signed
  - sbom: Software Bill of Materials required
  - vulnerability_scanning: All container images
  - provenance: Build provenance verification

### 11. Security Training и Awareness

#### 11.1. Developer training
```yaml
security_training:
  - mandatory: Secure coding practices
  - mandatory: OWASP Top 10 awareness
  - optional: Advanced threat modeling
  - recurring: Quarterly security updates

#### 11.2. Security champions program
```yaml
security_champions:
  - each_team: 1-2 security champions
  - responsibilities:
      - code_review_security_focus
      - security_tooling_advocacy
      - incident_response_participation
