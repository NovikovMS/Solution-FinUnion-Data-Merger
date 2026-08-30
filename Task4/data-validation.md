# Валидация данных при миграции

## Цель

Валидация подтверждает не только доставку всех записей, но и сохранение бизнес-смысла, истории, связей и качества после преобразования.

Равенство `source count = target count` недостаточно. Одинаковое количество записей может скрывать:

- неверное сопоставление клиентов.
- потерю обязательных атрибутов.
- изменение суммы или валюты.
- нарушение связей между клиентом, договором и счётом.
- применение неправильной версии справочника.
- потерю истории и temporal semantics.
- неверную обработку согласия, KYC или сторно.
- появление дублей при повторной загрузке.

Диаграмма: [`diagrams/data-validation-flow.mmd`](diagrams/data-validation-flow.mmd).
![data-validation-flow.png](diagrams/data-validation-flow.png)

## Принципы

- Проверки определяются до initial load и версионируются вместе с data contract.
- Source baseline фиксируется до преобразования.
- Critical validation выполняется автоматически для каждой migration wave.
- Business Data Owner утверждает пороги и допустимые исключения.
- Data Steward анализирует дефекты и владеет карантинной очередью.
- Producer Owner исправляет причину в авторитетном источнике.
- Проверка выполняется на уровне записей, агрегатов и распределений.
- Исправление не перезаписывает immutable raw.
- Повторный запуск использует тот же source cut, rule version и reference-data version.
- Результат валидации сохраняется как evidence для G2, G3, G4 и G5.

## Уровни валидации

| Уровень | Что проверяется | Примеры | Возможный результат |
|---|---|---|---|
| Техническая доставка | Полнота файлов, сообщений и диапазонов CDC | • Record count<br><br>• File checksum<br><br>• Kafka offsets<br><br>• CDC LSN<br><br>• Watermark gap<br><br>• Повторная доставка | Pass, replay или остановка загрузки |
| Структура | Соответствие схеме и типам | • Обязательные поля<br><br>• Тип данных<br><br>• Длина<br><br>• Precision/scale<br><br>• Encoding<br><br>• Schema version | Pass или quarantine |
| Синтаксис | Формальная корректность значения | • Формат даты<br><br>• Телефон<br><br>• Email<br><br>• Идентификатор<br><br>• Код валюты<br><br>• MIME type | Pass, normalize или quarantine |
| Семантика | Допустимость значения в бизнес-контексте | • Сумма больше нуля для заданного типа операции<br><br>• Допустимый статус<br><br>• Возраст и дата рождения<br><br>• Версия продукта действует на дату договора | Pass, business review или block |
| Справочники | Соответствие RDM и исторической версии | • Local-to-canonical mapping<br><br>• Период действия кода<br><br>• Отсутствие unknown для mandatory code | Pass или quarantine до mapping |
| Уникальность | Отсутствие технических и бизнес-дублей | • Source key<br><br>• Idempotency key<br><br>• Global customer ID<br><br>• Номер договора в области действия | Pass, dedup или Steward review |
| Ссылочная целостность | Существование связанных сущностей | • Счёт → клиент<br><br>• Договор → продукт<br><br>• Карта → счёт<br><br>• Кредит → договор<br><br>• Сторно → исходная операция | Pass, quarantine или publication block |
| Temporal consistency | Корректность порядка и периодов | • `valid_from < valid_to`<br><br>• Нет пересечения SCD-версий<br><br>• Статусы идут в разрешённом порядке<br><br>• Business date согласована | Pass, reorder, replay или block |
| Финансовая сверка | Сохранение учётного результата | • Debit = credit<br><br>• Суммы по валюте<br><br>• Остаток и обороты<br><br>• Выдачи и погашения<br><br>• Сторно | Pass или hard block |
| MDM-качество | Корректность identity resolution | • False merge<br><br>• False non-match<br><br>• Source links<br><br>• Match confidence<br><br>• Survivorship provenance | Auto-accept, Steward review или No-Go |
| Privacy и compliance | Законность и минимизация обработки | • Consent version<br><br>• KYC completeness<br><br>• PAN tokenization<br><br>• PII masking<br><br>• Retention и legal hold | Pass, security quarantine или hard block |
| Аналитическая эквивалентность | Сохранение показателей и распределений | • Business KPI<br><br>• Суммы по продуктам<br><br>• Null rate<br><br>• Cardinality<br><br>• Distribution drift<br><br>• Top categories | Pass, investigation или продление dual run |
| Бизнес-приёмка | Корректность реальных сценариев | • Customer 360<br><br>• Выписка<br><br>• Портфель<br><br>• Регуляторный отчёт<br><br>• Drill-down до source | Sign-off или No-Go |

## Валидация по этапам миграции

### 1. До миграции: baseline

Для каждого source dataset фиксируются:

- точный source cut и business date.
- количество записей и distinct business keys.
- null rate обязательных и ключевых атрибутов.
- duplicate rate.
- минимумы, максимумы и допустимые диапазоны.
- распределения по статусам, продуктам, валютам и источникам.
- количество orphan records.
- финансовые суммы и остатки.
- версии RDM, MDM rules и схем.
- известные дефекты, принятые Data Owner.

Baseline подписывают Data Owner, Data Steward и Producer Owner. Известный дефект источника не должен автоматически считаться дефектом миграции, но переносится в quality debt register.

### 2. При приёме в raw

Проверяется техническая неизменность:

- file checksum и размер.
- число сообщений или строк.
- последовательность offsets, LSN и watermarks.
- source ID, ingestion time и correlation ID.
- отсутствие необъяснимой потери или двойной поставки.
- возможность повторного чтения.

Raw сохраняет исходный payload даже при ошибке качества. Структурно нечитаемый объект изолируется, но не удаляется.

### 3. В standardized

Проверяются:

- schema contract.
- типы и форматы.
- обязательные поля.
- канонические идентификаторы.
- RDM mapping.
- дедупликация по source key и версии.
- нормализация времени, валют и кодировок.
- сохранение provenance до исходного payload.

Ошибка отдельной записи направляет её в доменный карантин. Массовая ошибка схемы останавливает обработку всей несовместимой версии.

### 4. В curated

Проверяются бизнес-правила:

- уникальность и жизненный цикл сущности.
- ссылки между сущностями.
- temporal consistency.
- MDM match/merge.
- financial reconciliation.
- consent, KYC и security classification.
- соответствие data-product contract.

Curated dataset получает статус `Provisional`, пока не завершены все проверки, необходимые для `Certified`.

### 5. В DWH и BI

Проверяются:

- гранулярность факта.
- SCD point-in-time join.
- отсутствие необъяснимого `Unknown` dimension key.
- соответствие curated snapshot и DWH batch.
- бизнес-метрики и аналитические распределения.
- drill-down отчёта до source record.
- воспроизводимость по `report_snapshot_id`.

Сравнение выполняется минимум для двух последовательных отчётных периодов до отключения legacy BI.

### 6. После cutover

Проверяются:

- новые записи из каждого источника.
- late events и replay.
- freshness и quarantine aging.
- жалобы и business exceptions.
- отсутствие обращений к retired route.
- стабильность качества относительно baseline.

Успешный cutover не закрывает дефекты автоматически. Нерешённые исключения переходят в BAU backlog с владельцем и SLA.

## Проверки по ключевым сущностям

| Сущность | Критические проверки качества | Hard gate |
|---|---|---|
| Клиент | • Уникальность source link и global ID<br><br>• Формат и непротиворечивость идентификаторов<br><br>• Полнота KYC<br><br>• Контакты<br><br>• False merge и false non-match<br><br>• Survivorship provenance | • Нет подтверждённого critical false merge<br><br>• Нет потерянной source record<br><br>• Все ambiguous records имеют Steward decision или карантин |
| Согласие | • Версия текста<br><br>• Цель<br><br>• Канал и время<br><br>• Срок<br><br>• Отзыв<br><br>• Evidence | Ни одно отозванное или просроченное согласие не становится активным |
| Счёт | • Уникальность ключа<br><br>• Валюта и статус<br><br>• Связь с клиентом и договором<br><br>• Остаток на business date | Нет orphan account.<br><br>Остатки и контрольные суммы совпадают |
| Договор | • Стороны<br><br>• Версия продукта<br><br>• Период действия<br><br>• Подписанный документ<br><br>• Жизненный цикл статуса | Нет active contract без клиента, продукта и действующей версии условий |
| Карта | • Уникальность токена<br><br>• Связь со счётом<br><br>• Статус и срок<br><br>• Авторизация и клиринг<br><br>• Отсутствие открытого PAN | PAN не покидает PCI-контур.<br><br>Нет active card без valid account link |
| Кредит | • Договор и счёт погашения<br><br>• График<br><br>• Остаток долга<br><br>• Выдачи и погашения<br><br>• Просрочка на business date | Остаток и график проходят кредитную и финансовую сверку |
| Платёж | • Source и idempotency keys<br><br>• Сумма и валюта<br><br>• Реквизиты<br><br>• Статусная цепочка<br><br>• Контрагент | Нет потерянного или двойного подтверждённого платежа |
| Операция | • Уникальность source key<br><br>• Debit/credit<br><br>• Business date<br><br>• Сторно<br><br>• Связь с платёжным или продуктовым фактом | Ненулевое финансовое расхождение блокирует Certified publication |
| Документ | • Checksum<br><br>• MIME type<br><br>• Антивирусная проверка<br><br>• Версия<br><br>• Связь с объектом<br><br>• Retention | Повреждённый, заражённый или несвязанный документ не публикуется |
| Справочное значение | • Канонический код<br><br>• Local mapping<br><br>• Период действия<br><br>• Непересекающиеся версии | Mandatory local code не публикуется без действующего canonical mapping |

## Сравнение source и target

Для каждого набора формируется validation manifest:

| Группа контроля | Source | Target | Сравнение |
|---|---|---|---|
| Объём | • Total rows<br><br>• Distinct keys<br><br>• Rows by partition | Те же показатели после учёта rejected и quarantined | `source = loaded + rejected + quarantined` |
| Контрольная сумма | Hash файла или детерминированный hash набора ключевых полей | Hash до и после преобразования по правилам mapping | Объяснимое отличие только для утверждённой нормализации |
| Полнота | Null и populated counts по critical fields | Те же counts и процент | Не хуже baseline без утверждённого исключения |
| Уникальность | Duplicate business keys | Duplicates после dedup и MDM | Не создаются новые необъяснимые дубли |
| Справочники | Частота local codes | Canonical codes и mapping version | Каждый mandatory code сопоставлен |
| Связи | Orphans по relationship | Orphans после global mapping | Новые orphan records запрещены |
| Время | Min/max event time, business date и version | Те же границы и temporal order | Нет потерянного периода или пересечения версий |
| Финансы | Count, amount, debit, credit, balances по валюте и business date | Те же показатели | Точное равенство для certified финансовых данных |
| Распределения | Status, product, channel, geography и amount bands | Те же распределения | Drift объяснён mapping rule или дефектом |
| Безопасность | PII/PAN classification и consent state | Маскирование, tokenization и policy | Не возникает более открытый класс доступа |

`Rejected` и `quarantined` не скрываются из формулы. Для каждой такой записи указываются причина, владелец и влияние на publication.

## Severity и решение

| Severity | Пример | Результат |
|---|---|---|
| Critical | • Финансовое расхождение<br><br>• Critical false merge<br><br>• Активировано отозванное согласие<br><br>• Раскрыт PAN<br><br>• Потеря delta | No-Go и Publication Block |
| High | • Orphan critical entity<br><br>• Неизвестный mandatory RDM code<br><br>• Нарушена история статусов | Quarantine affected records.<br><br>Go возможен только если Data Owner доказал изоляцию от cutover scope |
| Medium | • Необязательный атрибут хуже baseline<br><br>• Допустимое late event<br><br>• Некритичный descriptive mismatch | Исправление в согласованный SLA.<br><br>Data Owner принимает ограничение |
| Low | • Косметическое описание<br><br>• Неиспользуемый legacy attribute | Quality debt backlog с owner |

Порог устанавливается до запуска волны. Изменение порога после получения результата требует отдельного решения Data Owner и записи в audit.

## Решение Pass / Conditional Pass / Fail

### Pass

- Все critical и high checks пройдены.
- Source-target reconciliation замыкается.
- Карантин не содержит записей, влияющих на hard gate.
- Data Steward и Data Owner подписали результат.

### Conditional Pass

- Есть только изолированные medium/low defects.
- Для каждой записи или группы назначены owner и SLA.
- Consumer уведомлён об ограничении.
- Дефект не меняет финансовый, правовой или security-смысл.

### Fail

- Есть critical defect.
- Не замыкается формула полноты.
- Нет объяснения distribution drift.
- Записи карантина не имеют владельца.
- Невозможно воспроизвести проверку.

`Fail` означает No-Go для затронутой migration wave или data product.

## Автоматическая и ручная проверка

Автоматически проверяются:

- counts, hashes, duplicates и nulls.
- schema, types, formats и ranges.
- RDM mapping и referential integrity.
- temporal constraints.
- финансовые суммы и балансы.
- data drift и SLO.

Ручная проверка нужна для:

- неоднозначного MDM match.
- качества ФИО и адресов.
- юридического смысла согласия и договора.
- необычного distribution drift.
- business acceptance отчёта.

Ручная выборка использует риск-ориентированный подход:

- записи с низким match confidence.
- крупные финансовые суммы.
- редкие статусы и продукты.
- records после normalization.
- late и replayed events.
- записи, попавшие в карантин повторно.

Размер выборки и допустимый defect rate утверждаются Data Owner до G2.

## Ответственность

| Роль | Ответственность за валидацию |
|---|---|
| Data Owner | Утверждает правила, пороги, hard gates и итоговый Pass |
| Data Steward | Анализирует профиль, triage дефектов, управляет карантином и evidence |
| Producer Owner | Подтверждает source baseline и исправляет root cause |
| Migration Data Lead | Обеспечивает одинаковое выполнение правил между волнами |
| Data Quality Team | Реализует проверки, dashboards и versioning rules |
| Reconciliation Team | Выполняет финансовую и межсистемную сверку |
| Security / Compliance | Подтверждает privacy, consent, KYC, PCI и retention controls |
| Consumer Owner | Выполняет business acceptance и подтверждает пригодность |
| Platform Team | Обеспечивает исполнение, хранение результатов и replay |

## Evidence

Для каждой волны сохраняются:

- идентификатор source snapshot и watermarks.
- версии схем, mapping, RDM и quality rules.
- source и target profiles.
- результаты каждой проверки.
- список rejected и quarantined records с owners.
- объяснение каждого принятого расхождения.
- replay history.
- business sign-off.
- итоговый статус Pass, Conditional Pass или Fail.

Evidence регистрируется в DataHub и связывается с migration wave, data product, DWH batch и Iceberg snapshot.
