# Rialto Swap API Integration Guide

This document explains how approved partners can use Rialto's swap API to list
supported tokens, request swap quotes, and build executable swap transactions.

## Base URL

```text
https://rialto-trade-api.nirmaan.ai
```

## API Key Access

Request an API key from the Rialto team.

Protected endpoints require:

```http
Authorization: Bearer <api_key>
```

Example:

```bash
-H "Authorization: Bearer rialto_live_<prefix>.<secret>"
```

Keep API keys private. Do not expose partner API keys in public frontend code,
mobile apps, GitHub repositories, logs, or analytics tools.

## Supported Flow

The standard flow is:

1. Call `GET /tokens` to discover supported tokens.
2. Call `GET /quote` with sell token, buy token, amount, taker, and slippage.
3. Call `POST /swap` with the full quote response.
4. If the response includes a Permit2 payload, have the taker sign it.
5. Submit the returned transaction from the taker wallet.

## Public Endpoint: Tokens

`GET /tokens`

Auth: public

Returns tokens supported by the quote and swap APIs.

Example:

```bash
curl -sS 'https://rialto-trade-api.nirmaan.ai/tokens'
```

Example response shape:

```json
{
  "chain_id": 42161,
  "tokens": [
    {
      "name": "Wrapped Ether",
      "symbol": "WETH",
      "address": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
      "decimals": 18,
      "source": "whitelist",
      "type": "non_stable"
    }
  ]
}
```

Token fields:

| Field | Description |
| --- | --- |
| `name` | Token display name. |
| `symbol` | Token symbol. |
| `address` | Token contract address. |
| `decimals` | Token decimals. |
| `source` | How the token entered the supported set. |
| `type` | `stable` or `non_stable`. |

## Protected Endpoint: Quote

`GET /quote`

Auth: requires API key with quote access.

Returns the best route, expected output, route legs, platform fee, and estimated
network fee for an exact-input swap.

Required query params:

| Param | Description |
| --- | --- |
| `sell_token` | Token symbol or token address. |
| `buy_token` | Token symbol or token address. |
| `sell_amount` | Human decimal sell amount, for example `0.01`. |
| `taker` | Wallet address that will receive output. |
| `slippage_bps` | Max slippage in basis points. Example: `50` means 0.50%. |

Optional query params:

| Param | Description |
| --- | --- |
| `chain_id` | Chain id. Defaults to the server's configured chain. |
| `max_hops` | Max route hops. Defaults to the server's configured max. |

Example:

```bash
API_KEY='<api_key>'

curl -sS 'https://rialto-trade-api.nirmaan.ai/quote?sell_token=WETH&buy_token=USDC&sell_amount=0.01&taker=0xE968092b14829E5665a22531460Ad34012610F1f&slippage_bps=50' \
  -H "Authorization: Bearer $API_KEY"
```

Important response fields:

| Field | Description |
| --- | --- |
| `chain_id` | Chain id for the quote. |
| `sell_token` | Sell token address. |
| `buy_token` | Buy token address. |
| `sell_amount` | Raw integer sell amount in token smallest units. |
| `buy_amount` | Expected raw integer buy amount before slippage. |
| `platform_fee` | Platform fee breakdown. |
| `network_fee` | Estimated transaction gas cost, if available. |
| `taker` | Recipient/taker address. |
| `slippage_bps` | Slippage tolerance used for quote. |
| `route.legs` | Ordered route legs/pools. |

Example response shape:

```json
{
  "chain_id": 42161,
  "sell_token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
  "buy_token": "0xaf88d065e77cc2239327c5edb3a432268e5831",
  "sell_amount": "10000000000000000",
  "buy_amount": "30000000",
  "taker": "0xE968092b14829E5665a22531460Ad34012610F1f",
  "slippage_bps": 50,
  "route": {
    "sell_amount": "10000000000000000",
    "buy_amount": "30000000",
    "gas_estimate": 150000,
    "legs": [
      {
        "pool_id": "uniswap-v3:chain-42161:0xE968092b14829E5665a22531460Ad34012610F1f:500",
        "sell_token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
        "buy_token": "0xaf88d065e77cc2239327c5edb3a432268e5831",
        "sell_amount": "10000000000000000",
        "buy_amount": "30000000"
      }
    ]
  }
}
```

The exact amounts and route legs depend on live liquidity.

## Protected Endpoint: Swap

`POST /swap`

Auth: requires API key with swap access.

Builds executable transaction calldata from a prior quote. Send the full
`/quote` response in the request body.

Request body:

```json
{
  "quote": {
    "...": "full quote response object returned by /quote"
  },
  "deadline_secs": 300,
  "settlement": "permit2"
}
```

Common request fields:

| Field | Required | Description |
| --- | --- | --- |
| `quote` | Yes | Full response returned by `/quote`. |
| `deadline_secs` | No | Transaction deadline window. Defaults to `300`. |
| `settlement` | No | `permit2` or `allowance`. Defaults to `permit2`. |
| `unwrap_buy_token_to_eth` | No | Only valid when quote ends in wrapped native token. |
| `referral_code` | No | Optional partner referral code. |
| `permit2_nonce` | No | Optional Permit2 nonce override. |

Working example:

```bash
API_KEY='<api_key>'

curl -sS 'https://rialto-trade-api.nirmaan.ai/quote?sell_token=WETH&buy_token=USDC&sell_amount=0.01&taker=0xE968092b14829E5665a22531460Ad34012610F1f&slippage_bps=50' \
  -H "Authorization: Bearer $API_KEY" \
  -o quote.json

jq -n --slurpfile quote quote.json '{
  quote: $quote[0],
  deadline_secs: 300,
  settlement: "permit2"
}' > swap-request.json

curl -sS -X POST 'https://rialto-trade-api.nirmaan.ai/swap' \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  --data @swap-request.json
```

`quote.json` is the full JSON object returned by `/quote`. `swap-request.json`
is valid JSON that embeds that quote object under the `quote` field.

Important response fields:

| Field | Description |
| --- | --- |
| `settlement` | Settlement mode, usually `permit2`. |
| `quote.min_buy_amount` | Slippage-protected minimum output. |
| `tx.to` | Router contract address. |
| `tx.data` | Transaction calldata. |
| `tx.value` | Native token value to send, usually `0`. |
| `tx.estimated_gas` | Estimated gas units. |
| `tx.signature_offset` | Offset where Permit2 signature is inserted. |
| `permit2` | EIP-712 payload for taker signature, present for Permit2 settlement. |

Example response shape:

```json
{
  "settlement": "permit2",
  "quote": {
    "chain_id": 42161,
    "sell_token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
    "buy_token": "0xaf88d065e77cc2239327c5edb3a432268e5831",
    "sell_amount": "10000000000000000",
    "buy_amount": "30000000",
    "min_buy_amount": "29850000",
    "taker": "0xE968092b14829E5665a22531460Ad34012610F1f"
  },
  "tx": {
    "to": "0xE968092b14829E5665a22531460Ad34012610F1f",
    "data": "0x...",
    "value": "0",
    "estimated_gas": 270000,
    "signature_offset": 1234
  },
  "permit2": {
    "domain": {},
    "types": {},
    "primaryType": "PermitWitnessTransferFrom",
    "message": {},
    "nonce": "123",
    "deadline": 1760000000
  }
}
```

## Errors

Errors are returned as JSON:

```json
{ "error": "<message>" }
```

Common status codes:

| Status | Example Error | Meaning |
| --- | --- | --- |
| `400` | varies | Invalid params, unsupported token, no route, or swap-building error. |
| `401` | `unauthorized` | Missing, malformed, invalid, disabled, or expired API key. |
| `403` | `forbidden` | API key is valid but not allowed to call this endpoint. |
| `429` | `rate limit exceeded` | Per-key rate limit exceeded. |
| `500` | `internal server error` | Unexpected backend error. |

Examples:

```json
{ "error": "unauthorized" }
```

```json
{ "error": "forbidden" }
```

```json
{ "error": "rate limit exceeded" }
```

## Notes

- API keys are issued by the Rialto team.
- API keys may be scoped to quote-only or quote-and-swap access.
- API keys may have rate limits or expiry.
- Quotes depend on live liquidity and may become stale quickly.
- Partners should re-quote before building a swap if the user waits.
- Submit transactions from the taker wallet, not from the API server.
