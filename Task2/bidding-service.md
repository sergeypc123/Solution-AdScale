# Сервис ставок (Bidding Service)

## Границы

Сервис ставок отвечает на один вопрос: сколько мы готовы заплатить за этот показ и какая кампания его получает.

Что входит:
- приём кандидатов от Ad Selection;
- применение бизнес-правил: минимальная цена (floor price), стратегия ставки кампании, ограничения частоты показов (frequency cap), пейсинг бюджета;
- расчёт ставки по каждому кандидату и выбор победителя;
- резервирование бюджета под ставку;
- отправка результатов аукциона в Kafka.

Что не входит:
- определение пользователя и подбор кандидатов (Ad Selection);
- формирование разметки креатива (Delivery);
- окончательное списание денег и балансы (Finance);
- создание и редактирование кампаний (Campaign).

Сервис stateless: всё состояние хранится в Redis. Поэтому экземпляры можно добавлять и убирать без координации.

## API

Внутренний API на gRPC. Наружу сервис не открыт, DSP общается только с API Gateway.

```protobuf
service BiddingService {
  // Провести аукцион по кандидатам и вернуть победителя
  rpc Bid(BidRequest) returns (BidResponse);
}

message BidRequest {
  string request_id = 1;        // id запроса OpenRTB, для трассировки и идемпотентности
  string dsp_id = 2;            // от какого партнёра пришёл запрос
  string impression_id = 3;
  string user_id = 4;           // может быть пустым
  double floor_price = 5;       // минимальная цена из запроса DSP, CPM
  string currency = 6;
  repeated Candidate candidates = 7;
  int32 timeout_ms = 8;         // сколько времени осталось у запроса
}

message Candidate {
  string campaign_id = 1;
  string creative_id = 2;
  double relevance_score = 3;   // оценка от Ad Selection
}

message BidResponse {
  string request_id = 1;
  oneof result {
    Bid bid = 2;
    NoBid no_bid = 3;
  }
}

message Bid {
  string campaign_id = 1;
  string creative_id = 2;
  double price = 3;             // CPM
  string reservation_id = 4;    // id резерва бюджета
}

message NoBid {
  enum Reason { NO_CANDIDATES = 0; BELOW_FLOOR = 1; NO_BUDGET = 2; TIMEOUT = 3; DEPENDENCY_DOWN = 4; }
  Reason reason = 1;
}
```

Плюс стандартные эндпоинты для эксплуатации: health check (liveness/readiness) и метрики для Prometheus.

## Зависимости

| Зависимость | Зачем | Тип | Что если недоступна |
|---|---|---|---|
| Redis Cluster | Данные кампаний, бюджеты, frequency cap | Синхронно, в горячем пути | Работаем на локальном снимке кампаний, бюджет не резервируем и ставим только кампаниям с большим запасом бюджета, либо no-bid. См. reliability.md |
| Kafka | Отправка результатов аукциона | Асинхронно | Буфер в памяти. При переполнении события пишутся в локальный файл и досылаются потом. Аукцион не останавливается |
| Campaign Service | Источник данных кампаний | Не напрямую: через Redis и топик campaign-changes | Аукцион работает на последних данных в Redis |
| Finance Service | Окончательное списание | Не напрямую: читает события из Kafka | На аукцион не влияет |

Вызывают сервис: Ad Selection (gRPC). На первом этапе, пока Ad Selection в монолите, сервис вызывает монолит через тот же gRPC-контракт.

## Модель данных

Основное хранилище — Redis. Данные кладут Campaign Service и сам Bidding.

| Ключ | Тип | Содержимое | Кто пишет | TTL |
|---|---|---|---|---|
| `campaign:{id}` | Hash | статус, стратегия ставки, max CPM, расписание, ссылки на креативы | Campaign Service | без TTL, обновляется по событию |
| `budget:{campaign_id}:{date}` | String (число) | остаток дневного бюджета в микроединицах валюты | Finance задаёт лимит, Bidding уменьшает атомарно | 48 ч |
| `reservation:{reservation_id}` | Hash | campaign_id, сумма, время | Bidding | 15 мин |
| `freq:{user_id}:{campaign_id}` | String (число) | сколько раз показали пользователю | Bidding | равен окну frequency cap (например, 24 ч) |
| `pacing:{campaign_id}` | Hash | плановый и фактический расход по часам | Finance/Statistics | 48 ч |

Резервирование бюджета делается Lua-скриптом в Redis: проверить остаток, уменьшить и создать резерв одной атомарной операцией. Если показ не состоялся, резерв истекает, а Finance по событиям возвращает сумму в бюджет.

Событие в Kafka, топик `auction-results`:

```json
{
  "event_id": "uuid",
  "request_id": "string",
  "dsp_id": "string",
  "timestamp": "2026-10-26T12:00:00.123Z",
  "result": "BID | NO_BID",
  "campaign_id": "string",
  "creative_id": "string",
  "price_cpm": 1.25,
  "reservation_id": "string",
  "no_bid_reason": null
}
```

## Нефункциональные требования к сервису

- Время ответа: P95 не выше 20 мс, P99 не выше 30 мс.
- Пропускная способность: 18 000 RPS на этапе 1 с запасом 30%, далее 50 000 RPS за счёт добавления экземпляров.
- Не меньше 3 экземпляров в каждой зоне доступности.
- Автомасштабирование по загрузке CPU и по P95 latency.