# Задание 4. Миграция, cutover и технологическая дорожная карта

## Результат

Подготовлен 12-месячный переход от разрозненных данных FinUnion и RetailBank к целевой платформе. Миграция выполняется доменными волнами с обязательными контрактами, reconciliation, dual run, canary, rollback и подтверждением владельцев.

Операционные FU/RB CRM, Core Banking, Cards и Loans не заменяются одномоментно. Они сохраняют профильную ответственность и подключаются через API, события и CDC. MDM становится мастером глобальной идентичности клиента, а объединённые Lakehouse и DWH заменяют дублирующие аналитические контуры после проверки потребителей и истории.

## Состав решения

| Артефакт | Содержание |
|---|---|
| [Целевой результат](migration-target-state.md) | Disposition систем, мигрируемые данные, переключаемые потоки и критерии завершения |
| [План миграции](migration-plan.md) | Десять этапов на 12 месяцев, контрольные ворота, доменные волны, dual run и decommission |
| [Диаграмма миграции](diagrams/migration-roadmap.mmd) | Последовательность этапов G0–G5 и возврат при No-Go |
| [Customer MDM cutover](cutover-plan.md) | Подготовка, ограниченный freeze, final delta, сверка, canary, switch, fallback и hypercare |
| [Диаграмма cutover](diagrams/customer-cutover.mmd) | Взаимодействие CRM, Integration Layer, MDM, consumers и Cutover Board |
| [Валидация данных](data-validation.md) | Многоуровневые проверки полноты, качества, связей, истории, финансов и бизнес-смысла |
| [Диаграмма валидации](diagrams/data-validation-flow.mmd) | Путь от source baseline до Pass, Conditional Pass или карантина |
| [Риски миграции](migration-risks.md) | Реестр из 19 рисков, ранние сигналы, prevention, contingency и владельцы |
| [Технологическая дорожная карта](tech-roadmap.md) | Зоны Adopt, Trial, Assess и Hold по периодам 0–12 месяцев |

## Ключевые решения

1. **Волны вместо big bang.** Каждый домен независимо проходит initial load, delta, dual run, switch и stabilization.
2. **Система не выключается по календарю.** Decommission требует отсутствия consumers, archive manifest, legal approval и tested restore.
3. **MDM управляет идентичностью, а не финансами.** Счета, платежи, операции и задолженность остаются в профильных мастерах.
4. **Freeze ограничен.** Обычная работа CRM продолжается, а bulk update, изменение правил и несовместимые схемы временно запрещаются.
5. **Rollback не удаляет бизнес-изменения.** Маршрут чтения возвращается на прежнее представление, а новые CRM changes остаются в журнале для replay.
6. **Сверка включает детали.** Равенство агрегатов не компенсирует потерю ключей, сторно, source links или consent history.
7. **Provisional и Certified разделены.** Оперативная витрина может обновляться каждые пять минут, но сертификация требует Publication Gate.
8. **Технология вводится по потребности.** Stream Processor, read replica, MPP и multi-region проходят Trial или Assess по измеримому триггеру.
9. **Legacy доступен до закрытия rollback window.** После hypercare возврат становится отдельной миграцией.
10. **Карантин не остаётся бесхозным.** Data Steward владеет triage и aging, Producer исправляет источник, Data Owner принимает решение, Platform Team обеспечивает механизм.
11. **Количество записей — только первый контроль.** Миграция также проверяет структуру, значения, справочники, связи, temporal consistency, MDM, финансы, privacy и бизнес-метрики.

## 12-месячный результат

| Область | Результат |
|---|---|
| Master data | • MDM Customer Hub с глобальным `customer_id`<br><br>• RDM cross-mapping<br><br>• Канонический Product Catalog |
| Integration | • API Gateway<br><br>• Kafka и Schema Registry<br><br>• Управляемый CDC и file ingestion<br><br>• Replay |
| Data Platform | • Iceberg raw, standardized и curated<br><br>• Trino<br><br>• PostgreSQL DWH<br><br>• Airflow |
| Governance | • DataHub<br><br>• Owners и Stewards<br><br>• Contracts<br><br>• Lineage<br><br>• Publication Gate |
| Consumers | • Customer 360<br><br>• Пять доменных data products<br><br>• Governed BI<br><br>• Provisional-витрины |
| Legacy | • Старые отчёты и data routes отключены по факту готовности<br><br>• История архивирована<br><br>• Неизвестные consumers отсутствуют |

## Критерии успеха

- Critical datasets имеют owner, contract, source key, classification и lineage.
- MDM не теряет source records и остаётся ниже утверждённых порогов false merge.
- Финансовые record count, суммы, остатки и сторно сверены.
- Source-target validation закрывает counts, checksums, nulls, duplicates, relationships, temporal rules и distribution drift.
- Каждый retired asset имеет backup, manifest, retention и approval.
- Customer cutover прошёл rehearsal и сохраняет fallback до T+7 дней.
- Два последовательных отчётных периода подтверждают новую BI-публикацию.
- Пятиминутная Provisional-витрина показывает watermark и `as_of_timestamp`.
- Backup, replay, rollback и restore проверены на production-like объёме.
- Operations принял SLO, dashboards, runbooks и on-call.

## Основные риски

Наиболее опасны:

- false merge клиента.
- потеря delta.
- неверная версия согласия или KYC.
- расхождение financial detail при равных агрегатах.
- неизвестный legacy consumer.
- неработающий fallback.
- раскрытие ПДн или PAN.
- недостаточная capacity MDM, Kafka, DWH или Steward team.
- рост карантина без назначенного Steward и Producer Owner.

Любой открытый critical defect в customer identity, consent, security или rollback приводит к No-Go.

## Допущения

- FU/RB OLTP доступны для CDC либо контролируемой инкрементальной выгрузки.
- Владельцы источников предоставляют business keys, watermarks и контрольные суммы.
- Конкретные пороги качества утверждаются Data Owners до G2.
- Конкретные RPO и RTO утверждаются после Business Impact Analysis.
- PostgreSQL DWH проходит нагрузочное тестирование на фактическом профиле.
- Stream Processor не является обязательным компонентом на 12 месяцев.
- Полная консолидация операционных платформ находится вне scope этой миграции данных.
- PAN не покидает PCI-контур в открытом виде.
