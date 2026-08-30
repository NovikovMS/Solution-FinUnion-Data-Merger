# Целевой результат миграции

## Горизонт и границы

Целевой результат фиксируется на конец 12-го месяца. Миграция объединяет данные и аналитические процессы FinUnion и RetailBank, но не включает одномоментную замену всех операционных систем.

В пределах 12 месяцев:

- FU/RB CRM продолжают регистрировать обращения, заявки и изменения клиентского профиля.
- FU/RB Core Banking, Cards и Loans остаются мастерами созданных в них финансовых и продуктовых записей.
- MDM Customer Hub становится мастером глобальной идентичности клиента.
- RDM и Product Catalog становятся мастерами канонических справочников и продуктов.
- Kafka, API Gateway и CDC заменяют мигрированные точечные потоки.
- Iceberg Lakehouse хранит полную историю.
- PostgreSQL DWH публикует сертифицированные витрины.
- DataHub хранит ownership, data contracts, качество и lineage.

Полная консолидация Core Banking, Cards, Loans и CRM является отдельной бизнес-трансформацией и не считается условием завершения этой миграции данных.

## Disposition систем

| Система или роль AS-IS | Решение на месяц 12 | Целевое состояние | Условие |
|---|---|---|---|
| FU CRM | Transform | Источник клиентских изменений и мастер обращений FinUnion | • Публикует события по канонической схеме<br><br>• Использует глобальный `customer_id` для междоменных связей<br><br>• Не формирует альтернативную golden record |
| RB CRM | Transform | Источник клиентских изменений и мастер обращений RetailBank | • Публикует события по канонической схеме<br><br>• Использует глобальный `customer_id`<br><br>• Не формирует альтернативную golden record |
| FU/RB Core Banking | Retain and integrate | Мастера собственных счетов, договоров, платежей и проводок | • CDC или контролируемый increment<br><br>• Канонические source keys<br><br>• Нет аналитических запросов из BI |
| FU/RB Card Processing | Retain and integrate | Мастера карт и карточных операций | • PAN не покидает PCI-контур<br><br>• События и CDC используют токены<br><br>• Выполняется сверка клиринга |
| FU/RB Loan Systems | Retain and integrate | Мастера кредитов, графиков и задолженности | • Общая business date<br><br>• Сверка выдач и погашений<br><br>• CDC или контролируемый increment |
| FU ESB | Partial retire | Остаётся только для немигрированных интеграций | • Мигрированный маршрут выключен после проверки consumers<br><br>• История и конфигурация сохранены<br><br>• Нет скрытого downstream |
| RB point-to-point integrations | Retire by flow | Заменены API, Kafka или управляемым CDC | • Для каждого потока есть owner и контракт<br><br>• Пройден dual run<br><br>• Подтверждено отсутствие обращений |
| FU DWH | Retire after archive | История перенесена в Lakehouse, витрины — в объединённый DWH | • BI и отчётность переключены<br><br>• Закрытые периоды воспроизводимы<br><br>• Retention подтверждён |
| RB DWH | Retire after archive | История перенесена в Lakehouse, витрины — в объединённый DWH | • BI и отчётность переключены<br><br>• Выполнена сверка исторических периодов<br><br>• Retention подтверждён |
| FU Data Lake | Retire after migration | Данные перенесены в управляемые Iceberg-зоны и object storage | • Созданы manifests и checksums<br><br>• Восстановлен lineage<br><br>• Настроен legal hold |
| FU/RB BI | Retire by report | Отчёты переведены на объединённый DWH и governed semantic layer | • Владелец подтвердил метрики<br><br>• Два отчётных периода совпали<br><br>• Нет активных пользователей |
| Локальные справочники | Retain with mapping | Используются через cross-reference к RDM | • Каждый локальный код имеет канонический код<br><br>• Версия и период действия сохранены |

Статус `Retire` относится к конкретной функции или потоку. Выключение всей платформы допускается только после технической инвентаризации и подтверждения всех владельцев.

## Новые и целевые компоненты

| Компонент | Назначение | Готовность к месяцу 12 |
|---|---|---|
| MDM Customer Hub | Golden record, match/merge, survivorship, source links | • Production HA<br><br>• Steward workflow<br><br>• Split/undo merge<br><br>• Очередь в пределах SLO |
| RDM и Product Catalog | Канонические коды, версии и cross-mapping | • Версионированные публикации<br><br>• Ответственные Stewards<br><br>• Полное покрытие critical codes |
| API Gateway | Синхронные команды и запросы | • mTLS<br><br>• Rate limits<br><br>• Версионирование<br><br>• Наблюдаемость |
| Kafka и Schema Registry | Доменные события и контракты | • HA и retention<br><br>• Replay runbook<br><br>• Compatibility checks<br><br>• DLQ |
| CDC и managed file ingestion | Инкрементальные и архивные поставки | • Идемпотентность<br><br>• Watermarks<br><br>• Контроль полноты<br><br>• Quarantine |
| Data Quality Quarantine | Изоляция записей с ошибками смысла и качества | • Owner назначается по домену<br><br>• Steward triage и aging SLA<br><br>• Producer remediation<br><br>• Audit и controlled replay |
| Iceberg Lakehouse | Raw, standardized и curated история | • Catalog<br><br>• Table maintenance<br><br>• Lifecycle policy<br><br>• Time travel |
| PostgreSQL DWH | Сертифицированные факты, измерения и витрины | • Publication Gate<br><br>• SCD и snapshot manifest<br><br>• Backup и restore test |
| Trino | SQL-доступ к Iceberg и data products | • Resource groups<br><br>• Query limits<br><br>• Изоляция ad-hoc |
| Airflow | Оркестрация batch, backfill и certification | • Раздельные pools<br><br>• Retry и checkpoint<br><br>• SLA monitoring |
| DataHub | Каталог, ownership, data contracts и lineage | • Critical assets имеют owner<br><br>• Lineage source-to-report<br><br>• Quality status |

## Мигрируемые данные

| Волна | Сущности | Способ | Результат |
|---|---|---|---|
| Master data | • Клиент<br><br>• Согласие<br><br>• Продукт<br><br>• Справочное значение | • Full load<br><br>• Match/merge<br><br>• Source-link registry<br><br>• Incremental events | MDM, RDM и Product Catalog публикуют канонические версии |
| Product relations | • Счёт<br><br>• Договор<br><br>• Карта<br><br>• Кредит<br><br>• Заявка | • CDC<br><br>• Batch snapshots<br><br>• Global ID mapping | Product records связаны с глобальным клиентом и продуктом |
| Financial facts | • Платёж<br><br>• Операция<br><br>• Контрагент | • CDC и events<br><br>• Immutable raw<br><br>• Reconciliation | Полная история и certified аналитические факты |
| Service and evidence | • Обращение<br><br>• Документ | • Events<br><br>• Managed file ingestion<br><br>• Checksum manifest | Метаданные связаны с клиентом, содержимое хранится в защищённом object storage |
| Historical analytics | • FU/RB DWH history<br><br>• FU Data Lake<br><br>• Архивные выгрузки | • Partitioned backfill<br><br>• Validation<br><br>• Archive promotion | Воспроизводимая история в Iceberg и закрытые certified snapshots |

Raw сохраняет исходную запись без исправления. Нормализация и дедупликация выполняются в standardized, а data products публикуются из curated.

## Переключаемые потоки

| Поток AS-IS | Целевой поток | Стратегия переключения |
|---|---|---|
| FU/RB CRM → локальные копии клиента | FU/RB CRM → Kafka → MDM → golden-record events | Shadow, dual run, canary по подразделениям, затем полный switch |
| FU/RB OLTP → локальные DWH | FU/RB OLTP → CDC → Iceberg → Publication Gate → объединённый DWH | Параллельный расчёт двух отчётных периодов |
| FU/RB DWH → локальный BI | Объединённый DWH → governed BI | Переключение по отчёту после business sign-off |
| FU Data Lake и RB files | Managed ingestion → Iceberg raw → standardized → curated | Волны по периоду и домену с checksum |
| FU ESB и RB point-to-point | API Gateway или Kafka по контракту | Выключение каждого маршрута после consumer confirmation |
| Локальные справочники | RDM version → Kafka/API → consumers | Dual lookup, diff report, переход на канонический код |

## Пятиминутные отчёты

Для оперативных отчётов целевой путь использует Kafka или CDC, инкрементальные пятиминутные окна и Provisional data product. Полный пересчёт истории каждые пять минут запрещён.

Пятиминутная публикация содержит:

- `as_of_timestamp`.
- watermark источников.
- статус `Provisional`.
- долю опоздавших и отклонённых записей.
- ссылку на последнюю Certified-версию.

Certified-статус выдаётся только после завершения сверки. Если Airflow micro-batch не укладывается в SLA, Stream Processor проходит отдельный Trial и вводится по измеренному latency-триггеру.

## Критерии завершения перехода

| Область | Критерий | Evidence |
|---|---|---|
| Данные | 100% critical datasets имеют owner, contract, source key и классификацию.<br><br>Source-target validation проверяет полноту, качество, связи, temporal consistency и распределения | DataHub export, validation manifest и sign-off Data Owner |
| Клиенты | Не потеряна ни одна source record.<br><br>False merge и false non-match находятся ниже утверждённых порогов | MDM reconciliation report и Steward approval |
| Финансы | Record count, обороты, остатки и сторно совпадают | Подписанный reconciliation report по business date |
| История | Закрытый период воспроизводится из Iceberg snapshot | Restore drill с новым `report_snapshot_id` |
| Потоки | Lag и error rate находятся в SLA в течение согласованного observation window | Monitoring dashboard и incident log |
| Отчёты | Два последовательных отчётных периода совпадают в пределах утверждённой политики | Parallel-run comparison и business sign-off |
| Пятиминутные витрины | Provisional-набор публикуется не реже одного раза в 5 минут при нормальной нагрузке | End-to-end freshness dashboard |
| Безопасность | Нет открытых PAN, секретов и неавторизованного массового доступа | Security review и audit evidence |
| Восстановление | Backup, replay и rollback проверены на production-like объёме | Подписанный recovery test |
| Вывод из эксплуатации | Нет активных consumers, незакрытых legal hold и обязательных batch jobs | Consumer registry, access logs и approval board |

## Что не является критерием успеха

- количество установленных технологий без рабочих data products.
- перенос копий без lineage и reconciliation.
- выключение legacy ради календарного срока.
- совпадение только агрегатов при расхождении детальных финансовых фактов.
- использование MDM как мастера счетов, платежей или задолженности.
