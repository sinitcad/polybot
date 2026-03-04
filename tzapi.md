Вот полная техническая карта работы с Polymarket, особенно для BTC 15-минутных рынков.

***

## Архитектура API

Polymarket использует гибридную систему **CLOB (Central Limit Order Book)** — матчинг ордеров происходит off-chain, а расчёты on-chain через смарт-контракт на Polygon. Есть три основных REST API и два слоя WebSocket. [docs.polymarket](https://docs.polymarket.com/trading/overview)

### REST эндпоинты

| API | Базовый URL | Назначение |
|---|---|---|
| **CLOB API** | `https://clob.polymarket.com` | Ордербук, размещение ордеров, цены |
| **Gamma API** | `https://gamma-api.polymarket.com` | Метаданные рынков, поиск по slug |
| **Data API** | `https://data-api.polymarket.com` | Исторические данные, трейды |

Для работы с ценами по конкретному токену используются эндпоинты CLOB: [docs.polymarket](https://docs.polymarket.com/market-data/overview)
- `GET /price` — цена одного токена
- `GET /book` — полный ордербук
- `GET /prices-history` — исторические цены
- `GET /midpoint` и `GET /spread`

***

## WebSocket — что и когда слушать

Polymarket предоставляет 4 WS-канала: [docs.polymarket](https://docs.polymarket.com/market-data/websocket/overview)

| Канал | Эндпоинт | Авторизация |
|---|---|---|
| **Market** | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | Нет |
| **User** | `wss://ws-subscriptions-clob.polymarket.com/ws/user` | Да (HMAC) |
| **Sports** | `wss://sports-api.polymarket.com/ws` | Нет |
| **RTDS** | `wss://ws-live-data.polymarket.com` | Опционально |

**Для BTC 15-минутного окна нужны два канала:**

**1. Market channel** — подписка сразу после открытия соединения, передаёшь `assets_ids` (это `token_id` Yes/No токенов рынка): [docs.polymarket](https://docs.polymarket.com/market-data/websocket/overview)
```json
{
  "assets_ids": ["<YES_token_id>", "<NO_token_id>"],
  "type": "market",
  "custom_feature_enabled": true
}
```
Типы событий, которые ты получаешь:
- `book` — снапшот ордербука
- `price_change` — обновление уровня цены
- `last_trade_price` — исполнение сделки
- `best_bid_ask` — лучшие bid/ask (нужен `custom_feature_enabled: true`)
- `market_resolved` — рынок закрыт (критично для 15м окна!)

**2. User channel** — для мониторинга своих ордеров и трейдов: [docs.polymarket](https://docs.polymarket.com/market-data/websocket/user-channel)
```json
{
  "auth": {"apiKey": "...", "secret": "...", "passphrase": "..."},
  "markets": ["<condition_id>"],
  "type": "user"
}
```

**Heartbeat:** каждые 10 секунд отправлять `PING`, сервер ответит `PONG`. Если не отправлять — соединение обрывается. [docs.polymarket](https://docs.polymarket.com/market-data/websocket/overview)

***

## Проблема с поиском BTC 15м рынков

Это самое коварное место. Стандартный `gamma-api.polymarket.com/events/pagination` **не возвращает** крипто 15-минутные рынки. [reddit](https://www.reddit.com/r/PredictionsMarkets/comments/1q9uwxr/how_to_fetch_15minute_crypto_markets_via/nyyi4n8/)

**Рабочий подход** — slug-паттерн:
- URL на сайте: `polymarket.com/event/btc-updown-15m-1768032000`
- Timestamp в slug — это **Unix time начала 15-минутного окна**
- Запрос: `GET https://gamma-api.polymarket.com/markets?slug=btc-updown-15m-{timestamp}`

Получив рынок, берёшь `token_id` из поля `tokens[]` — это и есть ID для подписки на WS. [github](https://github.com/Polymarket/py-clob-client/issues/244)

***

## Аутентификация

Двухуровневая система: [docs.polymarket](https://docs.polymarket.com/trading/overview)
- **L1** — EIP-712 подпись приватным ключом → получаешь L2-кредиты (один раз)
- **L2** — HMAC-SHA256 с API key/secret/passphrase → все торговые запросы

***

## Лучшие SDK на GitHub

| Репозиторий | Язык | Звёзды | Что умеет |
|---|---|---|---|
| **[Polymarket/py-clob-client](https://github.com/Polymarket/py-clob-client)** | Python | Official | Подпись ордеров, REST + WS, auth |
| **[Polymarket/agents](https://github.com/Polymarket/agents)** | Python 3.9 | Official | AI-агенты, полный цикл трейдинга  [github](https://github.com/Polymarket/agents) |
| **[warproxxx/poly-maker](https://github.com/warproxxx/poly-maker)** | Python | Community | Маркетмейкинг, WS-мониторинг ордербука, Google Sheets для конфига  [github](https://github.com/warproxxx/poly-maker) |
| **[nevuamarkets/poly-websockets](https://github.com/nevuamarkets/poly-websockets)** | Python | Community | Plug-and-play WS: `book`, `price_change`, `last_trade_price`  [github](https://github.com/nevuamarkets/poly-websockets) |
| **[Trust412/Polymarket-spike-bot-v1](https://github.com/Trust412/Polymarket-spike-bot-v1)** | Python | 260 ⭐ | HFT, spike detection, автоматический take-profit/stop-loss  [github](https://github.com/Trust412/Polymarket-spike-bot-v1) |

**Официальный TypeScript SDK** — `@polymarket/clob-client` (npm), используется в документации. [polymarket101](https://www.polymarket101.com/en/docs/trading/api-trading)

***

## Типовой флоу для BTC 15м

1. **За ~30 сек до открытия окна** — вычисляй timestamp следующего 15м рынка, делай `GET /markets?slug=btc-updown-15m-{ts}`, получай `token_id`
2. **Подключайся к WS Market** — сразу шли подписку с `assets_ids`, слушай `book` для снапшота
3. **Торгуй** — `POST https://clob.polymarket.com/order` с EIP-712 подписанным ордером
4. **Слушай User WS** — следи за `order` и `trade` событиями своих позиций
5. **При `market_resolved`** — закрывай соединение, переходи к следующему окну