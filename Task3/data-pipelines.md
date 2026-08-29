# Пайплайны данных

Диаграмма: [`diagrams/data-pipelines.mmd`](diagrams/data-pipelines.mmd).

## Каталог пайплайнов

| ID | Пайплайн | Источник | Приёмник | Тип | Инструменты |
|---|---|---|---|---|---|
| P01 | Минимальный DWH на 3 месяца | • FU/RB CRM<br><br>• FU/RB Core<br><br>• FU/RB Cards<br><br>• FU/RB Loans | PostgreSQL DWH | Инкрементальный batch ETL | • Airflow<br><br>• SQL<br><br>• Stage tables<br><br>• Data Quality checks |
| P02 | Customer Golden Record | • CRM events<br><br>• MDM source links | • Iceberg standardized/curated<br><br>• PostgreSQL `dim_customer` | Async + ELT | • Kafka<br><br>• MDM<br><br>• Iceberg<br><br>• Airflow |
| P03 | Accounts, Payments & Transactions CDC | • Core Banking<br><br>• Payment Service<br><br>• Accounting Ledger | • Iceberg raw/standardized/curated<br><br>• DWH facts | CDC + streaming ingestion | • CDC connectors<br><br>• Kafka<br><br>• Iceberg<br><br>• Trino<br><br>• Airflow |
| P04 | Card Events | • Card Management<br><br>• Card Authorization<br><br>• Card Transaction Capture | Iceberg raw/standardized/curated | Streaming + micro-batch | • Kafka<br><br>• Stream processor<br><br>• Iceberg<br><br>• Trino |
| P05 | Loan Portfolio Snapshot | Loan Management | • Iceberg curated<br><br>• PostgreSQL portfolio mart | CDC + daily batch reconciliation | • CDC connectors<br><br>• Airflow<br><br>• Iceberg<br><br>• PostgreSQL |
| P06 | Reference and Product Data | • RDM<br><br>• Product Catalog | • Iceberg standardized<br><br>• DWH dimensions<br><br>• Consumer caches | Async publish + batch snapshot | • Kafka<br><br>• Schema Registry<br><br>• Airflow<br><br>• PostgreSQL |
| P07 | Documents and Extracted Metadata | Document Management / upload channels | • Object Storage<br><br>• Iceberg document metadata | Event-driven + batch extraction | • Object Storage<br><br>• Kafka<br><br>• Antivirus scanner<br><br>• OCR/extraction worker<br><br>• Iceberg |
| P08 | Historical Archive Ingestion | • FU Data Lake<br><br>• RB DWH exports<br><br>• Архивные файлы | Iceberg raw/standardized | Managed batch / backfill | • Object Storage<br><br>• Airflow<br><br>• Validation jobs<br><br>• Iceberg |
| P09 | Curated to Certified DWH | Iceberg curated | PostgreSQL DWH | Batch ELT + publication gate | • Airflow<br><br>• Trino/SQL<br><br>• PostgreSQL<br><br>• Data Quality framework |
| P10 | Metadata and Lineage | • Kafka<br><br>• Airflow<br><br>• Iceberg<br><br>• PostgreSQL DWH | DataHub | Async + scheduled metadata ingestion | • OpenLineage-compatible events<br><br>• DataHub emitters<br><br>• Airflow |

## P01. Минимальный PostgreSQL DWH

Пайплайн обеспечивает промежуточный результат за 3 месяца без обязательного запуска полного Lakehouse.

1. Источники формируют инкрементальные выгрузки по `updated_at` и стабильному source key.
2. Файлы принимаются в контролируемую landing-зону с manifest, row count и checksum.
3. Airflow загружает данные в PostgreSQL stage tables.
4. SQL-преобразования нормализуют типы, коды FU/RB, валюты, UTC и business date.
5. MDM source links подставляют глобальный `customer_id`.
6. Проверки сравнивают количество, суммы операций, остатки и orphan records.
7. SCD Type 2 обновляет измерения, а факты загружаются идемпотентно.
8. Успешный batch переводится в `certified` и становится доступен BI.

Ограничение P01 заключается в объёме и частоте. Полные карточные события, файлы документов и многолетняя история остаются за пределами PostgreSQL до подключения Lakehouse.

## P02. Customer Golden Record

CRM публикует клиентское изменение. Integration Layer валидирует схему и передаёт запись MDM. MDM выполняет match/merge, сохраняет source links и публикует новую версию golden record. Пайплайн записывает исходное событие в raw, нормализованную версию в standardized и разрешённую аналитическую проекцию в curated.

Автоматический merge допускается только выше утверждённого confidence threshold. Пограничные случаи поступают Data Steward. Операция split создаёт новую версию и событие исправления для потребителей.

## P03. Accounts, Payments & Transactions

CDC считывает подтверждённые изменения Core и Accounting Ledger. Kafka сохраняет порядок по `account_id` или `payment_id`. Raw хранит исходную запись и offset. Standardized добавляет глобальные идентификаторы, `source_domain`, event time, processing time и business date.

Curated разделяет:

- платёжное распоряжение и его жизненный цикл.
- бухгалтерскую операцию как неизменяемый факт.
- сторно как новую операцию со ссылкой на исходную.
- датированный снимок счёта.

Перед публикацией выполняется финансовая сверка по валюте и бизнес-дате.

## P04. Card Events

Карточный контур публикует авторизацию, клиринг, отмену и сторно. За пределы PCI-сегмента передаются `card_id`, tokenized PAN или последние четыре цифры. CVV и полный PAN не попадают в Kafka, Lakehouse, DWH и логи.

Stream processor выполняет техническую дедупликацию и добавляет event metadata. Iceberg commits могут выполняться micro-batch, чтобы не создавать большое количество маленьких файлов. Плановая compaction объединяет small files.

## P05. Loan Portfolio

CDC передаёт изменения кредита и графика. Ежедневный snapshot фиксирует остаток, просрочку, ставку и бизнес-дату. Airflow сравнивает snapshot с агрегированными выдачами, погашениями и остатком предыдущего дня.

Curated сохраняет полную версию графика и реструктуризаций. DWH получает только согласованный портфельный срез.

## P06. Reference and Product Data

RDM и Product Catalog публикуют канонический код, локальные соответствия FU/RB, версию и период действия. Consumer не принимает бизнес-событие с неизвестным обязательным кодом. Такая запись поступает в карантин до публикации маппинга или исправления producer.

## P07. Documents

Канал запрашивает короткоживущий signed URL и загружает файл в закрытый bucket. Событие загрузки запускает антивирусную проверку, вычисление хеша и регистрацию метаданных. OCR или извлечение текста выполняется только для разрешённых типов документов.

Бинарный объект остаётся в object storage. Iceberg хранит URI, хеш, версию, класс данных, бизнес-связь и результат обработки. Извлечённый текст шифруется и доступен только разрешённым сценариям.

## P08. Historical Archive

Каждая архивная поставка получает manifest:

- источник и период.
- список файлов и checksum.
- ожидаемое количество записей.
- ожидаемые финансовые суммы по валютам.
- schema version и кодировку.

Airflow загружает файлы в отдельный raw namespace, проверяет manifest и запускает нормализацию. Повторная поставка не перезаписывает предыдущую. Исправленный архив получает новую версию.

## Сценарии больших данных

| Сценарий | Источник | Характер данных | Целевое хранилище | Обработка | Потребители |
|---|---|---|---|---|---|
| История карточных операций и событий | • Card Authorization<br><br>• Card Transaction Capture<br><br>• Card Management | • Высокочастотные события<br><br>• Многолетняя история<br><br>• Поздние события<br><br>• Разные стадии авторизации и клиринга | • Iceberg raw для исходных событий<br><br>• Iceberg curated для унифицированной операции<br><br>• DWH для агрегатов | • Kafka streaming ingestion<br><br>• Micro-batch commits<br><br>• Дедупликация<br><br>• Watermark<br><br>• Compaction<br><br>• Batch reconciliation | • Anti-fraud<br><br>• Risk<br><br>• BI<br><br>• Финансовая сверка |
| Архивные данные после объединения | • FU Data Lake<br><br>• RB DWH exports<br><br>• Файловые архивы CRM/Core/Loans | • Исторические и архивные<br><br>• Структурированные и полуструктурированные<br><br>• Несколько схем и кодировок<br><br>• Неполный lineage | • Object Storage для исходных файлов<br><br>• Iceberg raw/standardized/curated<br><br>• DWH для сверенных срезов | • Managed batch<br><br>• Schema inference с ручным подтверждением<br><br>• ETL/ELT<br><br>• Backfill<br><br>• Контроль manifest и checksum | • BI<br><br>• Регуляторная отчётность<br><br>• Аудит<br><br>• Data Science |

## Общие технические правила

- Producer не удаляет и не меняет уже опубликованное финансовое событие.
- Schema Registry запрещает несовместимое изменение без новой версии.
- Raw является append-only и хранит source payload, source ID, offset и ingestion time.
- Каждая запись несёт correlation ID и ссылку на источник.
- Airflow task повторяется безопасно и не создаёт дубликат.
- DLQ используется для транспортных ошибок, карантин — для ошибок смысла и качества.
- DataHub получает lineage от source dataset до data product, DWH-таблицы и BI-витрины.
- Backfill использует отдельный resource pool и не вытесняет ежедневные критические загрузки.
