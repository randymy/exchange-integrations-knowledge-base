# Figure Markets

Figure Markets is a centralized, US-regulated spot exchange. Auth is OAuth2 client-credentials (no wallet signing); orders are placed via a REST Trader API, and live market data streams over an NDJSON HTTP subscription.

This doc captures what's not obvious from the official docs alone. If something here disagrees with the venue's own docs, the docs win — but most entries below are places where the **live wire** and the docs already disagree, and we mark those explicitly.

## Official docs (canonical)

- Overview: `figuremarkets.dev/api-docs/trader-api/`
- Authentication: `figuremarkets.dev/api-docs/trader-api/authentication/`
- Markets: `figuremarkets.dev/api-docs/trader-api/trading/markets/`
- Orders: `figuremarkets.dev/api-docs/trader-api/trading/orders/`
- Account: `figuremarkets.dev/api-docs/trader-api/trading/account/`
- Public API: `figuremarkets.dev/api-docs/public-api/markets/`
- Full OpenAPI: `figuremarkets.dev/api-docs/apis/trader-direct/`

Always cite these first.

## Three host families — don't mix them up

| Purpose                  | Production                                | UAT                                  |
| ------------------------ | ----------------------------------------- | ------------------------------------ |
| OAuth (token)            | `www.figuremarkets.com`                   | `www.figuremarkets.dev`              |
| Trader REST              | `trade.figuremarkets.com`                 | `trade.figuremarkets.dev`            |
| Public REST (no auth)    | `www.figuremarkets.com`                   | `www.figuremarkets.dev`              |

**Hard rule:** OAuth tokens are obtained from `www.*`, **never** from `trade.*`. The auth doc is explicit. A `curl` that hits `trade.*` for auth may appear to work today (or have worked in the past), but is off-spec — don't propagate it into a permanent integration.

**Public API path note.** The docs reference `/public` but the production path that the venue's own docs site uses is `/service-hft-exchange`. Both currently resolve to the same backend; prefer `/service-hft-exchange` until the docs catch up:

```text
https://figuremarkets.com/service-hft-exchange/api/v1/markets
```

## Authentication

OAuth 2.0 client credentials. Obtain credentials from the venue's trader-API console.

```text
POST https://www.figuremarkets.com/auth/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id=...
&client_secret=...
&scope=TRADER
```

Response:

```text
{"access_token": "<opaque>", "token_type": "Bearer", "expires_in": 86399, ...}
```

The token is **opaque** — don't try to parse as a JWT. It's valid for ~24 hours. Refresh proactively (60 seconds before expiry) and on any 401. Send as `Authorization: Bearer <token>` on every Trader API request.

## Two exchanges: US and Global (Cayman)

Figure Markets operates two distinct trading venues:

- **US exchange** — US-registered. `location=US`.
- **Global exchange (Cayman)** — Cayman-based. `location=CAYMAN`.

**Instruments are location-specific.** A symbol that's tradable on the US venue is not necessarily tradable on the Global venue, and vice versa. The instrument list returned by the discovery endpoints depends on the `location` you pass — calling `list_symbols` without specifying one gives you whatever the venue defaults to for your credentials, which is usually not the full picture.

**Symbol prefix convention:**
- `KY-...` prefix → **Global (Cayman)** instrument (e.g. `KY-BTC-USD`, `KY-ETH-USD`).
- No `KY-` prefix → **US** instrument (e.g. `BTC-USD-2S`, `ETH-USD`).

**Decide upfront which venue you're trading on.** Your credentials, instrument list, and order-routing all key off the location. Trying to place an order in a symbol from a venue your account isn't onboarded to returns `"Order for other location"`. Submitting a `list_*` call without a location returns a partial view and will silently miss instruments you actually have access to.

For every symbol-discovery call below, **always pass the location explicitly.** Treat it as a required parameter even when the API technically allows omitting it.

## Symbol naming — `-2S` suffix on BTC, `KY-` prefix on Global

Figure Markets uses venue-specific symbol naming. Two independent conventions to be aware of:

- **`KY-` prefix** → Global (Cayman) instrument (see Two-exchanges section above). US instruments do not carry this prefix.
- **`-2S` suffix on BTC pairs** (US side observed): `BTC-USD-2S`, `BTC-USDC-2S`. Other assets do not carry the suffix (`ETH-USD`, `ETH-USDC`, `SOL-USD`, ...). The Global-side BTC equivalent uses the `KY-` prefix instead (e.g. `KY-BTC-USD`).

**Always use the exact symbol the venue returns from the markets endpoint** — don't assume the "obvious" name works. When debugging "no data" or "unknown symbol" errors, the first check is whether the symbol string exactly matches what `list_symbols` (with the right location) or `/api/v1/markets?location=...` reports. Use `symbol`, not `displayName`.

## Symbol discovery — multiple endpoints (always pass `location`)

| Endpoint                                                                            | Auth | Use when                                            |
| ----------------------------------------------------------------------------------- | ---- | --------------------------------------------------- |
| `POST /api/v1/list_symbols` (body includes `location`)                              | Yes  | Just need symbol strings                            |
| `POST /api/v1/list_instruments` (body includes `location`, `pageSize`, `pageToken`) | Yes  | Need precision/tick/limits for many markets         |
| `POST /api/v1/get_instrument_metadata` (body: `{symbol}`)                           | Yes  | Need metadata for one market                        |
| `GET /v1/markets?location=US` (or `CAYMAN`)                                         | No   | Public list filtered by location                    |
| `GET /service-hft-exchange/api/v1/markets?location=US&candle_type=TRADE`            | No   | Same with candle filter; used in production         |

The `location` filter is not optional in practice. Without it, the response is whichever venue the API defaults to for your credentials — which silently omits instruments you might actually be able to trade on the other venue. Per the Orders doc, only markets matching your market location are valid for order entry by default; filter client-side as well so a missed location parameter doesn't slip an order to the wrong venue.

Two-venue accounts (some operators have both US and Cayman onboarding) need to call the discovery endpoints **once per location** and treat the unions or per-venue lists explicitly in their adapter layer.

## Wire format — scaled integer strings, not display values

The Trader API does **not** accept display values. Prices, quantities, and notionals are **strings of scaled integers**:

- `price = display_price × tickSize` (or `priceScale`)
- `orderQty = display_qty × quantityScale` (or `fractionalQuantityScale`)
- `notional = display_qty × display_price × both scales`

Fetch scales once at startup via `get_instrument_metadata`. Field names vary across docs (`priceScale` / `pricePerTick` / `tickSize`; `quantityScale` / `qtyScale` / `fractionalQuantityScale`); try them in order.

The Public API `/api/v1/markets` response is the simplest source — it returns `denomExponent` and `quoteExponent` directly:

```text
qty_scale   = 10 ** denomExponent
price_scale = 10 ** quoteExponent
```

Send everything as **strings**:

```json
{"price": "3500000000", "orderQty": "1000000"}
```

Not as JSON numbers. Sending `"price": 3500000000` (int) or `"price": 70000` (display) is rejected with an error that doesn't explain *why*.

## Account references — resource paths, not flat IDs

Accounts are returned as resource paths:

```text
firms/{firmId}/accounts/{accountId}
```

`list_accounts` returns these full strings. Pass the full string anywhere a request field is named `account` or `accounts`. Bare IDs are rejected with a generic 400.

## Order enums — full protobuf names

Use the full protobuf JSON names. Examples:

- `ORDER_TYPE_LIMIT`, `ORDER_TYPE_MARKET`, `ORDER_TYPE_STOP_LIMIT`
- `SIDE_BUY`, `SIDE_SELL`
- `TIME_IN_FORCE_DAY`, `TIME_IN_FORCE_IOC`, `TIME_IN_FORCE_GTC`, `TIME_IN_FORCE_FOK`

Bare strings like `"LIMIT"` or `"BUY"` are rejected. **There is no post-only TIF** — if a higher-level abstraction maps "post-only" to FM, collapse it to `TIME_IN_FORCE_GTC` (or refuse the order; either is a defensible choice).

## `insert_order` response uses `order_id`, not `orderId`

The live Trader API returns insert results as:

```json
{"order_id": "9ZW0P4X9003Q", ...}
```

The OpenAPI docs at `figuremarkets.dev/api-docs/...` and some older example payloads show `orderId` (camelCase). They disagree with the live wire as of this writing. Read both keys defensively on responses; **send `order_id` (snake_case)** on `cancel_order` requests — that matches what the live API expects on the input side too.

Symptom of getting this wrong: every successful place reports
`"Unexpected response: {'order_id': '...'}"` even though the order is live on the book. Cancels silently no-op because FM ignores the unknown `orderId` field.

## Streaming market data — NDJSON, not WebSocket

The Trader API's streaming channel is an **HTTP NDJSON subscription**, not a WebSocket. There is a WS service at `wss://figuremarkets.com/service-hft-exchange-websocket/` that partially exists, but it is not in the documented Trader API; the `MARKET` channel returns ticker-only data (no order quantities), and other channel names we tried (`ORDERBOOK`, `BBO`, `BOOK`) either silently drop the subscription or return nothing useful. Use the NDJSON stream instead.

```text
POST https://trade.figuremarkets.com/api/v1/create_market_data_subscription
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/x-ndjson

{
  "symbols": ["BTC-USD-2S"],
  "depth": 1
}
```

Response is `application/x-ndjson` — one JSON object per line. In Python with `requests`:

```python
with requests.post(url, json=body, headers=headers,
                   stream=True, timeout=None) as r:
    r.raise_for_status()
    for raw in r.iter_lines():
        if not raw:
            continue
        msg = json.loads(raw)
        # ... handle update / heartbeat per spec
```

Message types include `update` and (optional) `heartbeat`. Full schema in the OpenAPI spec.

For own-order events (fills, cancels), use `POST /api/v1/create_order_subscription` with the same NDJSON pattern.

## No sub-account model

Figure Markets is "one client = one account = one client_id/client_secret." There is no sub-account mechanism comparable to Vertex-family `link_signer` or Hyperliquid API wallets / subaccounts.

Practical consequences:

- **Delegating trading authority is not possible at the venue level.** Any "shared account" arrangement is operational, not protocol-enforced.
- **Per-bot or per-user credentials are the right unit of isolation.** Multiple users sharing one client_id share full account access, including withdrawal authority.
- For a platform that hosts traders, each trader brings their own Figure Markets credentials — there's no architectural alternative.

## Pagination — two conventions

| Endpoint                               | Pagination shape                                    |
| -------------------------------------- | --------------------------------------------------- |
| `list_instruments`, `search_orders`, `search_trades`, `search_executions` | `pageSize` (int) + `pageToken` (string); response has `nextPageToken`, absent on last page. Send empty `pageToken` first time. |
| Public API `/api/v1/markets`           | `page` (1-indexed) + `size` (capped at 50). 400 if `page=0`. |

## Public market data exposes top-of-book only

The public unauthenticated `/api/v1/markets` endpoint returns `bestBid` and `bestAsk` per symbol but not full L2 depth. **Depth requires OAuth via the NDJSON subscription.**

Practical consequence: if you're building a public market-data fan-out service or a price ladder, you can show top-of-book without authentication, but "10 levels of size" requires the per-user OAuth session. For a multiplexed market-data service, FM doesn't fit the same shape as venues that expose anonymous L2 — fan-out either needs a shared service account or accepts top-of-book only on the public path.

## Things to NOT do

- **Don't use `wss://figuremarkets.com/service-hft-exchange-websocket/`.** It exists but is not in the documented Trader API and the channel coverage is incomplete. Use the NDJSON stream.
- **Don't authenticate against `trade.figuremarkets.*`.** Always `www.*`.
- **Don't send display-value prices/qtys.** Always wire-scaled integer strings.
- **Don't assume symbol names.** Always look up via `list_symbols` (or one of the other discovery endpoints), and **always pass the `location` parameter** so the result actually covers the venue you're trading on. `KY-`-prefixed symbols belong to the Global (Cayman) exchange; non-prefixed symbols belong to US.
- **Don't use bare account IDs.** Always the full `firms/.../accounts/...` resource path.
- **Don't hardcode response key casing.** Read both `order_id` and `orderId` defensively; send `order_id`.

## Common 4xx debugging

| Symptom                                | Likely cause                                                  |
| -------------------------------------- | ------------------------------------------------------------- |
| 401 immediately on first call          | Wrong auth host (`trade.*` instead of `www.*`)                |
| 401 after 24h of working               | Token expired; refresh proactively                            |
| 400 on `insert_order`                  | Display values instead of wire-scaled integer strings         |
| 400 on `insert_order` (qty/price)      | Number instead of string                                      |
| 400 on `insert_order` (account)        | Bare ID instead of `firms/.../accounts/...` resource path     |
| 400 with "unknown symbol"              | Symbol typo, missing `-2S` suffix on BTC, or missing `KY-` prefix when targeting Global |
| Empty subscription stream              | Wrong endpoint, or symbol not in your location                |
| "Order for other location"             | US symbol submitted to a Global-onboarded account (or vice versa); pass the correct `location` |
| `list_symbols` returns fewer markets than expected | `location` not passed, or passed as a value your credentials aren't onboarded to |
| 400 with `getAllMarkets.page` message  | Public markets endpoint pages are 1-indexed                   |
| `"Unexpected response: {'order_id'...}"` from your own wrapper | Wrapper reads only `orderId`; live API returns `order_id` |

## Tick-precision outliers — read the venue's own metadata, but verify

Some FM symbols publish a `priceIncrement` that's a meaningful fraction of mid. As an example observed at time of writing, HASH-USD and HASH-USDC publish `priceIncrement = 0.001` on an asset trading around $0.01 — a single tick is ~9% of mid. Every other FM market sits at <0.03% tick-of-mid.

The bot path honors the published precision and submits orders at the published increment, which works but produces unusually coarse spreads. Worth running a relative-tick auditor across the venue's universe (`tick / mid` per symbol) at startup to flag outliers. The fix, when found, is upstream at the venue — you can't tighten a tick the venue enforces.
