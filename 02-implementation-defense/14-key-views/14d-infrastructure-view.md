
## 14d-infrastructure-view.md

```markdown
# Инфраструктурное представление архитектуры

## Обзор инфраструктурных компонентов

### 1. Cloud Infrastructure Strategy

#### 1.1. Multi-Cloud архитектура
**AWS (Основная платформа):**
- **Compute:** EC2, ECS, Lambda
- **Database:** RDS PostgreSQL, ElastiCache Redis
- **Storage:** S3, EBS
- **Networking:** VPC, ALB, Route53

**GCP (Аналитика и ML):**
- **Big Data:** BigQuery, Dataflow
- **ML Services:** AI Platform, Vertex AI
- **Storage:** Cloud Storage
- **Analytics:** Looker, Data Studio

#### 1.2. Региональное распределение
```yaml
regions:
  primary:
    - us-east-1 (Virginia): Главный регион
    - eu-west-1 (Ireland): Европейский регион
    - ap-southeast-1 (Singapore): Азиатский регион
  
  disaster_recovery:
    - us-west-2 (Oregon): DR сайт
    - eu-central-1 (Frankfurt): Европейский DR

### 2. Container Orchestration

#### 2.1. Kubernetes Cluster Architecture
**EKS (Elastic Kubernetes Service) конфигурация**
```yaml
# production-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: project-athena-prod
  region: us-east-1
  version: "1.28"

nodeGroups:
  - name: main-nodes
    instanceType: m5.2xlarge
    desiredCapacity: 10
    minSize: 3
    maxSize: 50
    volumeSize: 100
    labels:
      role: worker
    
  - name: spot-nodes
    instanceType: m5.2xlarge
    desiredCapacity: 5
    minSize: 0
    maxSize: 20
    spot: true
    labels:
      role: worker
      spot: "true"

#### 2.2. Namespace структура
```yaml
namespaces:
  - production: Продакшен сервисы
  - staging: Предпродакшен среда
  - monitoring: Мониторинг и логи
  - ingress: Внешние ingress контроллеры
  - database: Stateful сервисы БД

### 3. Сетевая инфраструктура

#### 3.1. VPC Design
```yaml
vpc:
  cidr: 10.0.0.0/16
  subnets:
    - public:
        - us-east-1a: 10.0.1.0/24  # Для load balancers
        - us-east-1b: 10.0.2.0/24
    - private:
        - us-east-1a: 10.0.10.0/24 # Для application servers
        - us-east-1b: 10.0.11.0/24
    - data:
        - us-east-1a: 10.0.20.0/24 # Для databases
        - us-east-1b: 10.0.21.0/24

#### 3.2. Load Balancing архитектура
```yaml
load_balancers:
  external:
    - name: api-gateway-alb
      scheme: internet-facing
      listeners:
        - port: 443 (HTTPS)
        - port: 80 (HTTP redirect)
      target_groups:
        - api-gateway:3000
        
  internal:
    - name: internal-services-nlb
      scheme: internal
      listeners:
        - port: 9092 (Kafka)
        - port: 5432 (PostgreSQL)

### 3. Сетевая инфраструктура

#### 3.1. VPC Design
```yaml
load_balancers:
  external:
    - name: api-gateway-alb
      scheme: internet-facing
      listeners:
        - port: 443 (HTTPS)
        - port: 80 (HTTP redirect)
      target_groups:
        - api-gateway:3000
        
  internal:
    - name: internal-services-nlb
      scheme: internal
      listeners:
        - port: 9092 (Kafka)
        - port: 5432 (PostgreSQL)

### 4. Базы данных и хранилища

#### 4.1. Database Cluster топологи
**PostgreSQL (RDS Aurora)**
```yaml
database:
  cluster:
    identifier: athena-main-db
    engine: aurora-postgresql
    version: "15.3"
    instances:
      - writer: db.r5.4xlarge
      - reader-1: db.r5.2xlarge
      - reader-2: db.r5.2xlarge
    storage:
      allocated: 1000  # GB
      iops: 10000
    backup:
      retention: 35 days
      window: 02:00-04:00 UTC

**MongoDB (Atlas Cluster)**
```yaml
mongodb:
  cluster:
    name: athena-documents
    tier: M40
    sharding:
      enabled: true
      shards: 3
    regions:
      - us-east-1 (Primary)
      - eu-west-1 (Secondary)
      - ap-southeast-1 (Secondary)

#### 4.2. Кэширующая инфраструктура
**Redis Cluster (ElastiCache)**
```yaml
redis:
  cluster:
    engine: redis
    version: "7.0"
    node_type: cache.r6g.2xlarge
    num_nodes: 6  # 3 primary, 3 replica
    parameters:
      maxmemory-policy: allkeys-lru
      timeout: 300

### 5. Message Brokers и очереди

#### 5.1. Kafka Cluster архитектура
**MSK (Managed Streaming for Kafka)**
```yaml
kafka:
  cluster:
    name: athena-events
    kafka_version: "3.5.1"
    broker_nodes:
      - type: kafka.m5.2xlarge
        count: 6
        storage: 1000 GB
    configuration:
      auto.create.topics.enable: false
      num.partitions: 50
      default.replication.factor: 3

#### 5.2. Dead Letter Queues
```yaml
dlq_queues:
  - workout_events_dlq:
      retention: 7 days
      alarms: true
  - social_events_dlq:
      retention: 3 days
      alarms: true

### 6. Мониторинг и observability

#### 6.1. Stack мониторинга
**Prometheus + Grafana**
```yaml
monitoring:
  prometheus:
    storage: 500 GB
    retention: 30 days
    scrape_interval: 15s
    
  grafana:
    dashboards:
      - application_metrics
      - business_metrics
      - infrastructure_health
      
  alerts:
    channels:
      - slack: #monitoring-alerts
      - pagerduty: critical-alerts

#### 6.1. Stack мониторинга
**ELK Stack + Jaeger**
```yaml
logging:
  elasticsearch:
    storage: 1 TB
    retention: 90 days
    
  jaeger:
    storage: 200 GB
    retention: 7 days

### 7. CI/CD Pipeline инфраструктура

#### 7.1. Build Infrastructure
**GitHub Actions runners**
```yaml
ci_cd:
  runners:
    - size: large (8CPU, 16GB)
      count: 10
      os: ubuntu-22.04
      
  environments:
    - development: auto-deploy
    - staging: manual-approval
    - production: manual-approval + tests

#### 7.2. Deployment стратегии
```yaml
deployment:
  strategies:
    - blue-green: Для критических сервисов
    - canary: Для пользовательских features
    - rolling: Для остальных сервисов
  
  rollback:
    automatic: true
    timeout: 15 minutes

### 8. Security Infrastructure

#### 8.1. Network Security
```yaml
security:
  network:
    - web_application_firewall: AWS WAF
    - ddos_protection: AWS Shield
    - network_acls: VPC ACLs
    - security_groups: Instance level
    
  access:
    - vpn: For administrative access
    - bastion_hosts: Jump servers
    - iam_roles: Least privilege principle

#### 8.2. Certificate Management
**Let's Encrypt + AWS Certificate Manager**
```yaml
certificates:
  domains:
    - api.athena.fit (API Gateway)
    - app.athena.fit (Web Application)
    - cdn.athena.fit (CDN)
    
  renewal: automatic
  providers:
    - aws_acm: For AWS services
    - cert_manager: For Kubernetes

### 9. Disaster Recovery Plan

#### 9.1. Backup стратегии
```yaml
backup:
  databases:
    - postgresql: Continuous + daily snapshots
    - mongodb: Continuous + hourly snapshots
    - redis: Daily snapshots
    
  storage:
    - s3: Versioning + cross-region replication
    - ebs: Daily snapshots
    
  retention:
    - daily: 35 days
    - weekly: 12 weeks
    - monthly: 36 months

#### 9.2. Recovery процедуры
**RTO/RPO targets:**
  - Критические сервисы: RTO < 30 min, RPO < 5 min
  - Важные сервисы: RTO < 2 hours, RPO < 1 hour
  - Остальные сервисы: RTO < 4 hours, RPO < 4 hours
 
### 10. Cost Management и оптимизация

#### 10.1. Budget alerts и оптимизации
```yaml
cost_management:
  budgets:
    - monthly: $50,000 (alert at 80%)
    - quarterly: $150,000 (alert at 90%)
    
  optimizations:
    - reserved_instances: For stable workloads
    - spot_instances: For batch processing
    - auto_scaling: Based on demand

#### 10.2. Resource tagging
```yaml
tags:
  mandatory:
    - Project: athena
    - Environment: production/staging/development
    - CostCenter: engineering
    - Owner: platform-team
    
  optional:
    - Version: 1.0.0
    - DeploymentDate: 2024-01-20