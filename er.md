```mermaid
erDiagram
    USER {
        int id PK
        string role "owner / officer"
        string last_name
        string first_name
        string middle_name
        string phone UK
        bool phone_confirmed
        string password_hash
        datetime created_at
    }

    PHONE_CONFIRMATION {
        int id PK
        int user_id FK
        string code
        datetime expires_at
        datetime confirmed_at
    }

    COUNTRY {
        string iso_code PK
        string name
    }

    DEADLINE_RULE {
        int id PK
        string country_iso_code FK "NULL — для всех стран"
        string category "обычный / ВКС / соотечественник"
        int deadline_days "7, 15, 30, 90"
        string count_from "дата въезда / дата прибытия"
        bool was_registered
        int priority
    }

    NOTIFICATION {
        int id PK
        int owner_id FK
        int deadline_rule_id FK
        string foreigner_last_name
        string foreigner_first_name
        date foreigner_birth_date
        string citizenship_iso_code FK
        string property_address
        date entry_date "дата въезда в РФ"
        date arrival_date "дата прибытия по адресу"
        date deadline_date "рассчитанный срок"
        string status "текущий: черновик / отправлено / на рассмотрении / на доработке / поставлен / отказ"
        datetime sent_at
    }

    STATUS_HISTORY {
        int id PK
        int notification_id FK
        string status
        int changed_by FK
        datetime changed_at
        string comment "причина отказа, пояснение"
    }

    REVISION_REQUEST {
        int id PK
        int notification_id FK
        int officer_id FK
        string comment "что нужно исправить или дослать"
        datetime created_at
        datetime resolved_at "NULL — пока не исправлено"
    }

    ATTACHMENT {
        int id PK
        int notification_id FK
        int revision_request_id FK "NULL — приложено при первичной подаче"
        string doc_type
        string file_key
        datetime uploaded_at
    }

    REGISTRATION_DOCUMENT {
        int id PK
        int notification_id FK, UK
        int officer_id FK
        date issue_date
        string file_key
    }

    USER ||--o{ PHONE_CONFIRMATION : "подтверждает"
    USER ||--o{ NOTIFICATION : "подаёт"
    COUNTRY |o--o{ DEADLINE_RULE : "определяет срок"
    COUNTRY ||--o{ NOTIFICATION : "гражданство"
    DEADLINE_RULE ||--o{ NOTIFICATION : "применён"
    NOTIFICATION ||--o{ ATTACHMENT : "содержит"
    NOTIFICATION ||--o{ STATUS_HISTORY : "история статусов"
    USER ||--o{ STATUS_HISTORY : "меняет статус"
    NOTIFICATION ||--o{ REVISION_REQUEST : "возвращено на доработку"
    USER ||--o{ REVISION_REQUEST : "запрашивает доработку"
    REVISION_REQUEST |o--o{ ATTACHMENT : "дослано по запросу"
    NOTIFICATION ||--o| REGISTRATION_DOCUMENT : "результат"
    USER ||--o{ REGISTRATION_DOCUMENT : "загружает"
```

## Пояснения

Модель описывает данные портала миграционного учёта: пользователей, уведомления о постановке иностранного гражданина на учёт, их рассмотрение сотрудником УМВД и результат.

### Сущности

- **USER** — пользователь портала. Поле `role` различает собственника помещения (`owner`), который подаёт уведомления, и сотрудника УМВД (`officer`), который их рассматривает. Вход выполняется по уникальному номеру телефона.
- **PHONE_CONFIRMATION** — SMS-код подтверждения телефона при регистрации и входе, со сроком действия и отметкой об использовании.
- **COUNTRY** — справочник стран (код ISO и название), используется для гражданства иностранца и для правил расчёта сроков.
- **DEADLINE_RULE** — правило расчёта срока постановки на учёт: категория иностранца (обычный, ВКС, соотечественник), число дней, точка отсчёта (дата въезда или дата прибытия) и признак того, состоял ли иностранец на учёте ранее. Если страна не указана, правило применяется ко всем остальным странам; при совпадении нескольких правил выбирается правило с наибольшим приоритетом.
- **NOTIFICATION** — уведомление о постановке иностранца на учёт: данные иностранца, адрес помещения, даты въезда и прибытия, рассчитанный срок и текущий статус. Список уведомлений пользователя — это история ранее запрошенных услуг.
- **STATUS_HISTORY** — журнал изменений статуса уведомления: какой статус установлен, кем и когда, с комментарием (например, причиной отказа). По нему пользователь просматривает историю статусов. Поле `NOTIFICATION.status` хранит последний статус для быстрого поиска.
- **REVISION_REQUEST** — запрос сотрудника на доработку уведомления: что нужно исправить или дослать. Когда пользователь отправляет доработанное уведомление, запрос закрывается (`resolved_at`).
- **ATTACHMENT** — копии документов, приложенные к уведомлению. Если документ дослан по запросу на доработку, он ссылается на этот запрос.
- **REGISTRATION_DOCUMENT** — документ о постановке на учёт, который сотрудник загружает при положительном решении. У уведомления может быть не больше одного такого документа.

### Связи

- Пользователь подтверждает телефон, подаёт уведомления, а сотрудник меняет их статусы, запрашивает доработку и загружает документы о постановке на учёт.
- Срок в уведомлении рассчитывается по правилу, которое выбирается по гражданству иностранца.
- Каждое изменение статуса уведомления порождает запись в истории статусов.
- Уведомление может несколько раз возвращаться на доработку; документы, досланные в ответ, связаны с конкретным запросом.
