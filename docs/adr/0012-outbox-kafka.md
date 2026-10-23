# ADR-0012: Outbox Pattern для публикации событий в Kafka

- Status: Accepted
- Date: 2026-02-14

## Context
Нужно публиковать событие OrderCreated в Kafka при создании заказа.
Требования:
- событие не должно теряться при падении сервиса
- дубли допустимы (consumer идемпотентный)
  Проблема:
- "DB commit + send to Kafka" не атомарно → возможна рассинхронизация

## Decision
Используем Transactional Outbox:
- в одной транзакции с заказом пишем запись в таблицу outbox
- отдельный publisher (фоновый процесс/worker) читает outbox и публикует в Kafka
- после успешной публикации помечаем запись как SENT
- дедупликация по event_id на стороне consumer

## Alternatives considered
1) Kafka transactions + exactly-once — сложнее, выше операционные риски, не нужно по требованиям.
2) Публиковать напрямую после commit — риск потери события при краше.
3) Two-phase commit — слишком тяжело.

## Consequences
Pros:
- гарантируем "не потерять событие" при сбоях сервиса
- масштабируем publisher отдельно

Cons / Risks:
- появляется таблица outbox и фоновая обработка
- нужно следить за лагом outbox (метрики/алерты)
  Observability:
- outbox lag, publish rate, retry rate, age of oldest message