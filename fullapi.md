<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Мне нужна самая подробная и самая свежая информация по поводу торговли на polymarket, особенно по поводу битка на 15 минутное окно. Сейчас мне нужна техническая часть. На какой API отправляется, какая ws слушается в какой момент времени. Какие библиотеки и SDK есть на гитхабе самые крутые?

Вот полная техническая карта работы с Polymarket, особенно для BTC 15-минутных рынков.

***

## Архитектура API

Polymarket использует гибридную систему **CLOB (Central Limit Order Book)** — матчинг ордеров происходит off-chain, а расчёты on-chain через смарт-контракт на Polygon. Есть три основных REST API и два слоя WebSocket.[^1_1]

### REST эндпоинты

| API | Базовый URL | Назначение |
| :-- | :-- | :-- |
| **CLOB API** | `https://clob.polymarket.com` | Ордербук, размещение ордеров, цены |
| **Gamma API** | `https://gamma-api.polymarket.com` | Метаданные рынков, поиск по slug |
| **Data API** | `https://data-api.polymarket.com` | Исторические данные, трейды |

Для работы с ценами по конкретному токену используются эндпоинты CLOB:[^1_2]

- `GET /price` — цена одного токена
- `GET /book` — полный ордербук
- `GET /prices-history` — исторические цены
- `GET /midpoint` и `GET /spread`

***

## WebSocket — что и когда слушать

Polymarket предоставляет 4 WS-канала:[^1_3][^1_4]


| Канал | Эндпоинт | Авторизация |
| :-- | :-- | :-- |
| **Market** | `wss://ws-subscriptions-clob.polymarket.com/ws/market` | Нет |
| **User** | `wss://ws-subscriptions-clob.polymarket.com/ws/user` | Да (HMAC) |
| **Sports** | `wss://sports-api.polymarket.com/ws` | Нет |
| **RTDS** | `wss://ws-live-data.polymarket.com` | Опционально |

**Для BTC 15-минутного окна нужны два канала:**

**1. Market channel** — подписка сразу после открытия соединения, передаёшь `assets_ids` (это `token_id` Yes/No токенов рынка):[^1_3]

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

**2. User channel** — для мониторинга своих ордеров и трейдов:[^1_5]

```json
{
  "auth": {"apiKey": "...", "secret": "...", "passphrase": "..."},
  "markets": ["<condition_id>"],
  "type": "user"
}
```

**Heartbeat:** каждые 10 секунд отправлять `PING`, сервер ответит `PONG`. Если не отправлять — соединение обрывается.[^1_3]

***

## Проблема с поиском BTC 15м рынков

Это самое коварное место. Стандартный `gamma-api.polymarket.com/events/pagination` **не возвращает** крипто 15-минутные рынки.[^1_6][^1_7]

**Рабочий подход** — slug-паттерн:

- URL на сайте: `polymarket.com/event/btc-updown-15m-1768032000`
- Timestamp в slug — это **Unix time начала 15-минутного окна**
- Запрос: `GET https://gamma-api.polymarket.com/markets?slug=btc-updown-15m-{timestamp}`

Получив рынок, берёшь `token_id` из поля `tokens[]` — это и есть ID для подписки на WS.[^1_8]

***

## Аутентификация

Двухуровневая система:[^1_1]

- **L1** — EIP-712 подпись приватным ключом → получаешь L2-кредиты (один раз)
- **L2** — HMAC-SHA256 с API key/secret/passphrase → все торговые запросы

***

## Лучшие SDK на GitHub

| Репозиторий | Язык | Звёзды | Что умеет |
| :-- | :-- | :-- | :-- |
| **[Polymarket/py-clob-client](https://github.com/Polymarket/py-clob-client)** | Python | Official | Подпись ордеров, REST + WS, auth |
| **[Polymarket/agents](https://github.com/Polymarket/agents)** | Python 3.9 | Official | AI-агенты, полный цикл трейдинга [^1_9] |
| **[warproxxx/poly-maker](https://github.com/warproxxx/poly-maker)** | Python | Community | Маркетмейкинг, WS-мониторинг ордербука, Google Sheets для конфига [^1_10] |
| **[nevuamarkets/poly-websockets](https://github.com/nevuamarkets/poly-websockets)** | Python | Community | Plug-and-play WS: `book`, `price_change`, `last_trade_price` [^1_11] |
| **[Trust412/Polymarket-spike-bot-v1](https://github.com/Trust412/Polymarket-spike-bot-v1)** | Python | 260 ⭐ | HFT, spike detection, автоматический take-profit/stop-loss [^1_12] |

**Официальный TypeScript SDK** — `@polymarket/clob-client` (npm), используется в документации.[^1_13]

***

## Типовой флоу для BTC 15м

1. **За ~30 сек до открытия окна** — вычисляй timestamp следующего 15м рынка, делай `GET /markets?slug=btc-updown-15m-{ts}`, получай `token_id`
2. **Подключайся к WS Market** — сразу шли подписку с `assets_ids`, слушай `book` для снапшота
3. **Торгуй** — `POST https://clob.polymarket.com/order` с EIP-712 подписанным ордером
4. **Слушай User WS** — следи за `order` и `trade` событиями своих позиций
5. **При `market_resolved`** — закрывай соединение, переходи к следующему окну
<span style="display:none">[^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_22][^1_23][^1_24][^1_25][^1_26][^1_27][^1_28]</span>

<div align="center">⁂</div>

[^1_1]: https://docs.polymarket.com/trading/overview

[^1_2]: https://docs.polymarket.com/market-data/overview

[^1_3]: https://docs.polymarket.com/market-data/websocket/overview

[^1_4]: https://docs.polymarket.com/quickstart/reference/endpoints

[^1_5]: https://docs.polymarket.com/market-data/websocket/user-channel

[^1_6]: https://www.reddit.com/r/PredictionsMarkets/comments/1q9uwxr/how_to_fetch_15minute_crypto_markets_via/nyyi4n8/

[^1_7]: https://www.reddit.com/r/PredictionsMarkets/comments/1q9uwxr/how_to_fetch_15minute_crypto_markets_via/

[^1_8]: https://github.com/Polymarket/py-clob-client/issues/244

[^1_9]: https://github.com/Polymarket/agents

[^1_10]: https://github.com/warproxxx/poly-maker

[^1_11]: https://github.com/nevuamarkets/poly-websockets

[^1_12]: https://github.com/Trust412/Polymarket-spike-bot-v1

[^1_13]: https://www.polymarket101.com/en/docs/trading/api-trading

[^1_14]: https://docs.polymarket.com

[^1_15]: https://github.com/Mist-kail/Polymarket-copy-trading-bot-2025-1/blob/main/README.md

[^1_16]: https://docs.polymarket.com/market-makers/trading

[^1_17]: https://docs.polymarket.com/developers/CLOB/introduction

[^1_18]: https://www.youtube.com/watch?v=rDkVmkrzpbI

[^1_19]: https://github.com/Polymarket/py-clob-client/blob/main/README.md

[^1_20]: https://github.com/dexorynlabs/polymarket-trading-bot-python

[^1_21]: https://docs.polymarket.com/api-reference/introduction

[^1_22]: https://www.scribd.com/document/923788925/WSS-Overview-Polymarket-Documentation

[^1_23]: https://github.com/orgs/community/discussions/152113

[^1_24]: https://polymarket-292d1b1b.mintlify.app/quickstart/websocket/WSS-Quickstart

[^1_25]: https://polymarketcn.com/api-tutorial/gamma-endpoints/markets

[^1_26]: https://docs.polymarket.com/developers/CLOB/websocket/wss-overview

[^1_27]: https://docs.polymarket.com/market-data/fetching-markets

[^1_28]: https://polymarket-292d1b1b.mintlify.app/developers/CLOB/websocket/wss-overview


---

# wss://ws-live-data.polymarket.com а это для чего вс?

`wss://ws-live-data.polymarket.com` — это **RTDS (Real-Time Data Socket)**, совершенно отдельный слой от CLOB WebSocket. Он предназначен не для торговли, а для **широковещательных рыночных данных и фида активности**.[^2_6]

***

## Что умеет RTDS

Полная таблица топиков:[^2_7]


| Topic | Type | Auth | Что даёт |
| :-- | :-- | :-- | :-- |
| `crypto_prices` | `update` | Нет | Реальная цена BTC/ETH/etc от оракула |
| `activity` | `trades` | Нет | Все сделки по рынку (фильтр по `market_slug`) |
| `activity` | `orders_matched` | Нет | Матчинг ордеров |
| `clob_market` | `price_change` | Нет | Изменение цен (фильтр по `token_id[]`) |
| `clob_market` | `agg_orderbook` | Нет | Агрегированный ордербук |
| `clob_market` | `last_trade_price` | Нет | Последняя цена сделки |
| `clob_market` | `market_created` | Нет | Новый рынок открыт |
| `clob_market` | `market_resolved` | Нет | Рынок закрыт ✅ |
| `clob_user` | `order` / `trade` | **ClobAuth** | Свои ордера и трейды |
| `rfq` | `*` | Нет | Request-for-quote события |
| `comments` | `*` | Нет | Комментарии/реакции |


***

## Ключевое отличие от CLOB WS

|  | CLOB WS (`ws-subscriptions-clob`) | RTDS (`ws-live-data`) |
| :-- | :-- | :-- |
| **Для чего** | Торговля, ордербук | Широкий фид, аналитика |
| **Протокол подписки** | JSON с `assets_ids` | JSON с `subscriptions[]` |
| **Ping интервал** | 10 сек | **5 сек** |
| **Всё в одном** | Нет | Да — CLOB + crypto + activity + RFQ |

По сути, RTDS — это **единый WebSocket**, который объединяет данные из нескольких источников. Если не хочешь держать два отдельных соединения, можно использовать только RTDS — он покрывает и `clob_market`, и `clob_user`, и `crypto_prices`.[^2_5]

***

## Применение для BTC 15м

Самое важное — `crypto_prices` топик. Это **цена реального BTC с оракула**, по которой Polymarket резолвит 15-минутный рынок. Ты можешь сравнивать цену оракула с ценой Yes/No токена в реальном времени и искать арбитраж.[^2_6]

```typescript
// npm install @polymarket/real-time-data-client
import { RealTimeDataClient } from "@polymarket/real-time-data-client";

const onConnect = (client) => {
  // Цена BTC от оракула
  client.subscribe({
    subscriptions: [{ topic: "crypto_prices", type: "update" }]
  });

  // Все сделки по конкретному 15м рынку
  client.subscribe({
    subscriptions: [{
      topic: "activity",
      type: "trades",
      filters: `{"market_slug":"btc-updown-15m-1768032000"}`
    }]
  });

  // Резолв рынка — критично!
  client.subscribe({
    subscriptions: [{ topic: "clob_market", type: "market_resolved" }]
  });
};

new RealTimeDataClient({ onMessage, onConnect }).connect();
```

Есть также Rust-крейт `polymarket-rtds` на lib.rs если нужна производительность.[^2_4]
<span style="display:none">[^2_1][^2_10][^2_2][^2_3][^2_8][^2_9]</span>

<div align="center">⁂</div>

[^2_1]: https://docs.polymarket.com/market-data/websocket/rtds

[^2_2]: https://docs.polymarket.com/market-data/websocket/overview

[^2_3]: https://www.reddit.com/r/learnpython/comments/1qo6z1o/python_websocket_client_for_streaming_live_market/

[^2_4]: https://lib.rs/crates/polymarket-rtds

[^2_5]: https://www.polytrackhq.app/blog/polymarket-websocket-tutorial

[^2_6]: https://docs.polymarket.com/developers/RTDS/RTDS-overview

[^2_7]: https://github.com/Polymarket/real-time-data-client/blob/main/README.md

[^2_8]: https://www.youtube.com/watch?v=rDkVmkrzpbI

[^2_9]: https://github.com/Polymarket/real-time-data-client

[^2_10]: https://www.reddit.com/r/PolymarketHQ/comments/1hdr4ft/websocket_for_polymarket/

