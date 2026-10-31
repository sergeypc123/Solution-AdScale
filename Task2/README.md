# Task2. Сервис ставок и интеграция с DSP

Здесь спроектирован самый критичный компонент: сервис ставок (Bidding Service), который первым выносится из монолита (см. ADR-002 в Task1). Также описан API-слой для новой DSP.

Главное требование: ответ DSP не дольше 80 мс при 18 000 RPS, а через год при 50 000 RPS. Сбой одной из зависимостей не должен ронять аукцион.

Файлы:

- [bidding-service.md](bidding-service.md) — границы сервиса ставок, API, зависимости, модель данных;
- [interaction.md](interaction.md) — протоколы взаимодействия и обоснование выбора, бюджет latency;
- [api-gateway.md](api-gateway.md) — API Gateway для DSP;
- [reliability.md](reliability.md) — паттерны надёжности;
- [diagrams/bid-request-sequence.puml](diagrams/bid-request-sequence.puml) — sequence-диаграмма обработки запроса на ставку;
- [diagrams/bidding-components.puml](diagrams/bidding-components.puml) — компоненты сервиса ставок.

Коротко: DSP приходит через Envoy по OpenRTB, внутри горячего пути используется gRPC, данные для аукциона берутся только из Redis, события уходят в Kafka асинхронно. Если ответ не успевает уложиться в бюджет времени, отдаём no-bid, а не тайм-аут.