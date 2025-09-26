# Информационное представление архитектуры

## Модели данных и потоки информации

### 1. Основные сущности данных

#### 1.1. Пользователь (User)
```json
{
  "user_id": "uuid_v7",
  "identity": {
    "email": "user@example.com",
    "phone": "+1234567890",
    "social_logins": ["google:123", "apple:456"]
  },
  "profile": {
    "personal_info": {
      "name": "John Doe",
      "birth_date": "1990-01-01",
      "gender": "male",
      "height_cm": 180,
      "weight_kg": 75
    },
    "sports_profile": {
      "primary_sports": ["running", "cycling"],
      "fitness_level": "intermediate",
      "goals": ["weight_loss", "endurance"],
      "weekly_target_minutes": 300
    },
    "privacy_settings": {
      "profile_visibility": "public",
      "activity_sharing": "friends_only",
      "location_sharing": "during_workouts"
    }
  },
  "preferences": {
    "notifications": {
      "workout_reminders": true,
      "social_interactions": true,
      "promotional": false
    },
    "units": {
      "distance": "kilometers",
      "weight": "kilograms",
      "temperature": "celsius"
    }
  },
  "timestamps": {
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-20T14:25:00Z",
    "last_login": "2024-01-20T14:25:00Z"
  }
}
#### 1.2. Тренировка (Workout)
```json
{
  "workout_id": "uuid_v7",
  "user_id": "uuid_v7",
  "metadata": {
    "sport_type": "running",
    "sub_sport": "road_running",
    "start_time": "2024-01-20T08:15:00Z",
    "end_time": "2024-01-20T09:30:00Z",
    "timezone": "Europe/Moscow",
    "device_info": {
      "device_id": "garmin_fenix_7_123",
      "software_version": "10.2.1",
      "sensors_used": ["gps", "heart_rate", "cadence"]
    }
  },
  "metrics": {
    "summary": {
      "duration_seconds": 4500,
      "distance_meters": 12500,
      "calories_kcal": 850,
      "average_heart_rate": 145,
      "max_heart_rate": 175,
      "elevation_gain_meters": 150
    },
    "samples": [
      {
        "timestamp": "2024-01-20T08:15:00Z",
        "heart_rate": 120,
        "latitude": 55.7558,
        "longitude": 37.6173,
        "altitude": 150,
        "cadence": 85,
        "speed_mps": 2.5
      }
    ],
    "laps": [
      {
        "lap_number": 1,
        "start_time": "2024-01-20T08:15:00Z",
        "end_time": "2024-01-20T08:30:00Z",
        "distance_meters": 2500,
        "average_pace_sec_per_km": 360
      }
    ]
  },
  "derived_insights": {
    "performance_score": 85,
    "recovery_time_hours": 48,
    "training_effect": 3.2,
    "weather_conditions": {
      "temperature_c": 15,
      "humidity_percent": 65,
      "conditions": "partly_cloudy"
    }
  }
}
#### 1.3. Социальные взаимодействия (Social Interactions)
```json
{
  "interaction_id": "uuid_v7",
  "type": "workout_like",
  "actor_user_id": "uuid_v7",
  "target_user_id": "uuid_v7",
  "target_workout_id": "uuid_v7",
  "content": {
    "message": "Great pace!",
    "reaction_type": "like"
  },
  "timestamps": {
    "created_at": "2024-01-20T10:15:00Z",
    "updated_at": "2024-01-20T10:15:00Z"
  },
  "visibility": "public"
}
### 2. Потоки данных в системе

#### 2.1. Поток записи тренировки
Mobile App → [GPS/HR Data] → Workout Service → [Validation] →
  → Workout Service → [Persistence] → MongoDB →
  → [Event: WorkoutCompleted] → Kafka →
    → Analytics Service → [Processing] →
    → Social Service → [Notifications] →
    → Gamification Service → [Achievement Check]

#### 2.2. Поток социальных взаимодействий
Client → [Like/Comment] → Social Service → [Validation] →
  → Social Service → [Persistence] → PostgreSQL →
  → [Event: SocialInteraction] → Kafka →
    → Notification Service → [Push Notification] →
    → Activity Feed Service → [Feed Update] →
    → Analytics Service → [Engagement Metrics]

### 3. Хранилища данных

#### 3.1. Операционные базы данных
- **PostgreSQL (транзакционные данные)**
  - Таблицы: users, social_connections, groups, achievements
  - Индексы: user_id, email, social_graph
  - Репликация: Master-Slave с автоматическим failover
- **MongoDB (документные данные)**
  - Коллекции: workouts, user_profiles, activity_feeds
  - Индексы: user_id, timestamp, geospatial
  - Sharding: По user_id для горизонтального масштабирования

#### 3.2. Кэш и быстрый доступ
- **Redis Cluster**
  - Сессии пользователей (TTL: 24 hours)
  - Кэш социального графа (TTL: 1 hour)
  - Лидерборды (Real-time updates)
  - Rate limiting counters 
- **Elasticsearch**
  - Поиск пользователей и групп
  - Full-text search по контенту
  - Геопространственные запросы

#### 3.3. Аналитические хранилища
- **Google BigQuery**
  - Исторические данные тренировок
  - Агрегированная аналитика
  - ML features storage
- **AWS S3 (Data Lake)**
  - Raw event dataк
  - Backup архивов
  - ML training datasets

### 4. Миграции и версионирование данных

- **Стратегия миграции**
```sql
  -- Example migration for new workout metrics
  - ALTER TABLE workouts ADD COLUMN IF NOT EXISTS weather_conditions JSONB DEFAULT '{}';
- **Версионирование API**
  - /api/v1/workouts  (current)
  - /api/v2/workouts  (future)
  - /api/v1/workouts?version=2 (gradual migration)

### 5. Data Governance и качество данных

#### Валидация данных
  - JSON Schema validation для всех API endpoints
  - Data quality checks в ETL процессах
  - Anomaly detection для метрик тренировокs

#### Data Retention Policies
  - Тренировочные данные: 7 ле
  - Социальные взаимодействия: 3 года
  - Аналитические события: 1 год (агрегированные)
  - Raw sensor data: 30 дней (после обработки)

### 6. Интеграционные потоки с внешними системами

#### E-commerce интеграция
  - Product Catalog Sync → [REST API] → Commerce Service →
  → [Cache Update] → Redis → 
  → [Recommendation Update] → ML Models 

#### Device manufacturers интеграция
  - Garmin/Fitbit Webhooks → [Data Normalization] →
  → Workout Service → [Standard Format] →
  → [Processing Pipeline] → Analytics