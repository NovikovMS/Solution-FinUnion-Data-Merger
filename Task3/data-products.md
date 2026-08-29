# Доменные data products

## Подход

Data product — не копия таблицы и не произвольная витрина. Это поддерживаемый набор данных с бизнес-владельцем, документированной семантикой, контрактом, качеством, SLA, классификацией и lineage.

Физическая платформа является общей. Домены владеют смыслом и качеством, а Platform Team предоставляет ingestion, Iceberg, Trino, PostgreSQL, Airflow и DataHub.

## Каталог

| Data product | Домен и BIAN | Основные данные | Владелец | Потребители | Интерфейсы | Качество и SLA |
|---|---|---|---|---|---|---|
| Customer 360 | Party / Customer<br><br>**Party Reference Data Directory** | • Глобальный `customer_id`<br><br>• Source links FU/RB<br><br>• Разрешённые профильные атрибуты<br><br>• Сегмент<br><br>• Контакты<br><br>• Согласия<br><br>• Связи с продуктами | Customer Data Owner | • CRM<br><br>• Customer Service<br><br>• Marketing<br><br>• Risk<br><br>• BI | • MDM API для операционных запросов<br><br>• Kafka golden-record events<br><br>• Iceberg curated<br><br>• DWH dimension | • Уникальный глобальный ID<br><br>• Порог completeness KYC<br><br>• Контроль false merge<br><br>• Событие до 5 минут после решения MDM<br><br>• Ежедневная сертификация аналитического среза |
| Accounts & Balances | Accounts<br><br>**Current Account** | • Счёт<br><br>• Договорная связь<br><br>• Валюта и статус<br><br>• Датированный остаток<br><br>• Доступный остаток<br><br>• Связь с клиентом и продуктом | Account Data Owner | • CRM<br><br>• Payments<br><br>• Loans<br><br>• Treasury<br><br>• BI<br><br>• Reporting | • Domain API для актуального состояния<br><br>• Kafka account events<br><br>• Iceberg curated history<br><br>• DWH snapshot fact | • Сверка остатка с ledger<br><br>• Нулевая потеря подтверждённых операций<br><br>• CDC freshness до 5 минут<br><br>• Сертифицированный daily snapshot |
| Payments & Transactions | Payments / Accounting<br><br>**Payment Order Initiation**<br><br>**Financial Accounting** | • Платёжное распоряжение<br><br>• Операции и сторно<br><br>• Счёт<br><br>• Контрагент<br><br>• Канал<br><br>• `source_domain`<br><br>• Business date | Payment Data Owner и Accounting Data Owner | • Customer Service<br><br>• Reconciliation<br><br>• AML<br><br>• Anti-fraud<br><br>• BI<br><br>• Regulatory Reporting | • Kafka payment/transaction events<br><br>• Iceberg curated detail<br><br>• Trino views<br><br>• DWH star schema | • Уникальность source key<br><br>• Баланс дебета и кредита<br><br>• Связность сторно<br><br>• CDC freshness до 5 минут<br><br>• Публикация DWH после финансовой сверки |
| Cards Portfolio | Cards<br><br>**Credit Card**<br><br>**Card Transaction Capture** | • Токенизированная карта<br><br>• Статус и срок действия<br><br>• Авторизации<br><br>• Клиринг<br><br>• Карточные операции<br><br>• Связь со счётом | Card Data Owner | • Customer Service<br><br>• Anti-fraud<br><br>• Finance<br><br>• BI<br><br>• Product Analytics | • Kafka card events<br><br>• Iceberg curated<br><br>• Trino views<br><br>• DWH portfolio mart | • PAN не выходит из PCI-контура<br><br>• Уникальность операции<br><br>• Связь карты со счётом<br><br>• Event freshness до 2 минут<br><br>• Ежедневная сверка клиринга |
| Loan Portfolio | Lending<br><br>**Consumer Loan**<br><br>**Corporate Loan** | • Кредит<br><br>• Договор<br><br>• График<br><br>• Остаток долга<br><br>• Просрочка<br><br>• Ставка<br><br>• Реструктуризация<br><br>• Business date | Loan Data Owner | • Risk<br><br>• Collections<br><br>• Finance<br><br>• BI<br><br>• Regulatory Reporting | • Kafka/CDC loan events<br><br>• Iceberg history<br><br>• Trino views<br><br>• DWH portfolio snapshot | • Непротиворечивый график<br><br>• Сверка выдач и погашений<br><br>• CDC freshness до 15 минут<br><br>• Сертифицированный daily snapshot до отчётного окна |

## Общий контракт

Каждый продукт публикует в DataHub:

- название, назначение и BIAN-домен.
- Data Owner, Data Steward и технический owner.
- входные источники и полный lineage.
- схему, business keys и правила версионирования.
- классификацию полей и политики доступа.
- freshness, availability и quality SLO.
- известные ограничения и допустимые сценарии использования.
- changelog и срок поддержки версии.

## Версионирование

1. Добавление необязательного поля выполняется backward-compatible версией.
2. Переименование, изменение типа или смысла требует новой major-версии.
3. Producer публикует старую и новую версии параллельно в согласованное окно.
4. Consumer регистрируется в DataHub и подтверждает переход.
5. Удаление версии допускается после отсутствия активных consumers и завершения retention window.
6. Исторические таблицы не переписываются без новой версии snapshot и аудита.

## Доступ

| Уровень | Пример | Политика |
|---|---|---|
| Public внутри банка | • Описание продукта<br><br>• Схема<br><br>• Владелец<br><br>• Quality score без данных | Доступ всем сотрудникам через DataHub |
| Internal | • Обезличенные агрегаты<br><br>• Общие продуктовые метрики | Role-based access и утверждённая бизнес-цель |
| Confidential | • Псевдонимизированные клиентские факты<br><br>• Счета<br><br>• Платежи<br><br>• Кредиты | ABAC по цели, маскирование, аудит и ограниченный export |
| Restricted | • Прямые идентификаторы<br><br>• Документы<br><br>• PAN внутри PCI-контура<br><br>• AML-признаки | Отдельные зоны, field-level encryption, MFA, four-eyes для массового доступа |

## Ответственность

- Data Owner утверждает смысл, SLO, доступ и допустимое использование.
- Data Steward контролирует определения, качество и разбор инцидентов.
- Producer исправляет причину дефекта в авторитетном процессе.
- Platform Team обеспечивает техническую доступность общей платформы.
- Consumer соблюдает назначение продукта и не создаёт локальное альтернативное определение ключевых метрик.

## Почему это не полный Data Mesh

Команды используют доменное владение и data as a product, но не создают отдельную инфраструктуру в каждом домене. Общие стандарты идентификаторов, безопасности, каталогизации, ingestion и хранения централизованы. Это снижает стоимость и риск расхождения терминов, пока объединённая организация формирует устойчивые доменные команды.
