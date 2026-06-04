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
| `swap_fee_bps` | Integrator fee in basis points. Requires an integrator key — see [Integrator Fees](#integrator-fees). |

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
attribution are resolved server-side from the quote `taker` (or, for integrator
swaps, from the integrator key — see [Integrator Fees](#integrator-fees)).

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

## Integrating the Swap Flow

This section is for wallets, frontends, and other routers that want to integrate
Rialto as an execution venue. The integration boundary is intentionally small:
call `/quote`, post that quote to `/swap`, then submit the transaction `/swap`
returns. You never build route calldata, pool calls, fee logic, slippage math, or
Permit2 witnesses yourself — Rialto does all of it.

### The two calls

| Endpoint | What it returns | What you do with it |
| --- | --- | --- |
| `GET /quote` | Best route, expected output, fee breakdown, gas estimate. | Display pricing to the user; pass the whole response to `/swap`. |
| `POST /swap` | A ready-to-send transaction for the RialtoRouter contract, plus (for Permit2) the EIP-712 message the user signs. | Sign if required, then submit from the taker wallet. |

Send the **entire** `/quote` response as the `quote` field of the `/swap` body —
do not modify or trim it.

### What `/swap` returns

The payload to send on-chain lives in the `tx` object:

| Field | Description |
| --- | --- |
| `tx.to` | RialtoRouter contract address — the `to` of your transaction. |
| `tx.data` | ABI-encoded calldata for the swap. |
| `tx.value` | Native value to send (`0` for ERC20 sells; the sell amount for native ETH sells). |
| `tx.estimated_gas` | Gas estimate you can seed wallet estimation with. |
| `tx.signature_offset` | Byte offset in `tx.data` where the taker's 65-byte Permit2 signature is inserted. Present only for Permit2 settlement. |
| `permit2` | EIP-712 typed-data message the taker signs. Present only for Permit2 settlement. |

The `settlement` field tells you which of the two modes below applies. Choose your
handling off `settlement`, not off assumptions — the backend decides the mode.

### Permit2 settlement (default for ERC20 sells)

Permit2 lets a user authorize a single, exact token transfer by **signing a
message** instead of sending a separate approval transaction. Rialto uses Permit2
with a **witness**: the signed EIP-712 message is bound to the precise swap —
recipient, buy token, minimum output, deadline, quote id, and the hash of the
route actions. That binding means the signature cannot be replayed for a
different route, output, or recipient.

A Permit2 response looks like:

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

Steps:

1. Have the taker wallet sign the `permit2` object as EIP-712 typed data.
2. Splice the returned 65-byte signature into `tx.data` at `tx.signature_offset`.
3. Send the transaction from the taker wallet (`to`, patched `data`, `value`).

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
  return `0x${data.slice(0, start)}${sig}${data.slice(start + sig.length)}`;
}
```

### Allowance settlement

`/swap` returns `settlement: "allowance"` (no `permit2` object, no
`signature_offset`) when Permit2 is not the right path — for example native ETH
sells, or a smart-contract-wallet taker that cannot produce an EOA Permit2
signature.

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

- **ERC20 sells:** the taker approves `tx.to` for at least the raw
  `quote.sell_amount`, then sends the transaction. Do not modify `tx.data`.
- **Native ETH sells:** send the transaction with `tx.value`; no ERC20 approval
  and no signature are needed.

### Simulation

Rialto already simulates every route. During quoting, each candidate route is
executed against live chain state via `eth_call` on the router's simulation
entrypoint, and a route is only returned if it actually executes and produces a
valid output. So the transaction `/swap` hands you is **pre-validated** — its
shape and output were checked on-chain, not merely encoded.

You can also simulate the final transaction yourself before submitting, by
running it as an `eth_call` from the taker address:

- **Permit2 mode:** insert the signature first, then simulate the patched
  `tx.data`.
- **Allowance mode:** simulate `tx.data` unchanged, with the taker's ERC20
  allowance in place (or `tx.value` for native ETH).

A successful simulation confirms the route executes and the output meets
`quote.min_buy_amount`. A revert means the quote is stale or the route no longer
fills — re-quote and rebuild.

### Submission

Submit the transaction from the **taker wallet**. The Rialto API never signs or
broadcasts transactions. Quotes reflect live liquidity and can go stale quickly,
so re-quote before building if the user waits.

### End-to-end example

```bash
API_KEY='<api_key>'

# 1. Quote
curl -sS 'https://rialto-trade-api.rialto.xyz/quote?sell_token=WETH&buy_token=USDC&sell_amount=0.01&taker=0xE968092b14829E5665a22531460Ad34012610F1f&slippage_bps=50' \
  -H "Authorization: Bearer $API_KEY" -o quote.json

# 2. Build the swap from that quote
jq -n --slurpfile quote quote.json '{ quote: $quote[0], deadline_secs: 300 }' > swap-request.json

curl -sS -X POST 'https://rialto-trade-api.rialto.xyz/swap' \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  --data @swap-request.json -o swap.json

# 3. In your app:
#    - Permit2:    sign swap.permit2 (EIP-712) -> splice the signature into
#                  swap.tx.data at swap.tx.signature_offset
#    - Allowance:  approve swap.tx.to for quote.sell_amount (ERC20 sells)
# 4. (Optional) eth_call swap.tx from the taker to simulate
# 5. Submit swap.tx from the taker wallet
```

## Integrator Fees

Approved integrators (wallets, frontends, and other apps) can charge **their own
fee on top of Rialto's** on every swap they route. Your fee is collected as part
of the swap and paid **to your wallet in the same on-chain transaction** — there
is no separate claim step, no fee contract to deploy, and no settlement to run
yourself.

### How it works at a glance

1. You request a quote with a `swap_fee_bps` value — your fee, in basis points.
2. Rialto adds your fee on top of its own and returns the full breakdown in the
   quote.
3. You post that quote to `/swap`. Rialto builds one transaction that, on
   execution, pays both Rialto's treasury and **your wallet** atomically.
4. The taker submits the transaction. Your fee lands in your wallet the moment
   the swap settles.

Your fee is **additive** — Rialto charges its standard fee, and your fee is taken
on top of it. Rialto does not take any cut of your portion.

### Getting set up

Integrator access is configured by the Rialto team. [Contact us](#api-key-access)
to be onboarded. We issue you an API key and bind three things to that key:

| Bound to your key | Meaning |
| --- | --- |
| **Payout wallet** | The address your fees are sent to. |
| **Maximum fee (bps)** | A per-key cap — the largest `swap_fee_bps` you may set. |
| **Integrator id** | An identifier used to attribute the swaps you route. |

Because these are bound to the **key** and not to the request, a leaked or
tampered request can never redirect your fees to another wallet or push your fee
above your agreed cap.

> Keep your integrator key private, the same as any other API key. Anyone with
> the key can route swaps under your integrator id (but still only ever pay your
> configured wallet, up to your cap).

### Setting your fee on `GET /quote`

Add `swap_fee_bps` to the standard quote request. Everything else is identical to
a normal quote.

| Param | Description |
| --- | --- |
| `swap_fee_bps` | Your fee in basis points, applied on top of Rialto's fee. `30` = 0.30%. `swapFeeBps` is also accepted. |

```bash
INTEGRATOR_KEY='<your_api_key>'

curl -sS 'https://rialto-trade-api.rialto.xyz/quote?sell_token=WETH&buy_token=USDC&sell_amount=0.01&taker=0xE968092b14829E5665a22531460Ad34012610F1f&slippage_bps=50&swap_fee_bps=30' \
  -H "Authorization: Bearer $INTEGRATOR_KEY"
```

You only need to send `swap_fee_bps`. Your payout wallet and cap come from your
key, so you do not pass them on the request.

### Reading the fee in the quote response

A quote that includes an integrator fee gains two things:

1. An **`integrator_fee`** block — your fee, your payout wallet, and your
   integrator id, echoed back so you can display or reconcile it.
2. Extra entries in **`platform_fee.fees`** — one line per recipient. Each line
   shows the **token the fee is taken in** and the **exact amount**, so you always
   know precisely what your cut will be for that swap.

```json
{
  "buy_amount": "19800022",
  "platform_fee": {
    "total_bps": 80,
    "fees": [
      {
        "side": "source",
        "token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
        "symbol": "WETH",
        "decimals": 18,
        "bps": "50",
        "amount": "50000000000000",
        "amount_decimal": "0.00005",
        "recipient": "0xa86b9655644e2b76be863664eea7ac5db3f8fc89"
      },
      {
        "side": "source",
        "token": "0x82af49447d8a07e3bd95bd0d56f35241523fbab1",
        "symbol": "WETH",
        "decimals": 18,
        "bps": "30",
        "amount": "30000000000000",
        "amount_decimal": "0.00003",
        "recipient": "0x23ecad7a98ce5ec6ccaac8bc01ae8d3bd07dd7a6"
      }
    ]
  },
  "integrator_fee": {
    "bps": 30,
    "recipient": "0x23ecad7a98ce5ec6ccaac8bc01ae8d3bd07dd7a6",
    "id": "your-integrator-id"
  }
}
```

In this example Rialto's fee is 50 bps and your fee is 30 bps, for a combined
`total_bps` of 80. The `recipient` on the second line is **your** payout wallet.
The `token` and `amount` fields tell you exactly which token the fee is taken in
and how much — Rialto selects the most suitable token in the swap automatically,
so you do not need to choose a fee token.

### Building the swap

Post the full quote to `/swap` using the **same integrator key**:

```bash
curl -sS -X POST 'https://rialto-trade-api.rialto.xyz/swap' \
  -H "Authorization: Bearer $INTEGRATOR_KEY" \
  -H "Content-Type: application/json" \
  --data @swap-request.json
```

`/swap` re-checks your fee against your key — it reads the fee size from the
quote, re-applies your cap, and takes your payout wallet from the key (never from
the quote). It then returns a single transaction that pays Rialto and your wallet
as part of the swap. From here the execution flow is exactly the same as a normal
swap (Permit2 or allowance settlement) — see the [Swap](#protected-endpoint-swap)
section. You do not build any fee logic yourself.

> Use the **same key** for `/quote` and `/swap`. The integrator fee is only
> honored when both calls are made with your integrator key.

### Caps and rules

| Rule | Behavior |
| --- | --- |
| Fee above your key's cap | `swap_fee_bps` greater than your configured maximum is rejected with `400`. |
| Combined fee too high | Rialto's fee **plus** your fee must not exceed the protocol maximum (currently **100 bps** total). Over the limit is rejected with `400`. |
| Non-integrator key | A key without integrator access that sends `swap_fee_bps` is rejected with `403`. |
| No fee requested | Omit `swap_fee_bps` (or send `0`) and the swap behaves as a standard swap with no integrator fee. |

### Summary

- Onboard once; we bind your payout wallet, cap, and id to your key.
- Add `swap_fee_bps` to `/quote` to set your fee per swap.
- Read `integrator_fee` and `platform_fee` to know exactly what you'll earn.
- Post the quote to `/swap` and submit the transaction — your fee is paid to your
  wallet atomically, on every swap.

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
