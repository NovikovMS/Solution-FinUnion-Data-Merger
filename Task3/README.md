# Задание 3. Целевая платформа данных

## Результат

Спроектирована целевая архитектура данных объединённого банка на 12 месяцев. Выбран гибрид PostgreSQL DWH и Data Lakehouse на object storage с Apache Iceberg. Trino предоставляет SQL-доступ к большой истории, Kafka доставляет события, Airflow управляет batch/ELT, а DataHub хранит каталог, ownership, качество и lineage.

Архитектура на 12 месяцев является целевым результатом по ТЗ. Горизонт 3–5 лет используется только для проверки запаса и доступных вариантов развития. Дополнительные компоненты не внедряются заранее, если фактическая нагрузка не достигает измеримых триггеров.

Архитектура сохраняет принятые границы:

- профильные OLTP остаются мастерами финансовых и продуктовых данных.
- MDM Customer Hub управляет золотой записью клиента.
- RDM и Product Catalog публикуют канонические справочники и продукты.
- Integration Layer не становится владельцем бизнес-данных.
- Lakehouse хранит исходную и полную детальную историю.
- PostgreSQL DWH публикует сверенные витрины для BI и отчётности.
- Data Mesh применяется как набор принципов data ownership и data as a product.

## Состав решения

| Артефакт | Содержание |
|---|---|
| [Целевая архитектура](target-data-architecture.md) | Компоненты To-Be на 12 месяцев, роли слоёв, режимы деградации и точки расширения |
| [Диаграмма To-Be](diagrams/target-data-architecture.mmd) | Operational Systems, MDM/RDM, API, Kafka, Airflow, Iceberg, Trino, PostgreSQL DWH, BI и DataHub |
| [Дерево решений масштабирования](diagrams/scalability-decisions.mmd) | Опциональные действия после 12 месяцев на основе SLA, ёмкости, качества и стоимости |
| [Архитектурные варианты](architecture-options.md) | Сравнение PostgreSQL DWH, Data Lakehouse и Data Mesh.<br><br>Критерии дальнейшего развития |
| [DWH и Lakehouse](dwh-lakehouse.md) | Распределение данных, hot/cold, Iceberg maintenance и триггеры structural scale |
| [Пайплайны данных](data-pipelines.md) | 10 пайплайнов, replay, dedup, archive promotion, workload isolation и SLO |
| [Диаграмма пайплайнов](diagrams/data-pipelines.mmd) | Путь от producers через ingestion и Iceberg к DWH, Trino и потребителям |
| [Аналитическая модель](analytical-model.md) | Звезда Payments & Transactions, SCD, point-in-time certification и эволюция при росте |
| [Диаграмма звезды](diagrams/transactions-star-schema.mmd) | `fact_transactions` и девять аналитических измерений |
| [Доменные data products](data-products.md) | Пять продуктов, уровни Provisional/Certified/Archived и согласованные SLO |

## Ключевые решения

1. **Гибрид вместо единственного хранилища.** DWH оптимизирован для стабильной отчётности, Lakehouse — для полной истории, событий и повторной обработки.
2. **Открытый table format.** Apache Iceberg добавляет ACID, schema evolution, partition evolution и time travel поверх object storage.
3. **Разделение хранения и SQL.** Trino выполняет интерактивные запросы к Iceberg, но не хранит данные и не обслуживает OLTP.
4. **События и batch имеют разные инструменты.** Kafka переносит доменные события, Airflow управляет batch, backfill и публикацией.
5. **Raw остаётся неизменяемым.** Standardized нормализует схемы и идентификаторы, curated публикует проверенные доменные наборы.
6. **DWH получает только сертифицированные данные.** Publication Gate блокирует BI и отчётность при нарушении финансовой сверки.
7. **Операция имеет единый `source_domain`.** Core, Cards и Loans публикуют общий аналитический факт без потери происхождения.
8. **Data products имеют доменных владельцев.** Общая Platform Team предоставляет инструменты, но не забирает бизнес-ответственность.
9. **Масштабирование запускается сигналом, а не календарём.** Read replicas, hot/cold, Stream Processor или MPP добавляются только после подтверждения ограничения.

## Готовность к дальнейшему росту

Через 12 месяцев система имеет необходимые точки расширения:

- Kafka и consumers масштабируются по доменам.
- Iceberg поддерживает partition evolution и разделение хранения от compute.
- Trino разделяет workloads через resource groups.
- Airflow изолирует daily certification, maintenance и backfill.
- PostgreSQL может использовать read replicas, rolling hot window и агрегаты.
- Контракты data products не зависят от конкретного физического certified engine.

Без обслуживания платформа будет деградировать. Наиболее вероятные причины — рост small files, конкуренция backfill с daily batch, исчерпание PostgreSQL load window, MDM backlog и неполный lineage.

Деградация контролируется относительными показателями:

- CDC lag к SLA.
- batch duration к доступному окну.
- PostgreSQL capacity и p95 запросов.
- Iceberg file count к post-compaction baseline.
- MDM queue к суточной мощности.
- broken lineage для critical datasets.
- стоимость платформы к росту бизнес-объёма.

Если показатели остаются в целевом диапазоне, архитектура не меняется. Если устойчиво превышают триггеры, применяется минимальное подходящее расширение и повторный нагрузочный тест.

## Допущения

- PostgreSQL DWH подходит для промежуточного объёма и сертифицированных витрин. Фактическая ёмкость подтверждается нагрузочным тестированием.
- Kafka, Iceberg, Trino, Airflow и DataHub обозначают целевые технологические роли. Эквивалент допускается при сохранении контрактов и свойств.
- Object storage поддерживает versioning, lifecycle policy, шифрование и retention lock.
- Источники предоставляют CDC либо контролируемые инкрементальные выгрузки.
- Конкретные RPO, RTO и сроки хранения утверждаются после Business Impact Analysis и правовой оценки.
- PAN не покидает PCI-контур в открытом виде.
- Data products используют BIAN как референс терминологии и границ, а не как обязательную физическую декомпозицию сервисов.
