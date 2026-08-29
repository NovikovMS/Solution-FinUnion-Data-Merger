# Задание 3. Целевая платформа данных

## Результат

Спроектирована целевая архитектура данных объединённого банка на 12 месяцев. Выбран гибрид PostgreSQL DWH и Data Lakehouse на object storage с Apache Iceberg. Trino предоставляет SQL-доступ к большой истории, Kafka доставляет события, Airflow управляет batch/ELT, а DataHub хранит каталог, ownership, качество и lineage.

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
| [Целевая архитектура](target-data-architecture.md) | Компоненты To-Be, роли слоёв, основной поток, надёжность, безопасность и ownership |
| [Диаграмма To-Be](diagrams/target-data-architecture.mmd) | Operational Systems, MDM/RDM, API, Kafka, Airflow, Iceberg, Trino, PostgreSQL DWH, BI и DataHub |
| [Архитектурные варианты](architecture-options.md) | Сравнение PostgreSQL DWH, Data Lakehouse и Data Mesh.<br><br>Обоснование гибридного решения |
| [DWH и Lakehouse](dwh-lakehouse.md) | Распределение данных, горизонты 3/12 месяцев, роли Iceberg и Trino, ограничения и меры |
| [Пайплайны данных](data-pipelines.md) | 10 пайплайнов, включая минимальный DWH, CDC, события, документы, архивы, metadata и lineage |
| [Диаграмма пайплайнов](diagrams/data-pipelines.mmd) | Путь от producers через ingestion и Iceberg к DWH, Trino и потребителям |
| [Аналитическая модель](analytical-model.md) | Звезда Payments & Transactions, гранулярность, SCD, меры, поздние факты и сверка |
| [Диаграмма звезды](diagrams/transactions-star-schema.mmd) | `fact_transactions` и девять аналитических измерений |
| [Доменные data products](data-products.md) | Customer 360, Accounts & Balances, Payments & Transactions, Cards Portfolio и Loan Portfolio |

## Ключевые решения

1. **Гибрид вместо единственного хранилища.** DWH оптимизирован для стабильной отчётности, Lakehouse — для полной истории, событий и повторной обработки.
2. **Открытый table format.** Apache Iceberg добавляет ACID, schema evolution, partition evolution и time travel поверх object storage.
3. **Разделение хранения и SQL.** Trino выполняет интерактивные запросы к Iceberg, но не хранит данные и не обслуживает OLTP.
4. **События и batch имеют разные инструменты.** Kafka переносит доменные события, Airflow управляет batch, backfill и публикацией.
5. **Raw остаётся неизменяемым.** Standardized нормализует схемы и идентификаторы, curated публикует проверенные доменные наборы.
6. **DWH получает только сертифицированные данные.** Publication Gate блокирует BI и отчётность при нарушении финансовой сверки.
7. **Операция имеет единый `source_domain`.** Core, Cards и Loans публикуют общий аналитический факт без потери происхождения.
8. **Data products имеют доменных владельцев.** Общая Platform Team предоставляет инструменты, но не забирает бизнес-ответственность.

## Допущения

- PostgreSQL DWH подходит для промежуточного объёма и сертифицированных витрин. Фактическая ёмкость подтверждается нагрузочным тестированием.
- Kafka, Iceberg, Trino, Airflow и DataHub обозначают целевые технологические роли. Эквивалент допускается при сохранении контрактов и свойств.
- Object storage поддерживает versioning, lifecycle policy, шифрование и retention lock.
- Источники предоставляют CDC либо контролируемые инкрементальные выгрузки.
- Конкретные RPO, RTO и сроки хранения утверждаются после Business Impact Analysis и правовой оценки.
- PAN не покидает PCI-контур в открытом виде.
- Data products используют BIAN как референс терминологии и границ, а не как обязательную физическую декомпозицию сервисов.
