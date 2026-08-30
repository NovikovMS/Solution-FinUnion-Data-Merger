# Инкрементальный план миграции

## Подход

Миграция выполняется волнами по доменам и data products. Каждая волна проходит один и тот же цикл:

1. Инвентаризация и назначение владельца.
2. Контракт и canonical mapping.
3. Initial load в immutable raw.
4. Инкрементальная синхронизация.
5. Проверка качества и reconciliation.
6. Shadow и dual run.
7. Canary и переключение.
8. Hypercare и вывод старого маршрута.

Ни одна волна не ожидает завершения всех остальных доменов. Общие идентификаторы, справочники, безопасность и observability создаются раньше продуктовых миграций.

Диаграмма: [`diagrams/migration-roadmap.mmd`](diagrams/migration-roadmap.mmd).
![migration-roadmap.png](diagrams/migration-roadmap.png)

Правила проверки: [`data-validation.md`](data-validation.md).

## План на 12 месяцев

| № | Период | Этап | Основные работы | Зависимости | Exit criteria | Результат | Accountable |
|---|---|---|---|---|---|---|---|
| 1 | Месяц 1 | Mobilization и governance | • Migration Board<br><br>• RACI<br><br>• Change control<br><br>• Data Owner и Steward<br><br>• Шаблоны evidence<br><br>• Общие определения SLA, RPO и RTO | Утверждённый scope объединения | • Назначены владельцы всех critical domains<br><br>• Утверждены decision rights<br><br>• Создан журнал рисков и зависимостей | Управляемая программа миграции | Executive Sponsor и Chief Data Officer |
| 2 | Месяцы 1–2 | Data discovery и critical scope | • Реестр критичных систем, таблиц, файлов и потоков<br><br>• Consumer registry для витрин первого релиза<br><br>• Профилирование клиента, счёта, продукта и операции<br><br>• Поиск PII и PCI<br><br>• Business keys, контрольные суммы и BIAN-маппинг<br><br>• Расширенный discovery остальных доменов продолжается после месяца 3 | Владельцы доступны для подтверждения | • 100% потоков промежуточного MVP имеют owner<br><br>• Для источников MVP известны consumers<br><br>• Зафиксированы business keys, baseline качества и правила retention | Baseline AS-IS для промежуточного MVP и backlog полного обследования | Enterprise Data Architect |
| 3 | Месяцы 1–2 | Foundation промежуточного MVP | • Среды dev/test/prod<br><br>• IAM, secrets и monitoring<br><br>• Airflow для P01<br><br>• PostgreSQL DWH со stage/certified schemas<br><br>• Минимальный MDM Customer Hub<br><br>• Управляемый batch/инкрементальный Integration Layer<br><br>• Минимальный DataHub | Security architecture и capacity estimate MVP | • Backup/restore проверен<br><br>• Сквозной synthetic flow source → MDM/DWH → BI виден в lineage<br><br>• Доступ выдаётся по RBAC/ABAC<br><br>• Повторный batch идемпотентен | Готовая основа для трёхмесячного результата | Platform Owner |
| 4 | Месяцы 2–3 | Промежуточный MVP: единый ID и отчётность | • Initial customer load<br><br>• Match/merge, source links и глобальный `customer_id`<br><br>• Critical RDM cross-mapping и Product Catalog mapping<br><br>• P01 incremental batch из FU/RB CRM, Core, Cards и Loans<br><br>• Витрины клиентов, счетов, продуктов и операций<br><br>• DQ checks, quarantine, BIAN mapping и migration backlog | Critical discovery и MVP foundation | • Нет потерянных source records<br><br>• False merge ниже утверждённого порога<br><br>• Critical codes имеют mapping<br><br>• Record count и финансовые суммы сверены<br><br>• BI-витрины опубликованы из certified batch<br><br>• Владельцы и lineage зарегистрированы | К концу месяца 3 доступны единый клиентский ID, временные потоки и единая управленческая отчётность | Customer Data Owner, Finance Data Owner и Analytics Platform Owner |
| 5 | Месяцы 3–6 | Industrialization MDM, integration и Lakehouse | • Расширение MDM-модели и Steward workflow<br><br>• Полноценные RDM и Product Catalog<br><br>• Kafka и Schema Registry<br><br>• CDC Core/Cards/Loans<br><br>• Managed file ingestion<br><br>• Object Storage, Iceberg raw/standardized/curated<br><br>• Table maintenance и replay playbook | Принятый промежуточный MVP и source access | • Initial load и delta не имеют разрыва<br><br>• Undo merge протестирован<br><br>• Каждая quarantine record имеет Data Steward и Producer Owner<br><br>• Replay выполнен на production-like объёме<br><br>• Compaction укладывается в окно | Промежуточные компоненты усилены до целевого эксплуатационного уровня | Data Platform Owner и Domain Data Owners |
| 6 | Месяцы 4–8 | Расширение DWH и analytical products | • Перевод P01-потоков на curated → DWH по доменам<br><br>• Conformed dimensions<br><br>• `fact_transactions` и портфельные факты<br><br>• SCD<br><br>• Publication Gate<br><br>• Snapshot manifest<br><br>• Customer 360 и портфельные products | Работающий P01, curated data и master identifiers | • Финансовая сверка пройдена<br><br>• Point-in-time воспроизводим<br><br>• BI нагрузочный тест успешен<br><br>• P01 exit criteria выполнены по первой целевой волне | MVP-отчётность развивается в целевую сертифицированную аналитику без перерыва сервиса | Analytics Platform Owner и Finance Data Owner |
| 7 | Месяцы 6–9 | Доменные migration waves | • Волна Customer<br><br>• Accounts & Balances<br><br>• Payments & Transactions<br><br>• Cards Portfolio<br><br>• Loan Portfolio<br><br>• Documents и history | MDM, RDM, ingestion и quality gates | • Каждый product имеет contract и SLO<br><br>• Dual run пройден<br><br>• Consumer подтвердил switch<br><br>• Есть tested fallback | Домены переключены независимо | Domain Data Owners |
| 8 | Месяцы 8–10 | BI и near-line reporting | • Перенос отчётов по приоритету<br><br>• Согласование метрик<br><br>• Semantic layer<br><br>• Пятиминутные Provisional-витрины<br><br>• Обучение пользователей | Certified DWH и product contracts | • Два отчётных периода совпали<br><br>• Пятиминутный поток укладывается в freshness SLO<br><br>• Certified и Provisional явно разделены | Единые отчёты и оперативные витрины | BI Product Owner |
| 9 | Месяцы 9–11 | Cutover и legacy isolation | • Customer MDM cutover<br><br>• Переключение data routes<br><br>• Read-only legacy<br><br>• Consumer monitoring<br><br>• Архивирование конфигурации | Успешный rehearsal и Go/No-Go | • Post-cutover controls зелёные<br><br>• Нет необработанной delta<br><br>• Rollback window закрыт решением Board | Новые маршруты являются production path | Migration Manager |
| 10 | Месяцы 10–12 | Decommission и stabilization | • Отключение старых отчётов и migrated routes<br><br>• Архивы и retention<br><br>• Cost optimization<br><br>• DR exercise<br><br>• Передача в BAU | Hypercare завершён и consumers подтверждены | • Нет обращений к retired assets<br><br>• Restore drill пройден<br><br>• Runbooks приняты Operations<br><br>• Residual risks утверждены | Устойчивая эксплуатация целевой платформы | Operations Owner |

Периоды перекрываются только там, где есть независимые команды и формально завершённый входной gate. Компоненты месяца 3 имеют ограниченный production scope: минимальный MDM обслуживает идентичность клиента, P01 — четыре обязательные управленческие витрины, а интеграция использует управляемые batch/инкрементальные поставки. Этапы 5–6 не создают эти возможности заново, а повышают их до целевого уровня: добавляют CDC, Kafka, Lakehouse, полные data products и расширенный governance.

## Обязательный validation gate

Перед dual run, consumer switch и decommission формируется validation manifest. Он включает:

- source и target counts с отдельным учётом rejected и quarantined.
- checksums и distinct business keys.
- completeness, uniqueness и validity.
- RDM mapping и referential integrity.
- temporal consistency и сохранение истории.
- MDM false merge / false non-match.
- financial reconciliation.
- consent, KYC, PII и PCI controls.
- сравнение KPI и распределений.
- business acceptance и drill-down до source.

Результат `Pass` разрешает следующий gate. `Conditional Pass` допустим только для изолированного medium/low дефекта с owner и SLA. `Fail` блокирует затронутую волну.

## Контрольные ворота

| Gate | Момент | Решение | Обязательное evidence |
|---|---|---|---|
| G0 Scope Approved | Конец месяца 1 | Запуск discovery и foundation | • Scope<br><br>• RACI<br><br>• Funding<br><br>• Initial risk register |
| G1 Interim MVP Accepted | Конец месяца 3 | Принять промежуточный результат и продолжить industrialization | • Security approval<br><br>• MDM reconciliation и source links<br><br>• P01 financial reconciliation<br><br>• Certified BI-витрины<br><br>• Critical BIAN/RDM mapping<br><br>• Backup/restore и monitoring |
| G2 Target Data Foundation Ready | Конец месяца 5 | Разрешить domain dual run на целевых потоках | • Source-to-target mapping<br><br>• Source baseline<br><br>• Initial/delta reconciliation<br><br>• CDC/Kafka contracts<br><br>• Quality rules и thresholds<br><br>• Quarantine ownership |
| G3 Product Certified | Месяцы 6–9 по волнам | Переключить consumers | • Validation manifest<br><br>• Contract<br><br>• Lineage<br><br>• SLO<br><br>• Parallel-run report<br><br>• Business sign-off |
| G4 Cutover Go/No-Go | Месяцы 9–11 | Начать критичное переключение | • Rehearsal<br><br>• Rollback test<br><br>• On-call roster<br><br>• Business approval |
| G5 Decommission Approved | Месяцы 10–12 | Удалить старый production path | • Zero-access evidence<br><br>• Archive manifest<br><br>• Legal approval<br><br>• Operations acceptance |

Gate может иметь решения `Go`, `Conditional Go` и `No-Go`. Условное разрешение содержит owner, срок и измеримый критерий закрытия.

## Порядок доменных волн

1. **Reference Data и Product Catalog.** Устраняют расхождение кодов до загрузки фактов.
2. **Customer 360.** Создаёт глобальный `customer_id` и source links.
3. **Accounts & Balances.** Связывает продуктовые записи с клиентом.
4. **Payments & Transactions.** Формирует основной финансовый факт и reconciliation.
5. **Cards и Loans.** Подключаются после стабилизации conformed dimensions.
6. **Documents и historical archives.** Мигрируются параллельно с ограниченной скоростью.

Платежи, Cards и Loans могут идти параллельно после Customer и RDM, если имеют отдельные resource pools и команды сверки.

## Правила dual run

- Старый и новый path получают один и тот же source cut.
- Сравниваются детальные ключи, record count, финансовые суммы и business metrics.
- Расхождение имеет classification, owner и срок исправления.
- Новая платформа не исправляет source data без решения Data Owner.
- Consumer switch не выполняется по результату одного успешного запуска.
- Минимум два последовательных отчётных периода подтверждают стабильность BI.
- Пятиминутная Provisional-витрина сравнивается по watermark, freshness и late-event rate.

## Переход из P01 в целевой P09

Прямой временный ETL source-to-DWH отключается отдельно по домену:

1. CDC или события стабильно поступают в raw.
2. Standardized применяет canonical keys и RDM.
3. Curated product имеет owner, contract и quality SLO.
4. DWH воспроизводится из зафиксированного Iceberg snapshot.
5. P01 и P09 совпадают по детальным ключам и контрольным суммам.
6. Consumer подтверждает переключение.
7. P01 выключается, а конфигурация и последний успешный результат архивируются.

## Decommission workflow

1. Перевести legacy asset в read-only.
2. Отключить расписание записи, сохранив возможность controlled restart.
3. Наблюдать access logs в течение утверждённого окна.
4. Получить подтверждение владельцев и consumers.
5. Зафиксировать final backup, manifest, checksum и retention.
6. Проверить legal hold и требования аудита.
7. Удалить secrets, routes и service accounts.
8. Освободить compute только после G5.

Если обнаружен неизвестный consumer, decommission останавливается. Это дефект discovery, а не основание отключить consumer принудительно.

## Управление изменениями

- Во время migration wave запрещены несовместимые изменения source schema без versioned contract.
- Массовое исправление данных согласуется с Migration Manager и Data Owner.
- Cutover и backfill не делят один resource pool с daily certification.
- Каждый production change имеет correlation ID, change ticket и rollback owner.
- Решения и отклонения хранятся в журнале Migration Board.
