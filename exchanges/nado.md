# Nado (Vertex fork on Ink L2)

Nado is a Vertex Protocol fork deployed on Ink (Kraken's Ethereum L2, chain `57073`). Architecture: off-chain matching engine, on-chain settlement, EIP-712 signed orders. Vertex SDK semantics carry over **mostly**, with some venue-specific quirks listed below.

This doc covers what was non-obvious from the docs alone. Read it alongside, not instead of, the official Vertex / Nado documentation.

## Hosts and chain IDs

| Network          | Gateway                              | Chain ID |
| ---------------- | ------------------------------------ | -------- |
| Ink Mainnet      | `https://gateway.prod.nado.xyz/v1`   | 57073    |
| Ink Sepolia (testnet) | (check venue's docs)            | 763373   |

**The chain ID is part of the EIP-712 domain.** Wrong chain ID does not produce a "wrong network" error — it produces a "signature does not match sender" error with a recovered address that doesn't look like anyone's. We've hit this twice; both times the error message was misleading. Hard-fail at startup if the chain ID doesn't match expected, and log the chain ID on every signing operation during integration.

## Product IDs — the spot/perp trap

Product IDs follow a pattern that is NOT obvious from the docs in the fork variant:

- **Odd numbers = Spot**: 1 = BTC spot, 3 = ETH spot, 5 = other spot
- **Even numbers = Perps**: 2 = BTC-PERP, 4 = ETH-PERP

We accidentally traded ETH **spot** (product_id 3) instead of ETH-PERP (product_id 4) for an entire session. The order placed cleanly, no warning. We only caught it by inspecting balances and seeing borrowed USDT0 with a wETH spot position — a completely different risk profile than a perp.

**Always** query `all_products` on startup and match by name (or by the `product_type` field), not by hardcoded ID. Validate the product type before sending any order.

```text
POST /query {"type": "all_products"}
→ data.perp_products[].product_id  // perps
→ data.spot_products[].product_id  // spot — don't confuse
```

## Price and quantity scaling (X18)

All prices and quantities use **X18 scaling** (multiply by 10^18):

| Display    | Wire value                             |
| ---------- | -------------------------------------- |
| $2148.50   | `2148500000000000000000`               |
| 0.05 ETH   | `50000000000000000`                    |
| $100 notional minimum | `100000000000000000000`      |

Common mistake: forgetting to scale when computing notional checks. `amount × price` must be compared against `min_size` **in X18 space**, not human-readable units.

## Minimum order size

The minimum is **notional-based**, not quantity-based:

```text
abs(amount) × price >= min_size
```

For BTC-PERP and ETH-PERP, `min_size = 100000000000000000000` ($100 notional). At ETH ~$2150, minimum qty ≈ 0.047 ETH (use 0.05). At BTC ~$69000, minimum qty ≈ 0.00145 BTC (use 0.002). Fetch and check the increment too:

- ETH-PERP size increment: `1000000000000000` = 0.001 ETH
- ETH-PERP price increment: `100000000000000000` = $0.10

## EIP-712 domain

```text
Domain {
    name: "Nado",       // NOT "Vertex" — even though the codebase is a Vertex fork
    version: "0.0.1",
    chainId: 57073,     // Ink Mainnet
    verifyingContract: <endpoint_address>  // fetch dynamically
}
```

The `verifyingContract` must be **fetched dynamically**:

```text
POST /query {"type": "contracts"}
→ data.endpoint_addr
```

Do not hardcode this address. It can change between deployments.

## link_signer (delegated trading keys)

For platforms using a delegation pattern (a primary key signs to authorize a secondary key to trade on a subaccount):

**Subaccount bytes32 format**: `bytes20(owner_wallet) + bytes12(ascii_name_padded)`

```text
owner:  0xabc...123  (20 bytes)
name:   "myname"     (ASCII, RIGHT-padded with zero bytes to 12 bytes)
result: 0xabc...123 6d796e616d6500000000000000
```

**Signer bytes32 format**: `bytes20(signer_address) + bytes12(zeros)` — the address is **LEFT-most 20 bytes**, zeros in the **RIGHT 12 bytes**.

The Vertex SDK's `subaccount_to_bytes32()` is the reference implementation. Be careful with `rjust` vs `ljust`: left-padding (`rjust(32, b'\0')`) is wrong here and is what we got the first time, costing ~8 hours to debug.

**Nonce precision warning.** The nonce for `LinkSigner` is a Unix timestamp in nanoseconds (~1.85 × 10^18), which **exceeds JavaScript's `Number.MAX_SAFE_INTEGER`** (9.0 × 10^15). If your nonce flows through any JS layer:

- JSON serialization: send as **string**.
- Frontend BigInt for signing: parse the string back to `BigInt`.
- Don't put it in `Number` anywhere in JS.

Python and Rust have no issue (use `int` or `u128`).

**link_signer scope is per-subaccount, not per-product.** Once linked, the signer key works for all products on that subaccount. Re-linking with a different signer revokes the previous one atomically.

## The "default" subaccount is the empty string

On Vertex-family venues, a wallet's primary/default subaccount uses `subaccount_name=""` (empty). The bytes32 sender is `bytes20(wallet) + bytes12(zeros)` — same encoding as any named subaccount, with a zero-length name.

**Do not synthesize** a name like `"main"`, `"default"`, or substitute the wallet address. All three produce a different bytes32, which the venue treats as a different subaccount, with different balances and orders. Most accounts will then look "empty" because you're looking at the wrong subaccount.

## Account health and order rejection

The venue rejects orders that would lower initial account health below zero. This is a **pre-trade check** — the order never hits the book:

```text
error code 2006: "Insufficient account health"
```

For market-makers and spreaders: if your passive quote fills but the hedge rejects because health dropped below threshold, you have an unhedged directional position. Always check health **before placing quotes**, not just on hedge.

```text
POST /query {"type": "subaccount_info", "subaccount": "0x..."}
→ data.healths[0].health  // initial health (for new orders)
→ data.healths[1].health  // maintenance health (for liquidation)
```

Health values are X18-scaled. A value of `3000000000000000000` = $3.00 initial health remaining.

**Position-aware health**: low health should NOT block reduce-only orders. If you're long, you can still sell to reduce. Query `perp_balances` to know current position direction and gate accordingly.

## Oracle price bounds

The venue rejects orders priced outside **20% to 500% of oracle price**:

```text
error: "Order price must be within a range of 20% to 500% of oracle price"
```

This catches hedge orders priced from stale or garbage market data. Always recompute hedge prices from fresh market data, not from a snapshot taken when the fill was first detected. The oracle price is in the `all_products` response.

## Market data

REST polling endpoint for L2 depth:

```text
POST /query {"type": "market_liquidity", "product_id": 4, "depth": 1}
→ data.bids[0][0]  // best bid price (X18)
→ data.asks[0][0]  // best ask price (X18)
```

The WebSocket protocol is documented for some products (BTC-PERP reliably) but underdocumented for others. In our integration we use REST polling for ETH-PERP and other less-trafficked markets. Build with a polling fallback by default; the WS is an optimization, not the source of truth.

## Fill detection

There is no native fill stream we found reliable across products. The working pattern is to poll `subaccount_orders` and diff against the previously-known open orders: any digest that disappeared was either filled or cancelled, and you cross-reference against your own cancel requests to distinguish.

```text
POST /query {"type": "subaccount_orders", "subaccount": "0x..."}
→ data.orders[]    // current open
```

For tight market-making, the polling cycle latency is meaningful; expect ~250 ms end-to-end fill detection. Hedge logic should be designed around that ceiling.

## Common error codes

| Code | Meaning                          | Common cause                                 |
| ---- | -------------------------------- | -------------------------------------------- |
| 2006 | Insufficient account health      | Not enough margin                            |
| 2024 | No previous deposits             | Wrong subaccount ref, or signer not linked   |
| 2094 | Invalid order size               | Below the $100 min notional                  |
| —    | Signature mismatch               | Wrong chain ID, wrong key, or wrong domain name |
| —    | Price out of range               | Outside 20–500% of oracle price              |

## Endpoint reference

```text
POST /execute   — place/cancel orders (requires EIP-712 signature)
POST /query     — read-only queries (no auth needed)

Query types:
  all_products, market_liquidity, subaccount_info,
  subaccount_orders, linked_signer, contracts
```

## Common debugging checklist

When a signed action is rejected:

1. **Print the recovered address** that the gateway claims. If it's not your address, the EIP-712 domain or message is wrong — not your private key.
2. **Verify the chain ID** matches the network you're talking to. Wrong chain ID is the single most common cause.
3. **Verify the domain name** is `"Nado"` (or whatever the specific fork uses), not `"Vertex"`.
4. **Verify the bytes32 encoding** of subaccount and signer fields. `bytes20(address) + bytes12(zeros|name_ascii)` is the rule; the address is the LEFT 20 bytes.
5. **Verify the `verifyingContract`** was fetched fresh from `/query contracts`, not hardcoded.

If the action is rejected for non-signature reasons:

1. Check `min_size` notional in X18 (not display units).
2. Check oracle bounds (20%–500% of oracle).
3. Check `account_health` is positive after the order's hypothetical fill.
4. For new accounts: confirm a deposit has been made first. Without prior deposits, even valid actions are rejected (error 2024).
