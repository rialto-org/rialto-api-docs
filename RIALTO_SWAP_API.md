# Rialto Swap API Integration Guide

This document explains how approved partners can use Rialto's swap API to list
supported tokens, request swap quotes, and build executable swap transactions.

## Base URL

```text
https://rialto-trade-api.rialto.xyz
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
4. If the response includes a Permit2 payload, have the taker sign the returned
   EIP-712 `PermitWitnessTransferFrom` message.
5. Insert the signature into `tx.data` at `tx.signature_offset`.
6. Submit the returned transaction from the taker wallet.

`/swap` returns the complete transaction payload for the RialtoRouter contract.
Integrators do not need to build route calldata, pool calls, fees, Permit2
witnesses, or settlement actions themselves.

## Public Endpoint: Tokens

`GET /tokens`

Auth: public

Returns tokens supported by the quote and swap APIs.

Example:

```bash
curl -sS 'https://rialto-trade-api.rialto.xyz/tokens'
```

Example response shape:

```json
{
  "chain_id": 42161,
  "tokens": [
    {
      "name": "ETH",
      "symbol": "ETH",
      "address": "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
      "decimals": 18,
      "source": "constant",
      "type": "non_stable"
    },
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
| `taker` | Non-zero wallet address that will receive output. |
| `slippage_bps` | Max slippage in basis points. Example: `50` means 0.50%. |

`slippageBps` is also accepted as an alias for `slippage_bps`.

Optional query params:

| Param | Description |
| --- | --- |
| `chain_id` | Chain id. Defaults to the server's configured chain. |
| `max_hops` | Accepted for compatibility; route policy is controlled server-side. |

Example:

```bash
API_KEY='<api_key>'

curl -sS 'https://rialto-trade-api.rialto.xyz/quote?sell_token=WETH&buy_token=USDC&sell_amount=0.01&taker=0xE968092b14829E5665a22531460Ad34012610F1f&slippage_bps=50' \
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
  "buy_token": "0xaf88d065e77c8cc2239327c5edb3a432268e5831",
  "sell_amount": "10000000000000000",
  "buy_amount": "19800022",
  "platform_fee": {
    "total_bps": 5,
    "fees": [
      {
        "side": "source",
        "token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
        "symbol": "WETH",
        "decimals": 18,
        "bps": "5",
        "bps_x100": 500,
        "amount": "5000000000000",
        "amount_decimal": "0.000005",
        "recipient": "0xa86b9655644e2b76be863664eea7ac5db3f8fc89"
      }
    ]
  },
  "network_fee": {
    "token": "0xeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeeee",
    "symbol": "ETH",
    "decimals": 18,
    "gas": "226046",
    "gas_price": "20000000",
    "gas_price_gwei": "0.02",
    "amount": "4520920000000",
    "amount_gwei": "4520.92",
    "amount_eth": "0.00000452092"
  },
  "taker": "0xe968092b14829e5665a22531460ad34012610f1f",
  "slippage_bps": 50,
  "candidate_paths": 8,
  "successful_routes": 7,
  "failed_routes": 1,
  "route": {
    "sell_amount": "10000000000000000",
    "buy_amount": "19800022",
    "gas_estimate": 106046,
    "legs": [
      {
        "pool_id": "uniswap-v3:chain-42161:0x6f38e884725a116c9c7fbf208e79fe8828a2595f:100",
        "sell_token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
        "buy_token": "0xaf88d065e77c8cc2239327c5edb3a432268e5831",
        "sell_amount": "10000000000000000",
        "buy_amount": "19809926"
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

The quote's `taker` must be a non-zero wallet address. A zero-address taker is
rejected by `/swap`.

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
| `deadline_secs` | No | Transaction deadline window. Defaults to `300`; valid range is `30..=3600`. |
| `settlement` | No | `permit2` or `allowance`. Can be omitted; the backend returns the effective settlement mode. |
| `unwrap_buy_token_to_eth` | No | Only valid when quote ends in wrapped native token. |
| `permit2_nonce` | No | Optional Permit2 nonce override. |

Client-supplied fee or referral override fields are ignored. Fee and referral
attribution are resolved server-side from the quote `taker`.

Working example:

```bash
API_KEY='<api_key>'

curl -sS 'https://rialto-trade-api.rialto.xyz/quote?sell_token=WETH&buy_token=USDC&sell_amount=0.01&taker=0xE968092b14829E5665a22531460Ad34012610F1f&slippage_bps=50' \
  -H "Authorization: Bearer $API_KEY" \
  -o quote.json

jq -n --slurpfile quote quote.json '{
  quote: $quote[0],
  deadline_secs: 300,
  settlement: "permit2"
}' > swap-request.json

curl -sS -X POST 'https://rialto-trade-api.rialto.xyz/swap' \
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
| `tx.signature_offset` | Offset where the 65-byte Permit2 signature is inserted. Present for Permit2 settlement. |
| `permit2` | EIP-712 payload for taker signature, present for Permit2 settlement. |

Example response shape:

```json
{
  "settlement": "permit2",
  "quote": {
    "chain_id": 42161,
    "sell_token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
    "buy_token": "0xaf88d065e77c8cc2239327c5edb3a432268e5831",
    "sell_amount": "10000000000000000",
    "buy_amount": "19800022",
    "min_buy_amount": "19701021",
    "taker": "0xe968092b14829e5665a22531460ad34012610f1f",
    "slippage_bps": 50,
    "candidate_paths": 8,
    "successful_routes": 7,
    "failed_routes": 1,
    "route": {
      "sell_amount": "10000000000000000",
      "buy_amount": "19800022",
      "gas_estimate": 106046,
      "legs": [
        {
          "pool_id": "uniswap-v3:chain-42161:0x6f38e884725a116c9c7fbf208e79fe8828a2595f:100",
          "sell_token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
          "buy_token": "0xaf88d065e77c8cc2239327c5edb3a432268e5831",
          "sell_amount": "10000000000000000",
          "buy_amount": "19809926"
        }
      ]
    }
  },
  "tx": {
    "to": "0xbb6c13e7e1649736dc36cf7071b7d63103fe351d",
    "data": "0x...",
    "value": "0",
    "estimated_gas": 226046,
    "signature_offset": 548
  },
  "permit2": {
    "domain": {
      "chainId": 42161,
      "name": "Permit2",
      "verifyingContract": "0x000000000022d473030f116ddee9f6b43ac78ba3"
    },
    "types": {
      "PermitWitnessTransferFrom": [
        { "name": "permitted", "type": "TokenPermissions" },
        { "name": "spender", "type": "address" },
        { "name": "nonce", "type": "uint256" },
        { "name": "deadline", "type": "uint256" },
        { "name": "witness", "type": "RialtoSwap" }
      ]
    },
    "primaryType": "PermitWitnessTransferFrom",
    "message": {
      "permitted": {
        "token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
        "amount": "10000000000000000"
      },
      "spender": "0xbb6c13e7e1649736dc36cf7071b7d63103fe351d",
      "witness": {
        "recipient": "0xe968092b14829e5665a22531460ad34012610f1f",
        "buyToken": "0xaf88d065e77c8cc2239327c5edb3a432268e5831",
        "minBuyAmount": "19701021",
        "feeRecipient": "0xa86b9655644e2b76be863664eea7ac5db3f8fc89",
        "srcBps": 5,
        "dstBps": 0,
        "referralCode": "0x0000000000000000000000000000000000000000000000000000000000000000",
        "quoteId": "0xdc61621af402cf7b8975cc41a97ff21ad2d81f528c0e490cf0e252768b8cd7f4",
        "actionsHash": "0x377090963ca2d0458a675da69e0f848da4d82adc4a066b5ad194a0572c2e50d9"
      }
    },
    "nonce": "102222442441090043455808165797985926558",
    "deadline": 1780300214
  }
}
```

## Integrator Swap Flow

This section is for wallets, frontends, and other routers that want to integrate
Rialto as an execution venue.

The integration boundary is simple: call `/quote`, post that quote to `/swap`,
then use the returned transaction object. `/swap` returns calldata for
RialtoRouter and, when required, the exact EIP-712 Permit2 witness the user must
sign. The integrator does not need to touch route encoding, pool calldata, fee
calculation, slippage math, or Permit2 witness construction.

### Permit2 settlement

For normal EOA ERC20 swaps, `/swap` usually returns:

```json
{
  "settlement": "permit2",
  "tx": {
    "to": "0x<rialto-router>",
    "data": "0x<calldata-with-signature-placeholder>",
    "value": "0",
    "estimated_gas": 226046,
    "signature_offset": 548
  },
  "permit2": {
    "domain": { "...": "..." },
    "types": { "...": "..." },
    "primaryType": "PermitWitnessTransferFrom",
    "message": { "...": "..." },
    "nonce": "102222442441090043455808165797985926558",
    "deadline": 1780300214
  }
}
```

Execution steps:

1. Ask the taker wallet to sign the `permit2` object as EIP-712 typed data.
2. Replace 65 bytes in `tx.data` at `tx.signature_offset` with the returned
   signature.
3. Send the transaction from the taker wallet using:
   - `to = tx.to`
   - `data = patched tx.data`
   - `value = tx.value`
   - `gas` from wallet estimation, optionally seeded with `tx.estimated_gas`

The EIP-712 message is a Permit2 `PermitWitnessTransferFrom` payload. Its
`witness` binds the user's signature to the exact Rialto swap parameters:
recipient, buy token, minimum output, deadline, fee fields, referral hash,
quote id, and route `actionsHash`. This prevents the signature from being
reused for a different route or different output constraints.

Example TypeScript signature patch:

```ts
function splicePermit2Signature(
  txData: string,
  signatureOffset: number,
  signature: string
): string {
  const data = txData.startsWith("0x") ? txData.slice(2) : txData;
  const sig = signature.startsWith("0x") ? signature.slice(2) : signature;

  if (sig.length !== 130) {
    throw new Error("Permit2 signature must be 65 bytes");
  }

  const start = signatureOffset * 2;
  const end = start + sig.length;
  return `0x${data.slice(0, start)}${sig}${data.slice(end)}`;
}
```

### Allowance settlement

`/swap` may return allowance settlement instead of Permit2. This happens when
the backend determines Permit2 is not the right execution path, for example for
native ETH sells or smart-contract-wallet takers.

Allowance response shape:

```json
{
  "settlement": "allowance",
  "tx": {
    "to": "0x<rialto-router>",
    "data": "0x<calldata>",
    "value": "0",
    "estimated_gas": 226046
  }
}
```

If `permit2` is absent, do not ask the user for a Permit2 signature and do not
modify `tx.data`.

For ERC20 sells, the taker must approve `tx.to` for at least the raw
`quote.sell_amount`, then send the returned transaction. For native ETH sells,
send the returned transaction with `tx.value`; no ERC20 approval or Permit2
signature is needed.

### Simulation and submission

Integrators can simulate the returned transaction exactly as they would submit
it:

- Permit2 mode: simulate after inserting the user's signature into `tx.data`.
- Allowance mode: simulate `tx.data` unchanged, after accounting for the
  taker's ERC20 allowance if the sell token is not native ETH.
- Native ETH sells: include `tx.value` in simulation and submission.

The transaction should be sent by the quote `taker` wallet. The API server
never signs or broadcasts transactions.

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
