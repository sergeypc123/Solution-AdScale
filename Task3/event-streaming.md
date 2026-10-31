# Потоковая обработка событий (Kafka)

## Топики

| Топик | Что внутри | Producer | Партиций | Ключ | Хранение |
|---|---|---|---|---|---|
| `auction-results` | Результаты аукциона (ставка или no-bid) | Bidding | 96 | `request_id` | 3 дня |
| `impressions` | Показы | Tracking | 48 | `campaign_id` | 7 дней |
| `clicks` | Клики | Tracking | 12 | `campaign_id` | 7 дней |
| `campaign-changes` | Изменения кампаний | Campaign | 12 | `campaign_id` | compacted (последнее состояние каждой кампании хранится всегда) |
| `budget-events` | Остановка кампании, изменение лимита | Finance | 12 | `campaign_id` | compacted |
| `balance-changes` | Изменения балансов | Finance (через outbox) | 6 | `advertiser_id` | 30 дней |
| `*.dlq` | События, которые не удалось обработать | Потребители | 3 | исходный | 14 дней |

Количество партиций берём с запасом под 50 000 RPS: одна партиция уверенно пропускает несколько тысяч сообщений в секунду, а увеличить их число потом можно, но при этом сломается порядок по ключу. Поэтому лучше сразу заложить больше.

Настройки для всех топиков: replication factor 3, `min.insync.replicas = 2`, у producer `acks = all` и `enable.idempotence = true`, сжатие lz4.

## Схема событий

Выбираем **Avro** со Schema Registry.

- Avro компактнее JSON в несколько раз, а при десятках тысяч событий в секунду это заметная экономия сети и диска.
- Schema Registry проверяет совместимость: producer не сможет выпустить несовместимое изменение и сломать потребителей. Режим совместимости — BACKWARD: новые поля добавляются только со значением по умолчанию.
- JSON проще отлаживать, но схема в нём не контролируется, и ошибки всплывают уже в проде.

Пример схемы показа:

```json
{
  "type": "record",
  "name": "Impression",
  "namespace": "com.adscale.events",
  "fields": [
    {"name": "event_id", "type": "string"},
    {"name": "event_time", "type": {"type": "long", "logicalType": "timestamp-millis"}},
    {"name": "request_id", "type": "string"},
    {"name": "dsp_id", "type": "string"},
    {"name": "campaign_id", "type": "string"},
    {"name": "creative_id", "type": "string"},
    {"name": "reservation_id", "type": ["null", "string"], "default": null},
    {"name": "price_micros", "type": "long"},
    {"name": "currency", "type": "string"},
    {"name": "user_id", "type": ["null", "string"], "default": null},
    {"name": "region", "type": "string"}
  ]
}
```

Деньги храним в микроединицах (`long`), а не в `double`, чтобы не было ошибок округления.

## Группы потребителей

| Группа | Читает | Что делает |
|---|---|---|
| `stats-aggregator` | impressions, clicks, auction-results | Пишет события пачками в ClickHouse |
| `analytics-loader` | impressions, clicks, campaign-changes | Загружает данные в базу аналитики, обновляет справочники |
| `finance-billing` | impressions, clicks | Идемпотентно списывает деньги, сверяет резервы бюджета |
| `finance-reconciler` | auction-results | Возвращает неиспользованные резервы в бюджет |
| `profile-updater` | impressions, clicks | Обновляет профили пользователей в Redis |
| `bidding-snapshot-{instance}` | campaign-changes, budget-events | Каждый экземпляр Bidding держит свою группу, чтобы получать все изменения |

Каждая группа читает в своём темпе. Если статистика отстала или упала, это не задевает финансы и аукцион. Отставание (consumer lag) мониторим, алерт для `finance-billing` при lag выше 1 минуты.

Финансы коммитят offset только после успешной записи транзакции в базу. Повторная доставка возможна и безопасна, потому что уникальный индекс по `event_id` не даст списать дважды.

## Политика хранения

- События показов и кликов хранятся 7 дней. Этого хватает, чтобы перечитать их после аварии любого потребителя или пересчитать статистику за неделю.
- Долгосрочно события хранятся в ClickHouse (сырые данные 90 дней, агрегаты 3 года) и в архиве в объектном хранилище.
- Compacted-топики хранят последнее состояние, из них можно восстановить кеш с нуля.