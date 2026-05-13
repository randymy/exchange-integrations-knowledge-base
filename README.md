# Exchange Integrations — Field Notes

Practical notes from integrating multiple perpetual-futures and spot exchanges. Each per-exchange file (see `exchanges/`) is a hard-won reference for engineers actually writing the code. This README captures problems that **recur across venues** and the conventions used here to talk about them.

These notes are written to be honest about what's deceptive, not to evangelize any venue. If the docs and the live wire disagree, the live wire wins — we say so each time we hit it.

---

## Contents

| Venue                      | Type                                          | Doc                                  |
| -------------------------- | --------------------------------------------- | ------------------------------------ |
| Nado (Vertex fork on Ink)  | DEX, perps, on-chain settlement               | [exchanges/nado.md](exchanges/nado.md)             |
| Hyperliquid + HIP-3 venues | DEX, perps, custom L1 + permissionless perp DEXes | [exchanges/hyperliquid.md](exchanges/hyperliquid.md) |
| o2 (Fuel L2)               | DEX, spot only, Fuel-native signing            | [exchanges/o2.md](exchanges/o2.md)             |
| Figure Markets             | Centralized, US-regulated, OAuth2              | [exchanges/figuremarkets.md](exchanges/figuremarkets.md) |

---

## 1. Wire format scaling — every venue is different

Display values (`0.05 ETH`, `$2148.50`) are not what the API wants. Each venue scales differently, and getting it wrong produces either silent-success-wrong-size or a 4xx with a message that doesn't name the offending field.

| Venue           | Scale                                                         | Type   |
| --------------- | ------------------------------------------------------------- | ------ |
| Nado / Vertex fork | X18 — multiply by 10^18                                       | integer |
| o2 / Fuel       | 9 decimals across all assets — multiply by 10^9               | integer |
| Hyperliquid     | szDecimals per asset (typically 4 for crypto); price has separate sig-fig + decimal rules | float, then formatted to N decimals |
| Figure Markets  | Per-market `denomExponent` (qty) and `quoteExponent` (price), passed as **strings** | scaled integer **string** |

Recurring rule: source the scaling from a metadata endpoint at startup. **Never hardcode it per asset.** When the venue updates a perp's tick from $0.10 to $0.05, hardcoded clients silently start rounding orders to the wrong increment.

## 2. Signing format — six pairs of incompatible conventions

| Venue           | Signing                                                                 |
| --------------- | ----------------------------------------------------------------------- |
| Nado / Vertex fork | EIP-712 typed data, secp256k1; per-action types (LinkSigner, Order, ...) |
| Hyperliquid     | EIP-712 with venue-specific domain; API wallet = delegated signer, separate from main account |
| o2 / Fuel       | Fuel personal message format (sha256 with `"\x19Fuel Signed Message:\n"` prefix) for owner; raw sha256 for session keys — **not EIP-712** |
| Figure Markets  | OAuth2 bearer token (no message signing); tokens opaque, ~24h TTL       |

The cost of wrong signing is rarely a clear error. Symptoms:
- **Wrong chain ID in EIP-712 domain**: signature recovers to a random-looking address; the venue says "signature does not match sender" and prints the recovered address. *It is not your key being wrong; it is the domain being wrong.*
- **Wrong domain name string**: same recovery failure. Vertex-fork venues each use their own brand name in the EIP-712 domain (`"Nado"`, etc.), not `"Vertex"`.
- **Wrong host for OAuth token**: 401 immediately on first call. (Figure Markets explicitly: tokens come from `www.*`, never from `trade.*`.)

When in doubt: log every signing input (domain name, chain ID, message bytes) during integration. The recovered address is the diagnostic — if it's not your address, your domain or your message hash is wrong, not your private key.

## 3. Nonce semantics differ by venue

| Venue          | Nonce model                                                                                             |
| -------------- | ------------------------------------------------------------------------------------------------------- |
| Nado / Vertex  | Time-based nanosecond timestamp for orders; strictly increasing for `link_signer` etc.                  |
| Hyperliquid    | Internal — managed by the SDK; you generally don't see it                                               |
| o2 / Fuel      | Strictly sequential per account, fetched from the contract before every action — **never cached safely** |
| Figure Markets | No client-side nonce; the bearer token is the entire auth                                               |

Two failure modes recur:
- **Caching a sequential nonce** (o2-style) causes silent replay rejection. Always re-fetch immediately before signing.
- **Nanosecond timestamps in JavaScript**: `1.85 × 10^18` exceeds `Number.MAX_SAFE_INTEGER` (`9.0 × 10^15`). If your nonce flows through any JS layer, it must be a string in JSON and parsed back to BigInt. Python and Rust have no issue.

## 4. Account / identity models — none of them are 1:1

| Venue          | Account model                                                                                          |
| -------------- | ------------------------------------------------------------------------------------------------------ |
| Nado / Vertex  | Wallet → subaccounts. Subaccount = `bytes20(owner) + bytes12(ascii_name_padded)`. `link_signer` delegates per-subaccount. |
| Hyperliquid    | Main account (holds funds, cannot be the order signer for safety) + API wallet (signs orders, cannot withdraw) + subaccounts (separate addresses, must be funded by main, created via SDK) |
| o2 / Fuel      | Owner key (creates account, deposits, sessions; can withdraw) + session keys (time-bound, scoped, cannot withdraw)                |
| Figure Markets | One user = one account = one client_id/client_secret. No sub-account or delegation. |

Two things to plan for early:
- **Storage shape.** A single `signing_key` column does not scale past one venue. Each venue has its own per-user credentials and they rotate independently. Plan for `{venue}_signing_key` columns, or a polymorphic `exchange_credentials(user_id, exchange, payload_encrypted)` table from the start.
- **Reconciling sub-accounts.** Sub-account naming, derivation, and visibility differ. Nado: user picks a name, the bytes32 is derived from it. Hyperliquid: SDK returns a computed address; you don't pick it. o2: there is no sub-account; account isolation requires separate owner keys. FM: no sub-account model exists.

## 5. Order types and TIF — no shared vocabulary

| Venue          | TIF names                                                                      | Notes                                       |
| -------------- | ------------------------------------------------------------------------------ | ------------------------------------------- |
| Nado / Vertex  | `GTC`, `IOC`, `FOK`, post-only flag                                            | Off-chain matching, on-chain settlement     |
| Hyperliquid    | `"Gtc"`, `"Ioc"`, `"Alo"` (add-liquidity-only = post-only)                     | Bare strings, case-sensitive                |
| o2 / Fuel      | `OrderType` enum: `SPOT` (= GTC), `MARKET`, `FILL_OR_KILL`, `POST_ONLY`, `BOUNDED_MARKET` | Python SDK exposes enum                      |
| Figure Markets | Full protobuf names: `TIME_IN_FORCE_GTC`, `TIME_IN_FORCE_DAY`, `TIME_IN_FORCE_IOC`, `TIME_IN_FORCE_FOK`; **no post-only** | Bare strings (`"GTC"`, `"IOC"`) are rejected |

Map them through a single internal `OrderTif` enum in your adapter layer; never let venue-specific TIF strings escape into business logic.

## 6. Symbol naming has no convention

Each venue picks a different format and a different normalization.

| Venue          | Example symbols                                              | Convention                                                |
| -------------- | ------------------------------------------------------------ | --------------------------------------------------------- |
| Nado / Vertex  | `BTC-PERP`, `ETH-PERP`                                       | Looked up by **product_id** internally; **odd = spot, even = perp** in the fork variant — easy to trade the wrong product |
| Hyperliquid    | `BTC`, `ETH`; HIP-3 venues use `xyz:TSLA`, `xyz:GOLD`         | Bare coin name; HIP-3 prefixed with venue dex             |
| o2 / Fuel      | `ETH/USDC`, `wBTC/USDC` (mainnet); `fETH/fUSDC` (testnet)     | Slash-separated; testnet prefixes with `f`                |
| Figure Markets | `BTC-USD-2S`, `BTC-USDC-2S`; `ETH-USD`, `HASH-USD`            | BTC pairs carry a `-2S` venue suffix; other assets don't  |

Recurring lesson: always list the available symbols at startup via the venue's own discovery endpoint, never hardcode. The same asset has different names on different venues and sometimes a different name on the same venue's testnet.

## 7. Fill detection — three patterns, two are reliable

| Venue          | Mechanism                                                                                   | Typical latency |
| -------------- | ------------------------------------------------------------------------------------------- | --------------- |
| Nado / Vertex  | Poll open-orders, diff against the prior tick. Orders that disappeared were filled or cancelled — no native fill stream we found reliable. | ~250 ms cycle    |
| Hyperliquid    | WebSocket `userEvents` channel — events for fills, liquidations, cancels                    | <100 ms          |
| o2 / Fuel      | WebSocket `stream_orders` — `order.close = true` indicates fill or cancel                    | <100 ms          |
| Figure Markets | NDJSON subscription via `create_order_subscription`                                          | <100 ms          |

For market-making and spread strategies: fill detection latency must be **lower than your quote refresh interval**, or you'll miss fills and either over-quote (placing duplicates) or under-hedge (leaving naked positions). Plan around the **slowest** venue in any cross-venue strategy.

## 8. Cross-venue strategies — pitfalls beyond any single venue

Patterns that come up repeatedly when wiring two or more venues together:

### 8.1 Spread-quoting math — same-side counterpart, not mid

The naive "quote around joint mid" formula (`bid_A = leg_b.mid × (1 − offset)`) silently conflates the operator's intended edge with the cross-venue basis. The correct formulation:

- Bidding leg A (we buy A → hedge sells leg B at the bid):
  `bid_A = leg_b.bid × (1 − offset)`
- Asking leg A (we sell A → hedge buys leg B at the ask):
  `ask_A = leg_b.ask × (1 + offset)`

This makes the operator's `offset_bps` always equal to the realized edge per round-trip fill. The mid-based version drifts with basis (sometimes favorable, sometimes adverse) and is hard to reason about as a target return.

### 8.2 Hedge failure is the real risk

Cross-venue spreading goes wrong in one specific way: the passive leg fills, the hedge leg fails (balance, health, connectivity, rate limit). You now hold a directional position you weren't trying to hold. Build for this case explicitly:

1. **Pre-check the hedge venue** before quoting. Balance, health, connectivity. Don't trust that "it was fine last tick."
2. **State machine, not implicit flow.** `quoting → filled → hedging → hedged → quoting` is the happy path; `→ hedge_failed → manual_intervention` is the sad path. Never automatically re-enter quoting from `hedge_failed` without operator action.
3. **Retry once, with more aggressive pricing.** If the hedge fails because the price moved, +N pay-up ticks usually clears it. If it still fails, stop — don't keep retrying; that's how stale state turns into a runaway loss.
4. **Surface the unhedged state prominently.** Log + alert + UI badge. A market-neutral strategy with an unhedged leg is no longer market-neutral.

### 8.3 Cross-venue label widths

Any column or identifier that combines venue + market for a cross-venue position needs to be sized for the longest plausible combination, not the longest single source. Example: `BTC-USD-2S/BTC-USDC-2S` is 22 characters; if your column is `VARCHAR(20)` (sized for `XYZ100-PERP`) the first cross-venue bot fails with an opaque truncation error.

### 8.4 Fill-detection asymmetry shapes which leg should be active

If leg A's fill is detected in ~100 ms and leg B's in ~250 ms, the venue you'd prefer as the **active** (quoting) leg is the one whose fills you detect first. Otherwise your hedge starts late by the slow venue's polling interval.

### 8.5 Quote-size minimum is the larger of the two legs

If leg A has a $10-notional minimum (e.g. `~0.00013 BTC` at $80k) and leg B has a $1-notional minimum, **the spread's minimum is the larger of the two** — anything smaller will be rejected at leg A and you'll have a one-sided spread. Don't hardcode a UI minimum; derive it from the per-leg metadata.

## 9. Observability that pays off

Patterns that took the longest to debug each had the same property: the symptom looked unrelated to the cause. A short, generic toolkit catches them earlier than any specific check would:

- **Log every external call's request and response.** Sanitize secrets, log everything else. The number of bugs caught by reading a log of the exact wire payload is much higher than the number caught by reading code.
- **Log every signing input** (domain, chain ID, message hash, recovered address) during integration. Drop to summary level afterwards.
- **Audit log table** for every sensitive action with structured fields: who, what, target, result, error_code, IP, user-agent. Don't put secrets in the details JSON.
- **Per-asset metadata cache** with periodic refresh — drives smart minimums, tick rounding, and notional checks. When a venue silently changes precision, the cache shows it.
- **Cross-asset outlier auditor on metadata.** When one venue exposes per-asset precision and most assets sit in a narrow band (e.g. tick is 0.01–0.05% of mid), flag any asset that sits >100× outside the band. Catches the rotting-config case where the venue's own metadata drifted from reality. Cheaper than reviewing every asset by hand.

## 10. Things that do NOT generalize

A short list of "looks similar, actually different" traps:

- **EIP-712 domain across Vertex forks.** Each fork rebrands the `domain.name`. Reusing one fork's domain string against another fork's gateway recovers a junk address.
- **WebSocket reliability.** Two venues that both advertise WS market data may have very different uptime and coverage. Always have a polling fallback before going live.
- **Order ID format.** Some venues return camelCase (`orderId`), some return snake_case (`order_id`), and at least one venue's docs disagree with its live wire. Read both keys defensively on responses; match the live wire on requests.
- **"Subaccount" semantics.** Different venues mean different things by "subaccount" — sometimes a separate on-chain address, sometimes a label inside one wallet's namespace, sometimes a delegated key. Never assume the wire format carries over.

## 11. Reading order

For an engineer just starting on a venue:

1. **Read the venue's per-file doc here** before reading the venue's official docs. We list the disagreements between docs and live wire — that's where your time will go.
2. **Then read the venue's official docs** end-to-end. The per-file doc here assumes you've seen the spec.
3. **Wire up the metadata endpoints first.** List markets, list symbols, list accounts. Get the discovery layer right before placing a single order.
4. **Place a tiny test order, far out of market.** Verify it appears in the open-orders list. Cancel it. Verify the cancel succeeded. If either step is flaky, fix that before doing anything else.
5. **Build the adapter layer.** Behind a small interface (`place_order`, `cancel_order`, `get_position`, etc.) so the rest of the code never sees venue-specific details.

The per-venue files capture the specific gotchas. Read them carefully — most of the entries cost someone half a day to a full day to figure out the first time.
