+++
date = '2026-01-18'
draft = false
title = 'Audit: практичное audit-логирование для Go'
source = 'https://github.com/w0rng/audit'
tags = ['go', 'audit', 'logging', 'slog']
+++

___
Версия на английском: [dev.to](https://dev.to/w0rng/audit-practical-audit-logging-for-go-5ae9)
___

Audit-логирование почти всегда появляется позже, чем нужно. Сначала хватает обычных логов, потом внезапно становится важно понять, кто именно поменял статус заказа, почему у пользователя пропала роль или откуда взялись странные данные в системе.

В большинстве проектов audit либо размазывается по бизнес-коду, либо пытается жить поверх обычного логгера без четкой модели. Я вынес эту задачу в отдельную библиотеку: **audit**.

Репозиторий: https://github.com/w0rng/audit

## Что делает audit

Библиотека решает конкретную задачу: хранить историю изменений сущностей.

На уровне модели это выглядит так:

* сущность идентифицируется строковым ключом (`order:12345`, `user:42`)
* каждое событие имеет автора, описание и timestamp
* изменения фиксируются по полям
* значения могут быть видимыми или скрытыми

Никакой магии, рефлексии и автоматических diff. Все изменения описываются явно.

## Базовый сценарий

Простейший пример: жизненный цикл заказа.

```go
logger := audit.New()

logger.Create(
    "order:12345",
    "john.doe",
    "Order created",
    map[string]audit.Value{
        "status":        audit.PlainValue("pending"),
        "total":         audit.PlainValue(99.99),
        "payment_token": audit.HiddenValue(),
    },
)
```

Чувствительные данные явно помечаются как скрытые. Они участвуют в событии, но не попадают в логи в открытом виде.

Дальнейшие изменения выглядят так же декларативно:

```go
logger.Update(
    "order:12345",
    "warehouse.system",
    "Order shipped",
    map[string]audit.Value{
        "status":          audit.PlainValue("shipped"),
        "tracking_number": audit.PlainValue("TRK123456789"),
    },
)
```

## История изменений и события

Библиотека предоставляет два уровня чтения данных.

Полная история изменений сущности:

```go
logs := logger.Logs("order:12345")
```

Каждая запись содержит список полей с `from` и `to`, автора и описание. Для вышеупомянутых логов данные следующие:
```json
[
  {
    "Fields": [
      { "Field": "status", "From": null, "To": "pending" },
      { "Field": "total", "From": null, "To": 99.99 },
      { "Field": "payment_token", "From": "***", "To": "***" }
    ],
    "Description": "Order created",
    "Author": "john.doe",
    "Timestamp": "2026-01-18T17:06:29.126233926+07:00"
  },
  {
    "Fields": [
      { "Field": "status", "From": "pending", "To": "shipped" },
      { "Field": "tracking_number", "From": null, "To": "TRK123456789" }
    ],
    "Description": "Order shipped",
    "Author": "warehouse.system",
    "Timestamp": "2026-01-18T17:06:29.126239744+07:00"
  }
]
```

Если нужны более легковесные события, можно работать через `Events` и фильтровать их по полям:

```go
events := logger.Events("order:12345", "status")
```

Это удобно для построения таймлайнов или агрегатов, например цепочки статусов заказа:
```json
[
  {
    "Timestamp": "2026-01-18T17:06:53.12607001+07:00",
    "Action": "create",
    "Author": "john.doe",
    "Description": "Order created",
    "Payload": { "status": { "Data": "pending",  "Hidden": false }}
  },
  {
    "Timestamp": "2026-01-18T17:06:53.126075226+07:00",
    "Action": "update",
    "Author": "warehouse.system",
    "Description": "Order shipped",
    "Payload": { "status": { "Data": "shipped",  "Hidden": false }}
  }
]
```

## Кастомное хранилище

`audit` не привязывается к способу хранения данных. Для этого используется интерфейс `Storage`.

[Пример простой реализации](https://github.com/w0rng/audit/blob/main/examples/custom_storage/main.go), которая сохраняет события в JSON-файл, есть в репозитории. Такое хранилище подключается через options pattern:

```go
storage := NewJSONFileStorage("audit_events.json")
logger := audit.New(audit.WithStorage(storage))
```

Это может быть файл, база данных, Kafka или любой другой backend. Core-библиотека об этом не знает.

## Интеграция с slog

Отдельный пакет `audit/slog` позволяет подключить audit к стандартному `log/slog`.

Handler извлекает audit-события из структурированных логов:

```go
handler := auditslog.NewHandler(auditLogger, auditslog.HandlerOptions{
    Handler: slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }),
    KeyExtractor: auditslog.AttrExtractor(auditslog.AttrEntity),
    ShouldAudit: func(record slog.Record) bool {
        return record.Level >= slog.LevelInfo
    },
})
```

Обычный лог автоматически становится audit-событием:

```go
slog.Info(
    "User account created",
    auditslog.AttrEntity, "user:123",
    auditslog.AttrAction, "create",
    auditslog.AttrAuthor, "admin",
    "email", "alice@example.com",
    "role", "editor",
)
```

Если в записи нет `entity` или уровень не проходит фильтр, событие не попадет в audit.  
Плюс таких логов в том, что они будут видны и в системах сбора логов, и с ними можно работать как с аудит событиями через `.Logs`, `.Events`.

## Итог

`audit` это небольшой, изолированный слой для audit-логирования:

* явная модель данных
* контроль над чувствительными полями
* расширяемое хранилище
* нативная интеграция со `slog`

Подходит для Go-сервисов, где audit важен как инфраструктурный слой, а не формальность.
