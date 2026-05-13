# Hyperliquid (+ HIP-3 permissionless perp DEXes)

Hyperliquid runs its own L1 with native crypto perps and supports permissionless perp DEXes built on the protocol (HIP-3). Both surfaces are reachable from the same SDK and signing key, but the routing has venue-specific rules that aren't obvious until you hit them.

This doc covers the non-obvious parts. Read alongside, not instead of, Hyperliquid's official docs and SDK source.

## Two surfaces, one SDK — `perp_dexs` matters

Hyperliquid's native perps live under the default dex ID `""` (empty string). HIP-3 venues (third-party perp DEXes deployed on Hyperliquid, e.g. for equity perps) live under their own dex ID — typically a short string the venue chose.

Both `Info` and `Exchange` SDK clients must be constructed with **every dex ID you intend to touch** preloaded:

```python
Info(base_url, skip_ws=True, perp_dexs=["", "xyz"])
Exchange(api_wallet, base_url, vault_address=vault, perp_dexs=["", "xyz"])
```

Failure mode: passing only `""` makes lookups for HIP-3 symbols (e.g. `xyz:TSLA`) succeed at the metadata level — `info.meta()` will return entries — but order placement fails silently or with a generic "asset not found" error. The SDK doesn't tell you the dex routing is wrong; it just routes everything to dex `""` and the order references an asset that doesn't exist there.

If you're integrating multiple HIP-3 venues, preload all their dex IDs in one client. There's no per-call dex override; it's bound at construction.

## `vault_address` — all-or-nothing order routing

Setting `vault_address` on the `Exchange` client routes **every** order through that subaccount. There is no per-order override.

```python
# Wrong: orders hit the main account, not the intended subaccount
Exchange(api_wallet, base_url, vault_address=None, ...)

# Right: orders hit the subaccount
Exchange(api_wallet, base_url, vault_address=subaccount_addr, ...)
```

Failure mode: a delegated trading bot constructed without `vault_address` will silently trade the **main account**, not the subaccount. No error is raised. The first sign is balance changes in the wrong place.

If your code path needs to manage multiple subaccounts, you'll need **multiple `Exchange` clients**, one per `vault_address`. They can share the same API wallet.

## API wallet ≠ main account — two distinct identities

Hyperliquid's "API wallet" is a delegated signing key generated at the venue's web app. It **signs orders** but **cannot withdraw**. The main account is a separate keypair that holds funds and owns subaccounts.

Practical consequence: any per-user storage needs both:

```text
hl_signing_key         — the API wallet's private key (signs orders)
hl_account_address     — the main account's address (holds funds, parent of subaccounts)
```

Conflating them into a single column breaks routing and breaks withdrawal logic. If you've ever stored just "the user's Hyperliquid key" in a single field, you've stored only one of the two identities and the other is recoverable only by asking the user again.

## Subaccount creation requires trading history

You cannot create a Hyperliquid subaccount on a master account that has **never traded**. The API returns "vault not found" at creation time, even with a valid signed request.

This blocks fresh users from onboarding a delegation flow before placing one trade themselves. Mitigation in product flow: gate "create subaccount" actions on a server-side check that the master account has nonzero 24h volume, with copy that explains why.

## Subaccount address is deterministic, not user-chosen

Unlike Vertex-family venues (where the user picks a subaccount name and the bytes32 is derived from it), Hyperliquid returns a **computed subaccount address** from the SDK. Don't generate it or namespace it yourself. Store both:

- `subaccount_address` — canonical, on-chain, returned by SDK
- `subaccount_name` — display label only, your application's metadata

The address is what every API call needs; the name is purely cosmetic.

## `szDecimals` is per-asset, not global

Order size precision varies per asset. Crypto perps default to `szDecimals = 4`; HIP-3 perps may differ. Hardcoding precision causes "invalid order size" rejections that don't name the offending field.

Always fetch `szDecimals` from `info.meta()` per coin and round size accordingly. The SDK exposes this; use it on every order, not just at startup.

## Price precision — two rules, both apply

Hyperliquid prices follow two simultaneous constraints:

1. **At most 5 significant figures.**
2. **At most `6 − szDecimals` decimals.**

The effective tick size depends on the asset's current price:

| Asset       | szDecimals | Mark price | Sig-fig tick | Decimal tick (6 − sz) | Effective tick |
| ----------- | ---------- | ---------- | ------------ | --------------------- | -------------- |
| BTC         | 4          | $80,000     | 1.0 (5-sf rule wins) | 0.01 | **1.0** |
| ETH         | 4          | $3,000      | 0.1 (5-sf wins)     | 0.01 | **0.1** |
| SOL         | 4          | $150        | 0.01 (5-sf wins)    | 0.01 | **0.01** |
| Some HIP-3 asset | 2     | $4,700      | 0.1 (5-sf wins)     | 0.0001 (decimal rule too fine) | **0.1** |

The single-rule formula `tick = 10^-(6 − szDecimals)` is what most clients use, and it's wrong for higher-priced assets — the sig-fig rule produces a coarser tick. Hardcoded clients silently round to a finer tick than the venue accepts and get cryptic rejections.

Compute the effective tick as `max(price-based-sigfig-tick, decimal-rule-tick)` and round price to that. Recompute when mark price moves enough to change the leading-digit position.

## TIF strings

Time-in-force values are **bare case-sensitive strings**:

| TIF        | Wire value |
| ---------- | ---------- |
| Good-til-cancel | `"Gtc"` |
| IOC        | `"Ioc"`    |
| Post-only (add-liquidity-only) | `"Alo"` |

`"GTC"`, `"IOC"` (uppercase), `"PostOnly"`, or `"PO"` are all rejected.

## Fill detection

The `userEvents` WebSocket channel streams fill, liquidation, and cancel events for the API wallet. Sub-100 ms typical. Reliable and the right primitive for a market-making or spread bot.

```python
async with info.subscribe({"type": "userEvents", "user": api_wallet_addr}) as sub:
    async for event in sub:
        if event["type"] == "fill":
            # event has coin, side, px, sz, time, hash
            ...
```

Use this rather than polling open-orders.

## L2 book fan-out

Market data for L2 depth is `l2Book` channel per coin. For multi-user products that watch the same markets, **pre-encode the payload once per emit and broadcast as bytes**, rather than re-encoding per subscriber. The naive `for q in subs: ws.send_json(payload)` pays JSON encoding N times for the same payload.

Also: throttle per-key at ~10 Hz at the multiplexer. `l2Book` emits 10–50 Hz on active markets; humans can't read faster than ~10 Hz. Per-key `last_sent_ts` plus a one-shot tail-flush task delivers the freshest snapshot at most once per 100 ms while never letting the final update get stuck. (Trades stay event-streamed; don't throttle those.)

## Common pitfalls

| Symptom                                                | Likely cause                                          |
| ------------------------------------------------------ | ----------------------------------------------------- |
| Order rejected with "asset not found"                  | HIP-3 dex not preloaded in `perp_dexs`                |
| Order placed but went to wrong account                 | `vault_address` not set on `Exchange` client          |
| "Invalid order size"                                   | Didn't round to `szDecimals` for this asset           |
| "Tick size" rejection on a higher-priced asset         | Used the decimal rule alone; sig-fig rule wins above ~$1k |
| Subaccount create returns "vault not found"            | Master account has zero trading history               |
| TIF rejected                                           | Used `"GTC"` instead of `"Gtc"`                       |
| Withdrawal request signed but ineffective              | Signed with the API wallet; only the main account can withdraw |

## Endpoint surface (informational)

The SDK abstracts most of this, but for reference:

```text
POST /info  — read-only queries (meta, l2Book, userEvents, ...)
POST /exchange  — actions (order, cancel, withdraw, subAccount, ...)
                  requires signed action body
```

Read the SDK source if there's any doubt about what's being signed — the action's exact JSON shape and the EIP-712 domain are venue-specific, and the SDK is the canonical reference.
