# Риски миграции

## Метод оценки

Вероятность:

- **Высокая:** событие ожидается хотя бы в одной волне без дополнительных контролей.
- **Средняя:** событие возможно при известной зависимости или сложном сценарии.
- **Низкая:** событие маловероятно, но требует контроля из-за последствий.

Влияние:

- **Критическое:** риск финансовой некорректности, нарушения прав клиента, безопасности или остановки критичного сервиса.
- **Высокое:** срыв migration wave, отчётности или значимый ручной разбор.
- **Среднее:** локальная задержка с контролируемым обходным решением.

Приоритет определяется влиянием, вероятностью и обнаруживаемостью. Критический риск не принимается только потому, что его вероятность низкая.

## Реестр рисков

| ID | Риск | Влияние | Вероятность | Ранний сигнал | Предотвращение | Contingency / fallback | Owner |
|---|---|---|---|---|---|---|---|
| R01 | False merge клиентов FU и RB | Критическое | Высокая | • Рост merge clusters с низким confidence<br><br>• Жалобы на чужие продукты<br><br>• Увеличение ручных split | • Blocking rules<br><br>• Golden sample<br><br>• Порог auto-merge<br><br>• Steward review<br><br>• Provenance | • Остановить auto-merge<br><br>• Split/undo merge<br><br>• Вернуть consumer на source view<br><br>• Повторить затронутую волну | Customer Data Owner |
| R02 | False non-match создаёт дубли golden record | Высокое | Высокая | • Один документ или контакт связан с несколькими global IDs<br><br>• Рост duplicate rate | • Нормализация<br><br>• Несколько match strategies<br><br>• Cross-source profiling | • Объединение после ручной проверки<br><br>• Controlled re-key downstream<br><br>• Новый certified snapshot | MDM Product Owner |
| R03 | Потеря delta между initial load и cutover | Критическое | Средняя | • Gap sequence или CDC LSN<br><br>• Source count растёт без target count<br><br>• Offset discontinuity | • Watermark contract<br><br>• Outbox или CDC<br><br>• Immutable raw<br><br>• Initial/delta overlap | • No-Go<br><br>• Повторный extract от последнего confirmed watermark<br><br>• Replay и reconciliation | Integration Owner |
| R04 | Двойная обработка CDC и доменного события | Высокое | Высокая | • Duplicate source keys<br><br>• Завышенные суммы<br><br>• Повторные golden versions | • Канонический путь по типу данных<br><br>• Idempotency key<br><br>• Dedup в standardized | • Остановить publication<br><br>• Пересобрать партицию из raw<br><br>• Новый batch и snapshot | Data Platform Owner |
| R05 | Неверная версия согласия или KYC | Критическое | Средняя | • Consent state не совпадает с source<br><br>• Missing evidence<br><br>• Нарушение temporal order | • Версия, цель и effective dates<br><br>• Point-in-time tests<br><br>• Break-glass journal | • Блокировать коммуникацию и доступ<br><br>• Вернуться к source consent view<br><br>• Compliance investigation | Compliance Data Owner |
| R06 | Несовместимое изменение source schema | Высокое | Высокая | • Schema Registry rejection<br><br>• DLQ spike<br><br>• Nulls в обязательном поле | • Versioned contracts<br><br>• Compatibility checks<br><br>• Contract tests<br><br>• Change freeze | • Quarantine новой версии<br><br>• Продолжить предыдущий contract<br><br>• Adapter и replay | Producer Owner |
| R07 | Расхождение dual run скрыто разными business dates | Критическое | Средняя | • Агрегаты совпадают только после ручного сдвига<br><br>• Разное EOD window | • Общий business calendar<br><br>• Explicit cutoff<br><br>• Point-in-time manifest | • Продлить dual run<br><br>• Пересчитать затронутый период<br><br>• Не переключать certified report | Finance Data Owner |
| R08 | Финансовые суммы совпадают, но потеряны детальные факты | Критическое | Средняя | • Record count расходится при равных totals<br><br>• Orphan reversal | • Сверка keys, count, debit/credit и reversals<br><br>• Referential checks | • Publication block<br><br>• Replay source interval<br><br>• Новый reconciliation | Accounting Data Owner |
| R09 | Платформа не выдерживает initial load или backfill | Высокое | Средняя | • Daily batch использует большую часть окна<br><br>• Kafka lag<br><br>• Trino queue<br><br>• Small-file growth | • Capacity test<br><br>• Отдельные pools<br><br>• Rate limit<br><br>• Compaction | • Приостановить backfill<br><br>• Уменьшить волну<br><br>• Добавить compute<br><br>• Перенести архив | Platform Owner |
| R10 | Пятиминутная витрина не укладывается в freshness SLO | Среднее | Средняя | • End-to-end latency растёт<br><br>• Late-event rate<br><br>• Micro-batch overlap | • Incremental aggregates<br><br>• Watermarks<br><br>• Отдельный pool<br><br>• Нагрузочный тест | • Показать последний Provisional `as_of`<br><br>• Увеличить окно<br><br>• Trial Stream Processor | BI Product Owner |
| R11 | Неполный lineage архивных данных | Высокое | Высокая | • Dataset без owner/source<br><br>• Неизвестная трансформация<br><br>• Нет checksum | • Archive manifest<br><br>• Discovery<br><br>• Lineage-as-code<br><br>• Quarantine | • Использовать только для research<br><br>• Запретить Certified<br><br>• Запросить повторную поставку | Data Governance Lead |
| R12 | Неизвестный consumer обращается к legacy после cutover | Высокое | Высокая | • Access logs после заявленного switch<br><br>• Service Desk incident<br><br>• Hardcoded endpoint | • Consumer registry<br><br>• Shadow logging<br><br>• Communication<br><br>• Canary | • Остановить decommission<br><br>• Временно восстановить read-only route<br><br>• Мигрировать consumer | Application Owner |
| R13 | Преждевременный decommission нарушает аудит или legal hold | Критическое | Низкая | • Незакрытый hold<br><br>• Отсутствует restore evidence<br><br>• Нет Compliance approval | • G5 gate<br><br>• Retention registry<br><br>• Immutable archive<br><br>• Four-eyes approval | • Остановить удаление<br><br>• Восстановить backup<br><br>• Incident и legal assessment | Records Management Owner |
| R14 | Раскрытие ПДн или PAN при миграции | Критическое | Средняя | • PAN/PII в log или DLQ<br><br>• Массовый export<br><br>• Неверная policy | • Tokenization<br><br>• Field encryption<br><br>• ABAC<br><br>• DLP<br><br>• Synthetic non-prod | • Остановить поток<br><br>• Отозвать credentials<br><br>• Изолировать данные<br><br>• Security response | Security Officer |
| R15 | MDM или Steward queue не успевает за поступлением | Высокое | Средняя | • Backlog выше суточной мощности<br><br>• Match latency растёт<br><br>• Manual review aging | • Incremental matching<br><br>• Blocking strategies<br><br>• Capacity model<br><br>• Обучение Stewards | • Остановить bulk load<br><br>• Уменьшить auto-match scope<br><br>• Добавить workers и Stewards | MDM Product Owner |
| R16 | Недостаток компетенций по Kafka, Iceberg или MDM | Высокое | Средняя | • Повторяющиеся incidents<br><br>• Один key person<br><br>• Runbook не исполняется | • Training<br><br>• Pairing<br><br>• On-call drills<br><br>• Vendor support<br><br>• Knowledge base | • Снизить число параллельных волн<br><br>• Привлечь временную экспертизу<br><br>• Продлить stabilization | Program Director |
| R17 | Rollback не восстанавливает согласованную идентичность | Критическое | Средняя | • Fallback route не тестировался<br><br>• Mapping изменился после recovery point<br><br>• Consumer хранит только global ID | • Rehearsal<br><br>• Mapping journal<br><br>• Recoverable route до T+7<br><br>• Controlled re-key | • Остановить switch<br><br>• Восстановить mapping<br><br>• Replay journal<br><br>• Ручная проверка affected clients | Cutover Manager |
| R18 | Наблюдаемость показывает технический успех при бизнес-дефекте | Высокое | Средняя | • Зелёные jobs при жалобах consumers<br><br>• Нет business controls | • Технические и бизнес-метрики<br><br>• Synthetic customers<br><br>• Owner sign-off<br><br>• Quality gate | • Остановить следующую волну<br><br>• Расширить controls<br><br>• Повторить certification | Data Quality Lead |
| R19 | Карантин растёт без назначенного владельца и исправления | Высокое | Высокая | • Records без Data Steward или Producer Owner<br><br>• Рост aging<br><br>• Повторные причины от одного producer | • Автоматическое назначение по домену<br><br>• Aging SLA<br><br>• Daily triage<br><br>• Dashboard по owner и producer | • Блокировать следующую wave и публикацию affected product<br><br>• Назначить Data Governance Lead временным owner<br><br>• Эскалировать Domain Data Owner | Domain Data Owner |

## Приоритетные риски cutover

Перед Customer MDM Go/No-Go должны быть закрыты или сведены к утверждённому уровню:

1. R01 — false merge.
2. R03 — потеря delta.
3. R05 — consent/KYC.
4. R14 — раскрытие ПДн.
5. R17 — неработающий rollback.

Любой открытый дефект из этой группы приводит к No-Go.

## Контрольные индикаторы программы

| Индикатор | Что показывает | Управленческое действие |
|---|---|---|
| Доля critical flows без owner | Недостаток governance | Не начинать wave без назначения owner |
| Доля source fields без mapping | Неготовность canonical model | Остановить initial load соответствующего набора |
| Reconciliation exceptions aging | Риск накопления необъяснённых различий | Эскалация Data Owner и уменьшение числа волн |
| Rollback rehearsal pass rate | Реальная обратимость | No-Go при провале critical сценария |
| Steward backlog / daily capacity | Пропускная способность MDM | Ограничить bulk matching |
| Batch window utilization | Запас платформы | Приостановить backfill и перераспределить compute |
| Broken lineage critical assets | Аудируемость | Блокировать Certified publication |
| Unknown legacy consumers | Готовность decommission | Сохранить read-only route |

## Принятие остаточного риска

- Критическое влияние принимает только Migration Board с Security, Compliance или Finance в зависимости от области.
- У каждого принятого риска есть owner, срок, компенсационный контроль и критерий пересмотра.
- Формулировка «исправим после cutover» без измеримого ограничения не является митигацией.
- Residual risk пересматривается перед G3, G4 и G5.
- Реализовавшийся риск преобразуется в incident и problem record, а не удаляется из реестра.
