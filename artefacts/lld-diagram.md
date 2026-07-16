# LLD — новая фича в ЛК родителя

LLD описывает только архитектуру и процесс новой фичи (импорт школьных оценок → анализ пробелов → рекомендации), в разрезе ЛК родителя. Остальные роли и модули платформы (учитель, ученик, аутентификация, подписки, аналитика) не входят в этот процесс.

## Компонентная диаграмма и поток данных

```mermaid
flowchart TB
    subgraph ParentLK["ЛК родителя"]
        UploadUI[Экран загрузки<br/>школьных оценок]
        GapsUI[Экран «Пробелы<br/>и рекомендации»]
        InterestUI[Блок «Возможности<br/>для ребёнка»]
    end

    subgraph FeatureCore["Сервисы фичи"]
        Ingest[School Grades<br/>Ingestion Service]
        GapEngine[Gap Analysis Engine]
        TaskRec[Task Recommendation<br/>Engine]
        InterestRec[Interest / Career<br/>Recommendation Engine]
    end

    subgraph Data["Хранилища данных"]
        SchoolGrades[(School Grades DB)]
        PlatformGrades[(Platform Grades DB<br/>существующие данные Учи.ру)]
        ContentCatalog[(Content & Courses<br/>Catalog)]
    end

    UploadUI -->|фото / ручной ввод<br/>школьных оценок| Ingest
    Ingest -->|валидация и запись| SchoolGrades

    SchoolGrades --> GapEngine
    PlatformGrades --> GapEngine

    GapEngine -->|слабые темы| TaskRec
    GapEngine -->|сильные темы| InterestRec

    TaskRec -->|подбор заданий| ContentCatalog
    InterestRec -->|подбор курсов / статей| ContentCatalog

    TaskRec -->|рекомендованные задания| GapsUI
    InterestRec -->|рекомендованный контент| InterestUI
```

## Процесс по шагам (sequence)

```mermaid
sequenceDiagram
    participant P as Родитель (ЛК)
    participant I as Ingestion Service
    participant SG as School Grades DB
    participant PG as Platform Grades DB
    participant G as Gap Analysis Engine
    participant TR as Task Recommendation
    participant IR as Interest Recommendation

    P->>I: Загружает школьные оценки
    I->>SG: Валидирует и сохраняет
    I-->>P: Подтверждение загрузки

    G->>SG: Забирает школьные оценки
    G->>PG: Забирает оценки платформы
    G->>G: Сопоставляет и находит<br/>пробелы / сильные темы

    G->>TR: Передаёт слабые темы
    G->>IR: Передаёт сильные темы

    TR-->>P: Задания для закрепления пробелов
    IR-->>P: Курсы / статьи по сильным темам
```

## Пояснение к компонентам

| Компонент | Назначение |
|---|---|
| **Экран загрузки школьных оценок** | Точка входа: родитель или ребёнок вносит реальные школьные оценки (фото дневника / ручной ввод) |
| **School Grades Ingestion Service** | Принимает, валидирует и нормализует загруженные оценки перед сохранением |
| **School Grades DB** | Хранилище школьных оценок, отдельное от оценок платформы |
| **Platform Grades DB** | Существующее хранилище результатов ребёнка внутри Учи.ру |
| **Gap Analysis Engine** | Сопоставляет школьные и платформенные оценки: находит темы, где ребёнок «просаживается» в школе, но платформа этого не видит, а также сильные темы |
| **Task Recommendation Engine** | По слабым темам подбирает конкретные задания/материалы платформы для закрепления |
| **Interest / Career Recommendation Engine** | По сильным темам подбирает внешний развивающий контент (курсы, статьи, профориентация) |
| **Content & Courses Catalog** | Справочник заданий платформы и внешнего контента, из которого берутся рекомендации |
| **Экран «Пробелы и рекомендации»** | Отображает родителю выявленные пробелы и предложенные задания |
| **Блок «Возможности для ребёнка»** | Отображает родителю рекомендации по сильным сторонам ребёнка |

## Точки роста
- `Ingestion Service` в будущем может получать оценки не только вручную, но и через интеграцию с электронным дневником школы (API).
- `Interest / Career Recommendation Engine` сейчас — правило-ориентированный слой поверх каталога контента, в перспективе может быть заменён на ML-модель.
