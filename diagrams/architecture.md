# Архитектура системы и UML-диаграмма компонентов

## Общая архитектура (C4 Level 2 — Container Diagram)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        СОЦИАЛЬНАЯ СЕТЬ (контур)                      │
│                                                                       │
│  ┌──────────┐    ┌─────────────┐    ┌──────────────────────────────┐ │
│  │  Web UI  │    │ Mobile App  │    │  Moderator Dashboard (Web UI)│ │
│  └────┬─────┘    └──────┬──────┘    └──────────────┬───────────────┘ │
│       │                 │                           │                  │
│       └────────┬────────┘                           │                  │
│                ↓                                    │                  │
│        ┌───────────────┐                            │                  │
│        │  API Gateway  │ ←── Kong + Nginx ──────────┘                  │
│        │ rate limiting │                                               │
│        └───────┬───────┘                                               │
│                │                                                       │
│         ┌──────▼──────┐                                               │
│         │    Kafka    │ ◄── topic: comments.raw (partitions=32)        │
│         │  (3 nodes)  │ ◄── topic: decisions.audit                     │
│         └──────┬──────┘                                               │
│                │                                                       │
│    ┌───────────▼────────────┐                                         │
│    │   ML Inference Service │  (4× GPU replicas, k8s HPA)             │
│    │   FastAPI + TorchServe │                                         │
│    │   RuBERT multi-task    │                                         │
│    └───────────┬────────────┘                                         │
│                │ sentiment + violation_score                           │
│    ┌───────────▼────────────┐    ┌─────────────┐                     │
│    │   Decision Engine      │←───│ Feature Store│                     │
│    │   Business rules       │    │ Redis (hot)  │                     │
│    │   Threshold logic      │    │ PG (cold)    │                     │
│    └──────┬─────────┬───────┘    └─────────────┘                     │
│           │         │                                                   │
│    ┌──────▼───┐  ┌──▼──────────────┐                                  │
│    │  Action  │  │ Human Review     │                                  │
│    │ Service  │  │ Queue (pending)  │                                  │
│    │ (gRPC)   │  └──────┬──────────┘                                  │
│    └──────┬───┘         │                                              │
│           │      ┌──────▼──────────┐                                  │
│           │      │ Moderator UI    │                                  │
│           │      │ Human-in-the-   │                                  │
│           │      │ Loop feedback   │                                  │
│           │      └──────┬──────────┘                                  │
│           │             │                                              │
│    ┌──────▼─────────────▼──────┐                                      │
│    │        Audit Log           │                                      │
│    │       ClickHouse           │ ← все решения, 3 года                │
│    └──────────────┬────────────┘                                      │
│                   │                                                    │
│    ┌──────────────▼────────────┐   ┌───────────────────────────┐      │
│    │ Monitoring: Prometheus    │   │  MLflow Model Registry    │      │
│    │ + Grafana + Evidently AI  │   │  (версии, A/B, rollback)  │      │
│    └───────────────────────────┘   └───────────────────────────┘      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## UML-диаграмма компонентов

```mermaid
graph TD
    subgraph Client ["Клиентский слой"]
        WEB[Web Application]
        MOB[Mobile Application]
        MWEB[Moderator Web UI]
    end

    subgraph Gateway ["API Gateway Layer"]
        GW[Kong API Gateway\nAuth · RateLimit · Routing]
    end

    subgraph Ingestion ["Слой приёма данных"]
        KAFKA[Apache Kafka\ncomments.raw]
    end

    subgraph ML ["ML Inference Layer"]
        INF[ML Inference Service\nFastAPI + TorchServe]
        REG[MLflow\nModel Registry]
        INF -- loads model --> REG
    end

    subgraph Business ["Бизнес-логика"]
        DE[Decision Engine\nbusiness rules]
        FS_HOT[Redis\nFeature Store hot]
        FS_COLD[PostgreSQL\nFeature Store cold]
        DE -- reads --> FS_HOT
        FS_HOT -- cache miss --> FS_COLD
    end

    subgraph Actions ["Слой действий"]
        ACT[Action Service\ngRPC]
        HRQ[Human Review Queue]
        HUI[Human-in-the-Loop\nModerator Interface]
    end

    subgraph Storage ["Хранилище"]
        PG[(PostgreSQL\nOLTP)]
        CH[(ClickHouse\nAudit Log)]
        S3[(S3\nArtifacts)]
    end

    subgraph Monitoring ["Мониторинг и аналитика"]
        PROM[Prometheus + Grafana]
        EV[Evidently AI\nDrift Monitor]
    end

    WEB --> GW
    MOB --> GW
    MWEB --> GW
    GW --> KAFKA
    KAFKA --> INF
    INF --> DE
    DE --> ACT
    DE --> HRQ
    HRQ --> HUI
    HUI --> PG
    ACT --> PG
    ACT --> CH
    HUI --> CH
    INF --> PROM
    DE --> PROM
    CH --> EV
    EV --> PROM
    S3 --> REG
```

---

## Описание компонентов

| Компонент | Технология | Назначение |
|---|---|---|
| **Kong API Gateway** | Kong + Nginx | JWT-аутентификация, rate limiting (1000 req/s на клиента), маршрутизация |
| **Apache Kafka** | Kafka 3.x, 3 брокера | Буферизация входящего потока; decoupling между сетью и ML |
| **ML Inference Service** | FastAPI, TorchServe, Docker | Асинхронный consume из Kafka, батч-инференс, возврат scores |
| **MLflow Model Registry** | MLflow | Хранение весов, сравнение версий, авто-деплой по метрикам |
| **Decision Engine** | Python service | Применение порогов и бизнес-правил; маршрутизация решений |
| **Feature Store (Redis)** | Redis Cluster | Hot features: risk_score, post_topic, user_history (TTL 1 ч) |
| **Feature Store (PG)** | PostgreSQL | Cold features: полная история пользователя |
| **Action Service** | gRPC Python | Атомарное обновление статуса комментария в основной БД |
| **Human Review Queue** | PostgreSQL-backed queue | FIFO-очередь для пограничных случаев |
| **Moderator Interface** | React Web UI | Просмотр предсказания + кнопки одобрить/заблокировать |
| **ClickHouse** | ClickHouse 23.x | Columnar OLAP; все audit-записи; аналитика тональности |
| **Prometheus + Grafana** | — | SLA-метрики: latency, throughput, error rate |
| **Evidently AI** | — | Детекция data drift и concept drift по недельным слайсам |
