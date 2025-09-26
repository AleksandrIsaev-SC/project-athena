
## 14c-concurrency-view.md

```markdown
# Представление многозадачности и параллелизма

## Модели параллельной обработки

### 1. Обработка высоконагруженных операций

#### 1.1. Параллельная запись тренировок
**Сценарий:** Тысячи одновременных тренировок по всему миру

**Архитектура обработки**
Mobile Clients → [Load Balancer] →
→ [API Gateway Instances] →
→ [Workout Service Cluster] →
→ [Message Broker Partitions] →
→ [Consumer Groups]


**Стратегии параллелизма:**
- **Sharding по user_id:** Распределение нагрузки между инстансами
- **Event-driven архитектура:** Асинхронная обработка тяжелых операций
- **Backpressure механизмы:** Защита от перегрузки

#### 1.2. Конкурентный доступ к социальному графу
**Проблема:** Множественные чтения/записи социальных связей

**Решение:**
```java
// Optimistic locking для социальных взаимодействий
public class SocialInteraction {
    @Version
    private Long version;
    
    public boolean addLike(String userId, String workoutId) {
        // CAS (Compare-and-Swap) операция
        return redisTemplate.opsForValue()
            .setIfAbsent(likeKey(userId, workoutId), true, Duration.ofHours(24));
    }
}

### 2. Модели обработки событий

#### 2.1. Kafka Consumer Groups для масштабирования
**Топик: workout_events**
```yaml
partitions: 50  # Для горизонтального масштабирования
replication_factor: 3
retention_ms: 604800000  # 7 дней

consumers:
  - analytics-consumer-group: 10 instances
  - social-consumer-group: 5 instances  
  - gamification-consumer-group: 3 instances
  - notifications-consumer-group: 8 instances

**Распределение партиций**
  - Ключ партиционирования: user_id % number_of_partitions
  - Гарантия порядка: События одного пользователя обрабатываются последовательно
  - Балансировка нагрузки: Automatic rebalancing при добавлении/удалении консьюмеров

#### 2.2. Обработка в реальном времени
**Stream Processing топологии**
```java
// Аналитика тренировок в реальном времени
KStream<String, WorkoutEvent> workoutStream = builder.stream("workout_events");

workoutStream
    .filter((key, workout) -> workout.getSportType() == "running")
    .groupByKey()  // Группировка по user_id
    .windowedBy(TimeWindows.of(Duration.ofDays(7)))
    .aggregate(
        WeeklyStats::new,
        (userId, workout, stats) -> stats.update(workout),
        Materialized.with(Serdes.String(), weeklyStatsSerde)
    )
    .toStream()
    .to("weekly_analytics");

### 3. Управление состоянием и блокировки

#### 3.1. Распределенные блокировки для критических операций
**Сценарий:** Обновление лидербордов во время массовых событий
```python
class LeaderboardService:
    def update_leaderboard(self, user_id, score):
        lock_key = f"leaderboard_lock:{user_id}"
        
        # Acquire distributed lock
        with redis.lock(lock_key, timeout=5000, blocking_timeout=5000):
            current_score = self.get_current_score(user_id)
            new_score = current_score + score
            
            # Atomic update
            pipeline = redis.pipeline()
            pipeline.zadd("global_leaderboard", {user_id: new_score})
            pipeline.zadd("weekly_leaderboard", {user_id: new_score})
            pipeline.execute()

#### 3.2. Optimistic vs Pessimistic locking
**Optimistic locking (для социальных взаимодействий)**
  - Версионирование записей в PostgreSQL
  - Retry механизм при конфликтах
  - Подходит для high-read/low-write сценариев
  
**Pessimistic locking (для финансовых транзакций)**
  - Row-level locks в БД
  - Подходит для high-contention операций
 
### 4. Многопоточность на уровне сервисов

#### 4.1. Thread pooling стратегии
**Workout Service конфигурация**
```yaml
server:
  tomcat:
    threads:
      max: 200           # Максимум потоков
      min-spare: 20      # Минимум ожидающих потоков
    
spring:
  task:
    execution:
      pool:
        core-size: 10    # Core threads
        max-size: 50     # Maximum threads
        queue-capacity: 1000

**Async обработка тяжелых операций**
```java
@Service
public class WorkoutAnalysisService {
    
    @Async("analysisTaskExecutor")
    public CompletableFuture<AnalysisResult> analyzeWorkout(Workout workout) {
        // CPU-intensive analysis
        return CompletableFuture.completedFuture(performAnalysis(workout));
    }
}

#### 4.2. Reactive programming для I/O операций
**WebClient для внешних API вызовов**
```java
public Mono<UserProfile> getUserProfileWithSocialData(String userId) {
    return webClient.get()
        .uri("/users/{userId}", userId)
        .retrieve()
        .bodyToMono(UserProfile.class)
        .flatMap(profile -> 
            webClient.get()
                .uri("/social/{userId}/friends", userId)
                .retrieve()
                .bodyToMono(FriendList.class)
                .map(friends -> {
                    profile.setFriends(friends);
                    return profile;
                })
        );
}

### 5. Балансировка нагрузки и scaling

#### 5.1. Horizontal scaling стратегии
**Kubernetes HPA (Horizontal Pod Autoscaler)**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: workout-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: workout-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

#### 5.2. Load balancing алгоритмы
**API Gateway настройки**
  - Round Robin: Для равномерного распределения
  - Least Connections: Для длительных соединений
  - IP Hash: Для sticky sessions когда необходимо

### 6. Обработка пиковых нагрузок

#### 6.1. Rate limiting и throttling
**Redis-based rate limiting**
```lua
-- Lua script for sliding window rate limiting
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

local clearBefore = now - window
redis.call('ZREMRANGEBYSCORE', key, 0, clearBefore)

local current = redis.call('ZCARD', key)
if current < limit then
    redis.call('ZADD', key, now, now)
    redis.call('EXPIRE', key, window)
    return 1
else
    return 0
end

**Применение лимитов**
  - Анонимные endpoints: 100 requests/minute
  - Аутентифицированные пользователи: 1000 requests/minute
  - Административные API: 5000 requests/minute

#### 6.2. Circuit breaker паттерн
**Resilience4j конфигурация**
```yaml
resilience4j:
  circuitbreaker:
    instances:
      socialService:
        failureRateThreshold: 50
        waitDurationInOpenState: 10000
        permittedNumberOfCallsInHalfOpenState: 10
        slidingWindowSize: 20

### 7. Мониторинг параллелизма

#### 7.1. Метрики для наблюдения
**Key metrics**
  - Thread pool utilization: Активные/ожидающие потоки
  - Message queue depth: Размер очереди Kafka
  - Database connection pool: Использование соединений
  - Lock contention: Время ожидания блокировок

#### 7.2. Distributed tracing
**Jaeger traces для отладки**
```java
@Slf4j
@Service
public class WorkoutProcessingService {
    
    public void processWorkout(Workout workout) {
        Span span = tracer.buildSpan("process-workout")
            .withTag("user.id", workout.getUserId())
            .start();
        
        try (Scope scope = tracer.activateSpan(span)) {
            // Распределенная обработка
            analyzeMetrics(workout);
            updateSocialFeed(workout);
            checkAchievements(workout);
        } finally {
            span.finish();
        }
    }
}

### 8. Backpressure и обработка перегрузок

#### 8.1. Reactive streams backpressure
```java
public Flux<WorkoutEvent> getRealTimeWorkoutStream(String userId) {
    return workoutEventService.getStream(userId)
        .onBackpressureBuffer(1000) // Buffer последних 1000 событий
        .delayElements(Duration.ofMillis(100)) // Контроль скорости
        .timeout(Duration.ofSeconds(30)); // Таймаут при неактивности
}

#### 8.2. Dead letter queues для ошибок
**Kafka DLQ конфигурация**
```yaml
spring:
  cloud:
    stream:
      bindings:
        workoutEvents-in:
          destination: workout_events
          group: analytics-group
          consumer:
            maxAttempts: 3
            backOffInitialInterval: 1000
            backOffMultiplier: 2.0
        workoutEvents-dlq:
          destination: workout_events_dlq
