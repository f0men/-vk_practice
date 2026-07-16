# LLD-диаграмма системы

Диаграмма нижнего уровня: сервисы, компоненты и потоки данных, необходимые для реализации фичи.

```mermaid
flowchart TB
    subgraph Client["Клиентская часть"]
        ParentLK[ЛК родителя]
        StudentLK[ЛК ученика]
    end

    subgraph Gateway["API Gateway / BFF"]
        GW[API Gateway]
    end

    subgraph Core["Основные сервисы платформы"]
        AuthSvc[Auth Service]
        SubSvc[Subscription Service]
        PlatformGrades[(Platform Grades DB<br/>оценки внутри Учи.ру)]
    end

    subgraph Feature["Новая фича"]
        Ingest[School Grades<br/>Ingestion Service]
        SchoolGrades[(School Grades DB)]
        GapEngine[Gap Analysis Engine]
        TaskRec[Task Recommendation<br/>Engine - закрепление тем]
        InterestRec[Interest / Career<br/>Recommendation Engine]
        ContentCatalog[(Content & Courses<br/>Catalog)]
    end

    subgraph Analytics["Аналитика и эксперименты"]
        EventCollector[Event Tracking Service]
        DWH[(Data Warehouse<br/>Hive / ClickHouse)]
        ExpService[Experiment Assignment<br/>Service - ЦГ / КГ]
        MetricsJob[Metrics Computation<br/>North Star + доп. метрики]
        BI[BI / Дашборды]
    end

    ParentLK -->|логин| GW
    StudentLK -->|логин| GW
    GW --> AuthSvc
    GW --> SubSvc
    GW --> ExpService

    ExpService -->|флаг ЦГ/КГ| ParentLK

    ParentLK -->|загрузка оценок<br/>фото / ручной ввод| Ingest
    Ingest --> SchoolGrades
    Ingest --> GapEngine
    PlatformGrades --> GapEngine
    SchoolGrades --> GapEngine

    GapEngine -->|выявленные пробелы| TaskRec
    GapEngine -->|сильные темы| InterestRec
    TaskRec --> ContentCatalog
    InterestRec --> ContentCatalog

    TaskRec -->|рекомендации заданий| ParentLK
    InterestRec -->|курсы / статьи| ParentLK

    ParentLK -->|события: открытие ЛК,<br/>переход по вкладкам,<br/>клик по подписке| EventCollector
    SubSvc -->|событие оплаты| EventCollector
    EventCollector --> DWH

    DWH --> MetricsJob
    MetricsJob --> BI
    ExpService --> DWH
```

## Пояснение к компонентам

| Компонент | Назначение |
|---|---|
| **ЛК родителя** | Основной интерфейс: успеваемость, школьные оценки, пробелы, рекомендации, подписка |
| **API Gateway** | Единая точка входа, маршрутизация запросов, применение флага эксперимента (ЦГ/КГ) |
| **School Grades Ingestion Service** | Приём и валидация загруженных школьных оценок (ручной ввод/фото/интеграция с эл. дневником) |
| **Gap Analysis Engine** | Сопоставляет школьные и платформенные оценки, определяет проблемные и сильные темы |
| **Task Recommendation Engine** | Подбирает задания/материалы платформы под выявленные пробелы |
| **Interest/Career Recommendation Engine** | На основе сильных предметов предлагает курсы, статьи, профориентационный контент |
| **Event Tracking Service** | Логирует поведенческие события родителя в ЛК (сессии, вкладки, клики на подписку) |
| **Experiment Assignment Service** | Распределяет пользователей целевого сегмента на ЦГ/КГ и хранит назначение |
| **Metrics Computation** | Считает North Star и вспомогательные метрики с нужными срезами (подписка есть/нет, версия подписки) |
| **BI / Дашборды** | Визуализация метрик для оценки результатов эксперимента |

## Потенциальные точки роста
- Интеграция `School Grades Ingestion Service` с электронными дневниками школ (API), а не только ручной ввод — снижает шум в данных.
- `Interest/Career Recommendation Engine` в перспективе может стать ML-моделью (сейчас — правило-ориентированный слой поверх каталога контента).
