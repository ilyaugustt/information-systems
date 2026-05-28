# Диаграмма структуры данных (ER)

**Описание:** Система использует распределённое хранилище. Горячие данные — Redis (кэш фич, временные решения), OLTP — PostgreSQL (модели, разметка, аккаунты), аналитика — ClickHouse (audit log, агрегаты тональности).

## ER-диаграмма (основные сущности)

```mermaid
erDiagram
    COMMENT {
        uuid id PK
        uuid user_id FK
        uuid post_id FK
        text content
        timestamp created_at
        varchar status "approved|blocked|pending"
    }

    USER {
        uuid id PK
        varchar username
        timestamp registered_at
        int violation_count
        float risk_score
    }

    POST {
        uuid id PK
        uuid author_id FK
        varchar topic
        varchar content_type
        timestamp published_at
    }

    ML_DECISION {
        uuid id PK
        uuid comment_id FK
        varchar model_version FK
        float sentiment_positive
        float sentiment_neutral
        float sentiment_negative
        float violation_score
        varchar final_label "positive|neutral|negative"
        boolean violation_flag
        float decision_threshold
        timestamp processed_at
        varchar decision_source "auto|human_override"
    }

    HUMAN_REVIEW {
        uuid id PK
        uuid ml_decision_id FK
        uuid moderator_id FK
        varchar original_label
        varchar corrected_label
        boolean violation_override
        text reason
        timestamp reviewed_at
    }

    MODEL_REGISTRY {
        varchar version PK
        varchar model_name
        float precision_violation
        float recall_violation
        float macro_f1_sentiment
        timestamp trained_at
        timestamp deployed_at
        boolean is_active
    }

    MODERATOR {
        uuid id PK
        varchar username
        varchar role
        int daily_review_count
    }

    AUDIT_LOG {
        uuid id PK
        uuid comment_id FK
        uuid decision_id FK
        varchar action "block|approve|escalate"
        varchar actor "ml_auto|moderator"
        timestamp action_at
        jsonb metadata
    }

    COMMENT ||--|| ML_DECISION : "gets classified by"
    COMMENT }o--|| USER : "written by"
    COMMENT }o--|| POST : "belongs to"
    ML_DECISION ||--o| HUMAN_REVIEW : "may be overridden"
    HUMAN_REVIEW }o--|| MODERATOR : "reviewed by"
    ML_DECISION }o--|| MODEL_REGISTRY : "produced by"
    ML_DECISION ||--|| AUDIT_LOG : "logged in"
```

## Распределённое хранилище

| Хранилище | Сущности | Обоснование |
|---|---|---|
| **PostgreSQL** | USER, POST, COMMENT, MODERATOR, MODEL_REGISTRY, HUMAN_REVIEW | ACID, внешние ключи, транзакции при обновлении статуса комментария |
| **ClickHouse** | AUDIT_LOG, агрегаты ML_DECISION | Аналитические запросы на 500M+ строках; columnar-хранение |
| **Redis** | Кэш risk_score пользователя, кэш фич поста, очередь pending reviews | TTL-кэш для < 1 мс доступа при инференсе |
| **MLflow** | Артефакты MODEL_REGISTRY (веса моделей, метрики) | Версионирование, A/B сравнение, rollback |
| **S3 (Yandex Object Storage)** | Сырые тексты комментариев для переобучения, бэкапы | Дешёвое хранение больших объёмов, доступ батчами |

**Почему данные распределены:** горячий путь инференса требует доступа к risk_score и фичам поста за < 1 мс — только Redis. OLTP-транзакции при блокировке — PostgreSQL. Аналитика тональности за квартал (сотни миллионов строк) — только ClickHouse справляется за приемлемое время.
