# UML-диаграммы поведения

## Диаграмма последовательности — обработка комментария

**Сценарий:** пользователь публикует комментарий, система классифицирует его и принимает решение об автоблокировке или отправке на ручную проверку.

```mermaid
sequenceDiagram
    actor U as Пользователь
    participant GW as API Gateway
    participant KF as Kafka
    participant ML as ML Inference
    participant DE as Decision Engine
    participant FS as Feature Store (Redis)
    participant ACT as Action Service
    participant PG as PostgreSQL
    participant CH as ClickHouse
    participant HUI as Human Review UI
    actor MOD as Модератор

    U->>GW: POST /comments {text, user_id, post_id}
    GW->>GW: JWT validation, rate limit check
    GW-->>U: 202 Accepted {comment_id}
    GW->>KF: produce → comments.raw

    Note over KF,ML: Асинхронная обработка

    KF->>ML: consume batch (up to 64 comments)
    ML->>FS: GET user_risk_score, post_topic
    FS-->>ML: {risk_score: 0.3, topic: "politics"}
    ML->>ML: tokenize + forward pass (RuBERT)
    ML-->>DE: {sentiment: [0.1, 0.2, 0.7], violation_score: 0.82}

    DE->>DE: apply threshold logic
    alt violation_score > 0.7 (auto-block)
        DE->>ACT: block_comment(comment_id, reason="auto_ml")
        ACT->>PG: UPDATE comments SET status='blocked'
        ACT->>CH: INSERT audit_log {action: 'block', actor: 'ml_auto'}
        ACT-->>DE: OK
    else violation_score 0.5–0.7 (human review)
        DE->>PG: INSERT human_review_queue {comment_id, scores}
        PG-->>DE: OK
        HUI->>MOD: 🔔 Уведомление: новый комментарий на проверке
        MOD->>HUI: view comment + ML prediction
        MOD->>HUI: click "Заблокировать" / "Одобрить"
        HUI->>ACT: moderator_decision(comment_id, label, override)
        ACT->>PG: UPDATE comments SET status=...
        ACT->>CH: INSERT audit_log {actor: 'moderator_id'}
        HUI->>PG: INSERT human_review {corrected_label} → training data
    else violation_score < 0.5 (auto-approve)
        DE->>ACT: approve_comment(comment_id)
        ACT->>PG: UPDATE comments SET status='approved'
        ACT->>CH: INSERT audit_log {action: 'approve', actor: 'ml_auto'}
    end
```

---

## Диаграмма активностей — жизненный цикл модели

```mermaid
flowchart TD
    A([Накопление размеченных данных]) --> B{Достаточно новых\nпримеров?\n≥ 5000 новых меток}
    B -- Нет --> A
    B -- Да --> C[Запуск переобучения\nв MLflow]
    C --> D[Обучение RuBERT\nна обновлённой выборке]
    D --> E[Валидация на hold-out]
    E --> F{Метрики улучшились?\nPrecision violation ≥ 0.95}
    F -- Нет --> G[Анализ ошибок\nAugmentation / HYP tune]
    G --> D
    F -- Да --> H[Регистрация модели\nв MLflow Registry]
    H --> I[Shadow mode deploy\n5% трафика]
    I --> J{Shadow метрики\nОК после 48ч?}
    J -- Нет --> K[Rollback к предыдущей версии]
    J -- Да --> L[Canary: 20% трафика]
    L --> M{Canary метрики\nОК после 24ч?}
    M -- Нет --> K
    M -- Да --> N[Full deploy: 100% трафика]
    N --> O([Мониторинг Evidently AI\nнепрерывно])
    O --> P{Drift detected?}
    P -- Нет --> O
    P -- Да --> A
```

---

## Диаграмма прецедентов (Use Case)

```mermaid
graph LR
    subgraph Actors
        U((Пользователь))
        MOD((Модератор))
        SYS((ML-система))
        ADM((Администратор))
        RKN((РКН))
    end

    subgraph UseCases ["Прецеденты системы"]
        UC1[Публикация комментария]
        UC2[Автоклассификация тональности]
        UC3[Автоблокировка нарушений]
        UC4[Ручная проверка пограничных случаев]
        UC5[Просмотр дашборда тональности]
        UC6[Настройка порогов Decision Engine]
        UC7[Экспорт отчёта для РКН]
        UC8[Управление версиями модели]
        UC9[Active learning разметка]
    end

    U --> UC1
    UC1 --> UC2
    UC2 --> UC3
    UC2 --> UC4
    MOD --> UC4
    MOD --> UC5
    ADM --> UC5
    ADM --> UC6
    ADM --> UC8
    SYS --> UC3
    SYS --> UC7
    SYS --> UC9
    RKN --> UC7
```
