# Task3. Данные, масштабирование и отказоустойчивость

Стратегия работы с данными после перехода на сервисы. Главные идеи:

- у каждого сервиса своё хранилище, общая база монолита постепенно разбирается;
- горячий путь аукциона работает только с Redis;
- события идут через Kafka, а статистика и аналитика живут в ClickHouse, отдельно от транзакционных данных;
- финансовые данные защищены сильнее всего: синхронная репликация, RPO = 0.

Файлы:

- [database-strategy.md](database-strategy.md) — какая база у какого сервиса и почему;
- [scaling.md](scaling.md) — репликация, шардирование, read replicas, CQRS;
- [caching.md](caching.md) — что кешируем, TTL, инвалидация, прогрев;
- [event-streaming.md](event-streaming.md) — топики Kafka, схема событий, группы потребителей, хранение;
- [failover.md](failover.md) — RPO/RTO, резервное копирование, 12-факторные принципы;
- [diagrams/data-architecture.puml](diagrams/data-architecture.puml) — хранилища и потоки данных;
- [diagrams/event-flow.puml](diagrams/event-flow.puml) — топики Kafka и потребители.