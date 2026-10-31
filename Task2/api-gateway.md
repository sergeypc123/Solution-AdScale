# API Gateway для DSP

## Выбор

Берём **Envoy**.

| Вариант | Плюсы | Минусы |
|---|---|---|
| Envoy | Написан на C++, минимальная задержка. Есть circuit breaking, outlier detection, rate limiting, нативный gRPC, подробные метрики | Сложная конфигурация |
| Kong (на NGINX) | Удобное управление, много плагинов | Плагины на Lua добавляют задержку, часть функций платная |
| NGINX | Быстрый, знакомый | Circuit breaker и продвинутый rate limiting только в платной версии или через самописные модули |
| Spring Cloud Gateway | Удобен в Java-стеке | JVM, GC-паузы, у нас нет Java |

Envoy выигрывает по задержке и по тому, что нужные паттерны надёжности в нём есть из коробки. Сложность конфигурации снимаем шаблонами в Git (configuration as code).

## Маршрутизация

- `POST /openrtb/2.6/{dsp_id}/bid` — запрос на ставку. Маршрутизируется в Ad Selection, а на время миграции часть трафика идёт в монолит.
- `GET /health` — для балансировщика.

На этапе миграции Envoy делит трафик по весам: например, 5% → новый сервис, 95% → монолит, затем веса постепенно меняются. Так новая DSP с первого дня ходит только в новые сервисы, а старые партнёры переключаются постепенно.

## Аутентификация

- mTLS с каждым партнёром: у DSP свой клиентский сертификат.
- Дополнительно API-ключ в заголовке, привязанный к `dsp_id`. Envoy сверяет ключ и `dsp_id` из пути.
- Белый список IP-адресов партнёра.

## Ограничение частоты запросов

- Лимит на партнёра по договору, например 20 000 RPS для новой DSP с допустимым всплеском.
- Локальный rate limit на каждом экземпляре Envoy (без сетевых вызовов) плюс глобальный через сервис лимитов на Redis, для точного подсчёта на весь кластер.
- При превышении лимита отвечаем `HTTP 204 No Content`, то есть no-bid по стандарту OpenRTB, а не ошибкой. Для DSP это нормальный ответ, и штрафа за него нет.

## Автоматический размыкатель

- Circuit breaking: максимум одновременных запросов и ожидающих запросов к upstream. Если лимит превышен, Envoy сразу отвечает no-bid, не нагружая сервис.
- Outlier detection: экземпляр Ad Selection, который вернул 5 ошибок подряд или отвечает медленно, исключается из балансировки на 30 секунд.
- Таймаут на весь запрос 70 мс, после него no-bid.

## Мониторинг времени отклика

- Гистограмма latency (P50, P95, P99) по каждому партнёру и маршруту, в Prometheus.
- Доля no-bid и причины (таймаут, лимит, размыкатель).
- Алерт, если P95 для новой DSP выше 70 мс 5 минут подряд, и критический алерт, если выше 80 мс.
- Трассировка: Envoy создаёт trace id и передаёт его дальше. Часть запросов (например, 1%) уходит в Jaeger.

## Пример конфигурации (фрагмент)

```yaml
route_config:
  virtual_hosts:
    - name: dsp
      domains: ["rtb.adscale.example"]
      routes:
        - match: { prefix: "/openrtb/2.6/" }
          route:
            timeout: 0.070s
            weighted_clusters:
              clusters:
                - name: ad_selection
                  weight: 100
                - name: legacy_monolith
                  weight: 0
clusters:
  - name: ad_selection
    type: STRICT_DNS
    typed_extension_protocol_options:
      envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
        explicit_http_config: { http2_protocol_options: {} }
    circuit_breakers:
      thresholds:
        - max_connections: 2000
          max_pending_requests: 500
          max_requests: 5000
    outlier_detection:
      consecutive_5xx: 5
      interval: 5s
      base_ejection_time: 30s
```