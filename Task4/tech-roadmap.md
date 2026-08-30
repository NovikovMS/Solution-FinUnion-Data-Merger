# Технологическая дорожная карта

## Принцип

Зона показывает решение на 12-месячном горизонте:

- **Adopt:** внедрить и использовать в production.
- **Trial:** проверить на ограниченном production-like или canary-сценарии.
- **Assess:** провести исследование и сохранить вариант развития без обязательства внедрения.
- **Hold:** не внедрять в рамках миграции.

Технологические названия обозначают роль. Допускается эквивалент при сохранении контрактов, эксплуатационных свойств и возможности миграции.

## Обзор зон

| Зона | Технология или подход | Решение | Почему |
|---|---|---|---|
| Adopt | MDM Customer Hub | Production к волне Customer | Требуется единый `customer_id`, source links, match/merge, survivorship и Steward workflow |
| Adopt | Reference Data Management | Production до продуктовых волн | Канонические коды и версии уменьшают расхождения FU/RB до загрузки фактов |
| Adopt | Product Catalog | Production до Accounts/Cards/Loans | Общий `product_id` отделяет определение продукта от локального экземпляра |
| Adopt | Kafka | Production event backbone | Развязывает producers и consumers, поддерживает replay и независимые доменные потоки |
| Adopt | Schema Registry | Обязателен для Kafka и CDC contracts | Блокирует несовместимые изменения до production consumer |
| Adopt | API Gateway | Production для синхронных сценариев | Централизует mTLS, аутентификацию, rate limiting и versioning |
| Adopt | CDC и managed file ingestion | Основной путь изменений и архивов | Исключает аналитическую нагрузку на OLTP и сохраняет watermarks |
| Adopt | Object Storage | Основное файловое и архивное хранение | Масштабируется дешевле DWH и поддерживает versioning, lifecycle и retention |
| Adopt | Apache Iceberg | Table format Lakehouse | Даёт ACID commits, schema evolution, partition evolution и time travel |
| Adopt | Iceberg Catalog | Production metadata control | Нужен для согласованных commits и управления snapshots |
| Adopt | Trino | SQL serving над Iceberg | Отделяет compute от storage и обслуживает большую историю без загрузки в DWH |
| Adopt | PostgreSQL DWH | Certified serving на 12 месяцев | Подходит для стабильных витрин при контролируемом объёме и подтверждённом load test |
| Adopt | Airflow | Batch и ELT orchestration | Управляет зависимостями, retry, backfill, pools и publication workflow |
| Adopt | Data Quality / Publication Gate | Обязательный control | Не допускает публикацию несверенных финансовых и critical datasets |
| Adopt | DataHub | Каталог и governance | Централизует ownership, contracts, quality status и lineage |
| Adopt | Immutable raw и replay | Обязательный operating model | Делает миграцию воспроизводимой и снижает риск необратимого исправления |
| Trial | Stream Processor | Пилот для пятиминутной Provisional-витрины | Нужен только если Airflow micro-batch не обеспечивает стабильный end-to-end SLA |
| Trial | PostgreSQL read replica | Пилот при BI contention | Изолирует чтение, но добавляет replication lag и эксплуатационную сложность |
| Trial | Automated lineage-as-code | Пилот на двух critical data products | Снижает broken lineage при росте числа pipelines |
| Trial | Token vault integration | Пилот для Cards analytics | Централизует токены при росте карточных сценариев, но не требуется для открытого PAN вне PCI |
| Trial | Automated canary and rollback routing | Пилот на Customer MDM | Сокращает время восстановления, но решение о rollback остаётся контролируемым |
| Assess | Columnar или MPP DWH | Benchmark без production commitment | Возможен после исчерпания PostgreSQL hot/cold, replicas и агрегатов |
| Assess | Multi-region DR | BIA и design study | Стоимость оправдана только подтверждёнными RPO/RTO и требованиями regulator |
| Assess | Object Storage tiering | Cost model и restore test | Cold tier снижает стоимость, но может увеличить recovery time |
| Assess | Separate immutable regulatory publish | Prototype manifest и restore | Нужен при требовании воспроизводить каждую публикацию независимо от текущего DWH |
| Assess | Policy-as-code для data access | Security assessment | Полезен при росте ABAC-политик, но требует зрелого IAM и ownership |
| Hold | Полная Data Mesh инфраструктура по доменам | Не внедрять | Отдельные платформы увеличат стоимость и расхождение стандартов на этапе объединения |
| Hold | Одномоментная замена FU/RB Core, CRM, Cards и Loans | Не включать | Слишком большой blast radius для 12 месяцев и не требуется для объединения данных |
| Hold | Hadoop/HDFS как новое хранилище | Не внедрять | Object Storage и Iceberg дают требуемые свойства без отдельного HDFS-кластера |
| Hold | Прямой BI-доступ к OLTP | Запретить | Создаёт конкуренцию за ресурсы и обходит quality/certification |
| Hold | Full refresh каждые 5 минут | Запретить | Не масштабируется и создаёт непредсказуемую нагрузку вместо incremental processing |
| Hold | Graph DB без подтверждённого use case | Не внедрять | Customer source links и основные связи покрываются MDM и relational моделью |
| Hold | Search Engine как источник истины | Не использовать | Индекс допустим как производная проекция, но не как мастер клиента или финансового факта |

## Дорожная карта по периодам

### Месяцы 0–3: Foundation

Внедрить:

- IAM, secrets и encryption baseline.
- Kafka и Schema Registry.
- API Gateway.
- Object Storage и Iceberg Catalog.
- Airflow с отдельными pools.
- DataHub.
- Observability и audit.

Проверить:

- synthetic source-to-report flow.
- backup/restore.
- Kafka replay.
- Iceberg commit и time travel.
- schema compatibility rejection.
- field-level masking.

Результат: G1 разрешает production ingestion.

### Месяцы 4–6: Master data и ingestion

Внедрить:

- MDM Customer Hub.
- RDM и Product Catalog.
- CDC connectors.
- managed file ingestion.
- raw, standardized и curated.
- Data Quality и quarantine.
- Iceberg table maintenance.

Проверить:

- initial load плюс delta без gap.
- MDM split/undo merge.
- archive manifest.
- compaction и replay на production-like объёме.
- source-to-curated lineage.

Результат: G2 разрешает dual run доменов.

### Месяцы 7–9: Serving и migration waves

Внедрить:

- PostgreSQL DWH.
- Trino resource groups.
- Publication Gate.
- data products и contracts.
- governed BI и semantic definitions.
- point-in-time snapshot manifest.

Провести Trial:

- read replica при подтверждённой конкуренции BI и ETL.
- lineage-as-code для Customer 360 и Payments & Transactions.
- automated canary routing для Customer MDM.

Результат: G3 разрешает consumer switch по продукту.

### Месяцы 10–12: Cutover и stabilization

Внедрить:

- production cutover Customer MDM.
- перевод отчётов.
- legacy read-only и decommission workflow.
- DR и replay exercises.
- FinOps dashboard.

Провести Trial:

- пятиминутную Provisional-витрину.
- Stream Processor только при провале micro-batch SLA.
- token vault integration для карточного сценария.

Провести Assess:

- MPP benchmark на реальном профиле запросов.
- multi-region DR по результатам BIA.
- cold storage tiering и restore time.

Результат: G5 разрешает вывод старых маршрутов и передачу платформы в BAU.

## Триггеры Trial и Assess

| Вариант | Не запускать, пока | Триггер | Критерий успешности |
|---|---|---|---|
| Stream Processor | Airflow micro-batch стабильно укладывается в 5 минут | End-to-end freshness устойчиво нарушается из-за orchestration или stateful processing | • p95 freshness в SLO<br><br>• Replay воспроизводим<br><br>• Exactly-once effect достигается идемпотентностью<br><br>• Стоимость принята |
| PostgreSQL read replica | BI не конкурирует с load | p95 BI и load window ухудшаются из-за чтения | • Primary load стабилен<br><br>• Replica lag виден consumer<br><br>• Failover procedure протестирована |
| Columnar/MPP DWH | PostgreSQL имеет запас после оптимизации | Load window, p95 и стоимость не восстанавливаются hot/cold, aggregates и replicas | • Reconciliation идентичен<br><br>• SCD и security поддержаны<br><br>• TCO оправдан |
| Multi-region DR | BIA не требует малого RTO при потере региона | Утверждённые RPO/RTO не достигаются single-region HA | • Failover drill<br><br>• Нет split-brain<br><br>• Data residency соблюдена |
| Token vault | Текущая токенизация покрывает consumers | Растёт число систем, которые должны разрешённо сопоставлять токены | • PAN не раскрывается<br><br>• Rotation и revocation работают<br><br>• Доступ аудитируется |

Сам календарный срок не является триггером технологии.

## Пятиминутная Provisional-витрина

### Базовый вариант Adopt

```text
Kafka / CDC → Raw → Incremental transform → 5-minute aggregate → Provisional view
```

Airflow запускает короткий micro-batch с watermark. Каждый запуск:

- читает только новую и позднюю delta.
- выполняет идемпотентный merge.
- публикует `as_of_timestamp`.
- не изменяет Certified snapshot.
- показывает late-event и quarantine metrics.

### Вариант Trial

Stream Processor вводится, если micro-batch не обеспечивает SLA. Он формирует stateful aggregates и пишет идемпотентный результат в serving layer. Immutable raw остаётся источником replay.

Переход на streaming не отменяет reconciliation и не превращает Provisional в Certified.

## Эксплуатационная готовность

Технология переходит в Adopt только при наличии:

- Service Owner.
- SLO и monitoring.
- backup, restore или replay.
- security model.
- capacity model.
- runbook и on-call.
- upgrade и rollback procedure.
- cost allocation.
- production-like test.

Успешная установка без этих элементов не является внедрением.

## Отказ от избыточной сложности

- Не создавать отдельный Kafka, Iceberg и DataHub для каждого домена на старте.
- Не переносить всю историю в PostgreSQL.
- Не добавлять Stream Processor только из-за наличия событий.
- Не выбирать MPP по календарю без benchmark.
- Не использовать DataHub как хранилище бизнес-данных.
- Не выполнять match/merge в Integration Layer.
- Не выключать legacy до подтверждения consumers и legal retention.
