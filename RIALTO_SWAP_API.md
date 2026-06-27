# Rialto Swap API Integration Guide

This document explains how approved partners can use Rialto's swap API to list
supported tokens, request firm swap quotes, and execute the returned transaction.

## Base URL

The public API base URL below for Rialto:

```text
https://rialto-trade-api.rialto.xyz
```

## Router Registries

Rialto publishes Router Registry contracts so partners can discover the active
Rialto router instead of hardcoding router addresses.

Use feature id `2` when you submit the transaction returned by `GET /quote`
from the taker wallet. Use feature id `3` when you use Rialto's gasless relayer
flow through `POST /gasless/submit`.

| Feature id | Flow |
| --- | --- |
| `2` | Taker-submitted / normal swap |
| `3` | Gasless / relayed swap |

Registry deployments:

| Chain | Chain id | API access | Router Registry |
| --- | --- | --- | --- |
| RH Chain | `chain_id` | `<your_robinhood_rpc_url>` | `0x71a120CbBf3Ce7cD910a3c50fF77aFc62735687E` |

Registry reads:

| Function | Meaning |
| --- | --- |
| `ownerOf(uint256 featureId)` | Current active router for the feature. Reverts when the feature is paused or uninitialized. |
| `prev(uint128 featureId)` | Previous router accepted during a migration dwell window. Reverts when no previous router is set. |
| `next(uint128 featureId)` | Staged next router, or the zero address when none is staged. |
| `getFeature(uint128 featureId)` | One-call read returning `(previous, current, next, paused)`. |

In this registry, "owner" means the router address currently assigned to a
feature id. It is not the registry admin owner.

Raw JSON-RPC `curl` examples:

```bash
# Robinhood Chain:
# Use your own Robinhood Chain RPC endpoint if you need direct registry reads.
# RPC_URL='<your_robinhood_rpc_url>'
# REGISTRY_ADDRESS='0x71a120CbBf3Ce7cD910a3c50fF77aFc62735687E'

# Feature 2 = taker-submitted quote transaction.
# Feature 3 = gasless relayer flow.
FEATURE_ID=2
FEATURE_HEX=$(printf '%064x' "$FEATURE_ID")

# Current router: ownerOf(uint256)
DATA="0x6352211e${FEATURE_HEX}"
curl -sS "$RPC_URL" \
  -H 'Content-Type: application/json' \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"eth_call\",\"params\":[{\"to\":\"$REGISTRY_ADDRESS\",\"data\":\"$DATA\"},\"latest\"]}"

# Previous router: prev(uint128)
DATA="0xe2603dc2${FEATURE_HEX}"
curl -sS "$RPC_URL" \
  -H 'Content-Type: application/json' \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"eth_call\",\"params\":[{\"to\":\"$REGISTRY_ADDRESS\",\"data\":\"$DATA\"},\"latest\"]}"

# Staged next router: next(uint128)
DATA="0x74bcde51${FEATURE_HEX}"
curl -sS "$RPC_URL" \
  -H 'Content-Type: application/json' \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"eth_call\",\"params\":[{\"to\":\"$REGISTRY_ADDRESS\",\"data\":\"$DATA\"},\"latest\"]}"

# Full feature state: getFeature(uint128)
DATA="0x46df01a4${FEATURE_HEX}"
curl -sS "$RPC_URL" \
  -H 'Content-Type: application/json' \
  -d "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"eth_call\",\"params\":[{\"to\":\"$REGISTRY_ADDRESS\",\"data\":\"$DATA\"},\"latest\"]}"
```

`eth_call` returns ABI-encoded hex. For single-address calls, the address is the
last 20 bytes of the returned 32-byte word. For `getFeature`, decode four ABI
words as `(address previous, address current, address next, bool paused)`.

## API Key Access

Partners can request an integrator API key through the self-service flow in
[Requesting an Integrator API Key](#requesting-an-integrator-api-key).

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

Rialto supports two execution flows.

### Taker-submitted flow

Use this when the user's wallet will submit the on-chain transaction and pay
gas:

1. Call `GET /tokens` to discover supported tokens.
2. Call `GET /quote` with sell token, buy token, amount, taker, and slippage.
3. Inspect `quote.issues` and ask the user to approve if `issues.allowance` is
   present.
4. If the quote includes a Permit2 payload, have the taker sign the returned
   EIP-712 `PermitWitnessTransferFrom` message.
5. Insert the signature into `tx.data` at `tx.signature_offset`.
6. Submit the returned `tx` from the taker wallet.

### Gasless relayer flow

Use this when the user signs Permit2 and Rialto broadcasts the transaction:

1. Call `GET /quote` with the normal quote params plus
   `permit2_owner=<taker wallet>`.
2. Inspect `quote.issues`. The taker must have enough balance and enough
   allowance to Permit2 before relay submission.
3. Have the taker sign the returned EIP-712 `permit2` typed data.
4. Send `{ quote_id, signature, idempotency_key? }` to `POST /gasless/submit`.
   Use a unique order id or UUID for `idempotency_key` when you want retries to
   return the same relay request.
5. Poll `GET /gasless/status/{relay_id}` until the status is terminal.

Gasless supports ERC20 Permit2 quotes. Native ETH sells still require the
taker-submitted flow because ETH must be sent as `tx.value`.

`/quote` returns the complete transaction payload for the RialtoRouter contract.
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
  "chain_id": chain_id,
  "tokens": [
    {
      "name": "ETH",
      "symbol": "ETH",
      "address": "<native_eth_sentinel>",
      "decimals": 18,
      "source": "constant",
      "type": "non_stable"
    },
    {
      "name": "Wrapped Ether",
      "symbol": "WETH",
      "address": "<sell_token_address>",
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

Returns the best route, expected output, route legs, platform fee, estimated
network fee, and an executable transaction for an exact-input swap.

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
| `max_hops` | Optional route-depth hint; final route policy is controlled server-side. |
| `permit2_owner` | Optional non-zero Permit2 token owner for gasless/relayed ERC20 swaps. For gasless, this must equal `taker`. Also accepts `permit2Owner` and `permitOwner`. |
| `swap_fee_bps` | Integrator fee in basis points. Requires an integrator key — see [Integrator Fees](#integrator-fees). |
| `swap_fee_token` | Optional fee-token hint. Fees are charged in the token selected by Rialto. |
| `swap_fee_recipient` | Ignored. The payout wallet is bound to the API key. |

Example:

```bash
API_KEY='rialto_live_example.redacted_secret'

curl -sS 'https://rialto-trade-api.rialto.xyz/quote?sell_token=WETH&buy_token=USDG&sell_amount=0.01&taker=<taker_wallet_address>&slippage_bps=50' \
  -H "Authorization: Bearer $API_KEY"
```

Important response fields:

| Field | Description |
| --- | --- |
| `chain_id` | Chain id for the quote. |
| `quote_id` | UUID handle for the server-stored quote. This is bound into the Permit2 witness as a bytes32 `quoteId`, so the signature is tied to this exact quote. |
| `settlement` | Execution mode: `permit2` or `allowance`. |
| `tx` | Ready-to-send transaction object. In direct flow, send this from the taker wallet after any required approval/signature step. In gasless flow, Rialto submits this transaction after `/gasless/submit`. |
| `permit2` | EIP-712 typed-data payload. Present only for Permit2 settlement. For gasless quotes, `permit2.owner` is the token owner that must sign. |
| `issues` | Frontend blockers and execution hints for balance and allowance. |
| `sell_token` | Sell token address. |
| `buy_token` | Buy token address. |
| `sell_amount` | Raw integer sell amount in token smallest units. |
| `buy_amount` | Expected raw integer buy amount before slippage. |
| `min_buy_amount` | Slippage-protected raw minimum buy amount encoded into `tx.data`. |
| `platform_fee` | Platform fee breakdown. |
| `integrator_fee` | Present when an integrator fee was accepted for this key. |
| `referral` / `referral_fee` | Present when referral attribution or referral fee policy applies. |
| `network_fee` | Estimated transaction gas cost, if available. |
| `taker` | Recipient/taker address. |
| `slippage_bps` | Slippage tolerance used for quote. |
| `route.legs` | Ordered route legs/pools. |

`issues` fields:

| Field | Meaning |
| --- | --- |
| `issues.balance` | Non-null when the taker does not have enough sell-token balance. Do not submit the transaction. |
| `issues.allowance` | Non-null when an ERC20 approval is needed before submitting `tx`. Approve `issues.allowance.spender` for at least `sell_amount`. |
| `issues.simulationIncomplete` | `true` when the backend could not complete all quote simulation checks. |
| `issues.invalidSourcesPassed` | Reserved field; currently an empty array. |

Example response shape:

```json
{
  "quote_id": "b7b0a3d8-9f6a-4f4a-92d2-1b0e4b5d0c6a",
  "settlement": "permit2",
  "tx": {
    "to": "<rialto_router_address_from_quote>",
    "data": "0xdbefe2d6...",
    "value": "0",
    "estimated_gas": 255053,
    "signature_offset": 548
  },
  "permit2": {
    "domain": {
      "name": "Permit2",
      "chainId": chain_id,
      "verifyingContract": "<permit2_contract_address>"
    },
    "types": {
      "EIP712Domain": [
        { "name": "name", "type": "string" },
        { "name": "chainId", "type": "uint256" },
        { "name": "verifyingContract", "type": "address" }
      ],
      "PermitWitnessTransferFrom": [
        { "name": "permitted", "type": "TokenPermissions" },
        { "name": "spender", "type": "address" },
        { "name": "nonce", "type": "uint256" },
        { "name": "deadline", "type": "uint256" },
        { "name": "witness", "type": "RialtoSwap" }
      ],
      "TokenPermissions": [
        { "name": "token", "type": "address" },
        { "name": "amount", "type": "uint256" }
      ],
      "RialtoSwap": [
        { "name": "recipient", "type": "address" },
        { "name": "buyToken", "type": "address" },
        { "name": "minBuyAmount", "type": "uint256" },
        { "name": "deadline", "type": "uint64" },
        { "name": "feeRecipient", "type": "address" },
        { "name": "srcBps", "type": "uint16" },
        { "name": "dstBps", "type": "uint16" },
        { "name": "referralCode", "type": "bytes32" },
        { "name": "quoteId", "type": "bytes32" },
        { "name": "actionsHash", "type": "bytes32" }
      ]
    },
    "primaryType": "PermitWitnessTransferFrom",
    "message": {
      "permitted": {
        "token": "<sell_token_address>",
        "amount": "10000000000000000"
      },
      "spender": "<rialto_router_address_from_quote>",
      "nonce": "102222442441090043455808165797985926558",
      "deadline": "1780300214",
      "witness": {
        "recipient": "<taker_wallet_address>",
        "buyToken": "<buy_token_address>",
        "minBuyAmount": "19701021",
        "deadline": 1780300214,
        "feeRecipient": "<zero_address>",
        "srcBps": 0,
        "dstBps": 50,
        "referralCode": "<zero_bytes32>",
        "quoteId": "<quote_id_bytes32>",
        "actionsHash": "<actions_hash_bytes32>"
      }
    },
    "nonce": "102222442441090043455808165797985926558",
    "deadline": 1780300214
  },
  "issues": {
    "allowance": {
      "actual": "0",
      "spender": "<permit2_contract_address>"
    },
    "balance": null,
    "simulationIncomplete": false,
    "invalidSourcesPassed": []
  },
  "chain_id": chain_id,
  "sell_token": "<sell_token_address>",
  "buy_token": "<buy_token_address>",
  "sell_amount": "10000000000000000",
  "buy_amount": "19800022",
  "min_buy_amount": "19701021",
  "platform_fee": {
    "total_bps": 50,
    "fees": [
      {
        "side": "destination",
        "token": "<buy_token_address>",
        "symbol": "USDG",
        "decimals": 6,
        "bps": "50",
        "bps_x100": 5000,
        "amount": "9900",
        "amount_decimal": "0.0099",
        "recipient": "<rialto_fee_recipient_address>"
      }
    ]
  },
  "network_fee": {
    "token": "<native_eth_sentinel>",
    "symbol": "ETH",
    "decimals": 18,
    "gas": "226046",
    "gas_price": "20000000",
    "gas_price_gwei": "0.02",
    "amount": "4520920000000",
    "amount_gwei": "4520.92",
    "amount_eth": "0.00000452092"
  },
  "taker": "<taker_wallet_address>",
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
        "pool_id": "uniswap-v3:chain-chain_id:<pool_address>:100",
        "sell_token": "<sell_token_address>",
        "buy_token": "<buy_token_address>",
        "sell_amount": "10000000000000000",
        "buy_amount": "19809926"
      }
    ]
  }
}
```

The exact amounts and route legs depend on live liquidity.

## Executing a Quote

This section is for wallets, frontends, and other routers that want to integrate
Rialto as an execution venue. The integration boundary is intentionally small:
call `/quote`, then submit the transaction returned in that same quote. You
never build route calldata, pool calls, fee logic, slippage math, Permit2
witnesses, or settlement actions yourself — Rialto does all of it.

### The executable transaction

The payload to send on-chain lives in the quote's `tx` object:

| Field | Description |
| --- | --- |
| `tx.to` | RialtoRouter contract address — the `to` of your transaction. |
| `tx.data` | ABI-encoded calldata for the swap. |
| `tx.value` | Native value to send (`0` for ERC20 sells; the sell amount for native ETH sells). |
| `tx.estimated_gas` | Backend gas estimate. You should still call wallet/RPC gas estimation before sending. |
| `tx.signature_offset` | Byte offset in `tx.data` where the taker's 65-byte Permit2 signature is inserted. Present only for Permit2 settlement. |
| `permit2` | EIP-712 typed-data message the taker signs. Present only for Permit2 settlement. |

The `settlement` field tells you which mode applies. Choose your handling from
`settlement`, not from assumptions — the backend decides the mode.

Do not modify `quote_id`, `route`, `platform_fee`, `permit2.message.witness`, or
`tx.data` except for replacing the Permit2 signature placeholder described
below. A changed route or witness will not match the signed quote and may revert.
Use the returned `permit2` payload as the source of truth; do not recompute the
bytes32 witness `quoteId` from the UUID yourself.

### Permit2 settlement (default for ERC20 sells)

Permit2 lets a user authorize a single, exact token transfer by **signing a
message** instead of sending a separate approval transaction. Rialto uses Permit2
with a **witness**: the signed EIP-712 message is bound to the precise swap —
recipient, buy token, minimum output, deadline, quote id, and the hash of the
route actions.

A Permit2 response looks like:

```json
{
  "settlement": "permit2",
  "tx": {
    "to": "<rialto_router_address_from_quote>",
    "data": "0xdbefe2d6...",
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

1. If `issues.allowance` is non-null, ask the taker to approve
   `issues.allowance.spender` for at least `sell_amount`. For Permit2 quotes this
   spender is the Permit2 contract.
2. Have the taker wallet sign the `permit2` object as EIP-712 typed data.
3. Splice the returned 65-byte signature into `tx.data` at `tx.signature_offset`.
4. Send the transaction from the taker wallet (`to`, patched `data`, `value`).

Most wallet libraries accept the returned `domain`, `types`, `primaryType`, and
`message` directly. If your library rejects `EIP712Domain` inside `types`, remove
only that key before calling `signTypedData`; do not change the message or
witness fields.

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

`/quote` returns `settlement: "allowance"` (no `permit2` object, no
`signature_offset`) when Permit2 is not the right path — for example native ETH
sells, or a flow where the taker already wants to use ERC20 allowance.

```json
{
  "settlement": "allowance",
  "tx": {
    "to": "<rialto_router_address_from_quote>",
    "data": "0x2f1b6c8a...",
    "value": "0",
    "estimated_gas": 226046
  }
}
```

- **ERC20 sells:** if `issues.allowance` is non-null, the taker approves
  `issues.allowance.spender` for at least the raw `sell_amount`, then sends the
  transaction. Do not modify `tx.data`.
- **Native ETH sells:** send the transaction with `tx.value`; no ERC20 approval
  and no signature are needed.

### Simulation

Every route Rialto returns is validated against live chain state before it is
quoted, so the transaction in `/quote` is **pre-validated** — its expected output
has been checked, not merely encoded.

You can also simulate the returned transaction yourself before submitting it. A
successful simulation confirms the route still fills and the output meets
`min_buy_amount`; a failure means the quote has gone stale — request a fresh
quote.

### Submission

For the taker-submitted flow, submit the transaction from the **taker wallet**.
The Rialto API does not sign or broadcast this direct-flow transaction. Quotes
reflect live liquidity and can go stale quickly, so request a fresh quote if the
user waits.

## Gasless Relayer

The gasless flow lets the taker sign Permit2 while Rialto submits the on-chain
swap transaction. The user still authorizes the exact swap with a Permit2
witness signature, but the relayer wallet pays the network gas.

### Gasless constraints

| Constraint | Behavior |
| --- | --- |
| Settlement | Gasless requires `settlement: "permit2"` and a returned `permit2` payload. |
| Token type | ERC20 sells only. Native ETH sells require the taker-submitted flow. |
| Permit owner | Pass `permit2_owner=<taker>` on `/quote`. For gasless, `permit2_owner` must equal `taker`. |
| Recipient | The quote response controls the recipient through the Permit2 witness. Do not edit it. |
| Allowance | The taker must still approve Permit2 if `issues.allowance` is non-null. |
| Balance | Do not submit if `issues.balance` is non-null. |

### Step 1: request a gasless quote

Add `permit2_owner` to the standard quote request:

```bash
API_KEY='rialto_live_example.redacted_secret'
TAKER='<taker_wallet_address>'

curl -sS "https://rialto-trade-api.rialto.xyz/quote?sell_token=USDG&buy_token=WEEK&sell_amount=1&taker=$TAKER&permit2_owner=$TAKER&slippage_bps=50&chain_id=0000" \
  -H "Authorization: Bearer $API_KEY"
```

The response includes:

- `quote_id`: the UUID you will send to `/gasless/submit`.
- `permit2`: the EIP-712 typed data the taker signs.
- `issues`: balance and allowance blockers.
- `tx`: the transaction Rialto will submit after receiving the signature.

### Step 2: user approval and Permit2 signature

If `issues.allowance` is non-null, the taker must approve
`issues.allowance.spender` for at least the raw `sell_amount`. For gasless
Permit2 quotes, the spender is the Permit2 contract.

Then have the taker sign the `permit2` typed data exactly as returned.

### Step 3: submit to the relayer

`POST /gasless/submit`

Auth: requires API key with swap access.

Request body:

| Field | Description |
| --- | --- |
| `quote_id` | UUID returned by `/quote`. The quote must have been requested with `permit2_owner`. |
| `signature` | `0x`-prefixed 65-byte Permit2 signature over the typed data returned by `/quote`. |
| `idempotency_key` | Optional client-generated unique string for one relay attempt, such as your internal order id or a UUID. Reuse the same value only when retrying the same quote submission; do not reuse it across different quotes or users. |

Example:

```bash
curl -sS 'https://rialto-trade-api.rialto.xyz/gasless/submit' \
  -H "Authorization: Bearer $API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "quote_id": "b7b0a3d8-9f6a-4f4a-92d2-1b0e4b5d0c6a",
    "signature": "0xabc123...",
    "idempotency_key": "partner-order-123"
  }'
```

Example response:

```json
{
  "relay_id": "f3f6a35f-93b9-4629-9a2e-02d2b80648ef",
  "quote_id": "b7b0a3d8-9f6a-4f4a-92d2-1b0e4b5d0c6a",
  "status": "submitted",
  "chain_id": chain_id,
  "tx_hash": "0x6b8f...",
  "error": null
}
```

### Step 4: poll relay status

`GET /gasless/status/{relay_id}`

Auth: requires API key with swap access.

Statuses:

| Status | Meaning |
| --- | --- |
| `accepted` | Request accepted and queued. |
| `submitted` | Relayer broadcasted the transaction. |
| `confirmed` | Transaction mined successfully. |
| `failed` | Transaction was submitted but failed or could not be confirmed. |
| `expired` | Request expired before successful relay. |
| `rejected` | Request failed validation before relay. |

Example:

```bash
curl -sS 'https://rialto-trade-api.rialto.xyz/gasless/status/f3f6a35f-93b9-4629-9a2e-02d2b80648ef' \
  -H "Authorization: Bearer $API_KEY"
```

### Python gasless example

This example requests a gasless quote, signs Permit2, submits the signature to
the relayer, and polls until a terminal status. Keep all keys in environment
variables; the values below are placeholders.

```bash
python3 -m pip install requests web3 eth-account

export API_KEY='rialto_live_example.redacted_secret'
export RPC_URL='<your-rh-rpc-url>'
export PRIVATE_KEY='0xREDACTED_TAKER_PRIVATE_KEY'
python3 rialto_gasless_swap.py
```

```python
import os
import time
from urllib.parse import urlencode

import requests
from eth_account import Account
from eth_account.messages import encode_typed_data
from web3 import Web3

API_BASE = "https://rialto-trade-api.rialto.xyz"
CHAIN_ID = chain_id
SELL_TOKEN = "USDG"
BUY_TOKEN = "WEEK"
SELL_AMOUNT = "1"
SLIPPAGE_BPS = 50

ERC20_ABI = [
    {
        "name": "approve",
        "type": "function",
        "stateMutability": "nonpayable",
        "inputs": [
            {"name": "spender", "type": "address"},
            {"name": "value", "type": "uint256"},
        ],
        "outputs": [{"name": "", "type": "bool"}],
    }
]


def require_env(name):
    value = os.getenv(name)
    if not value:
        raise RuntimeError(f"missing {name}")
    return value


def tx_fees(w3):
    latest = w3.eth.get_block("latest")
    priority = w3.to_wei("0.01", "gwei")
    return {
        "maxPriorityFeePerGas": priority,
        "maxFeePerGas": int(latest["baseFeePerGas"]) * 2 + priority,
    }


api_key = require_env("API_KEY")
private_key = require_env("PRIVATE_KEY")
w3 = Web3(Web3.HTTPProvider(require_env("RPC_URL")))
taker = Account.from_key(private_key).address

params = {
    "sell_token": SELL_TOKEN,
    "buy_token": BUY_TOKEN,
    "sell_amount": SELL_AMOUNT,
    "taker": taker,
    "permit2_owner": taker,
    "slippage_bps": SLIPPAGE_BPS,
    "chain_id": CHAIN_ID,
}
quote_response = requests.get(
    f"{API_BASE}/quote?{urlencode(params)}",
    headers={"Authorization": f"Bearer {api_key}"},
    timeout=30,
)
quote_response.raise_for_status()
quote = quote_response.json()

if quote.get("settlement") != "permit2" or not quote.get("permit2"):
    raise RuntimeError("gasless requires a Permit2 quote")
if quote.get("issues", {}).get("balance"):
    raise RuntimeError(f"insufficient balance: {quote['issues']['balance']}")

allowance = quote.get("issues", {}).get("allowance")
if allowance:
    token = Web3.to_checksum_address(quote["sell_token"])
    spender = Web3.to_checksum_address(allowance["spender"])
    amount = int(quote["sell_amount"])
    approve_tx = w3.eth.contract(address=token, abi=ERC20_ABI).functions.approve(
        spender, amount
    ).build_transaction(
        {
            "from": taker,
            "chainId": CHAIN_ID,
            "nonce": w3.eth.get_transaction_count(taker),
            **tx_fees(w3),
        }
    )
    approve_tx["gas"] = int(w3.eth.estimate_gas(approve_tx) * 1.2)
    signed_approval = Account.sign_transaction(approve_tx, private_key)
    approval_hash = w3.eth.send_raw_transaction(signed_approval.raw_transaction)
    receipt = w3.eth.wait_for_transaction_receipt(approval_hash)
    if receipt.status != 1:
        raise RuntimeError(f"approval reverted: {approval_hash.hex()}")

typed_data = {
    "domain": quote["permit2"]["domain"],
    "types": quote["permit2"]["types"],
    "primaryType": quote["permit2"]["primaryType"],
    "message": quote["permit2"]["message"],
}
signed_permit = Account.sign_message(
    encode_typed_data(full_message=typed_data),
    private_key,
)
signature = "0x" + bytes(signed_permit.signature).hex()

submit_response = requests.post(
    f"{API_BASE}/gasless/submit",
    headers={"Authorization": f"Bearer {api_key}"},
    json={
        "quote_id": quote["quote_id"],
        "signature": signature,
        "idempotency_key": f"partner-{quote['quote_id']}",
    },
    timeout=60,
)
submit_response.raise_for_status()
relay = submit_response.json()
print("relay_id:", relay["relay_id"])
print("initial_status:", relay["status"])

terminal = {"confirmed", "failed", "expired", "rejected"}
while relay["status"] not in terminal:
    time.sleep(5)
    status_response = requests.get(
        f"{API_BASE}/gasless/status/{relay['relay_id']}",
        headers={"Authorization": f"Bearer {api_key}"},
        timeout=30,
    )
    status_response.raise_for_status()
    relay = status_response.json()
    print("status:", relay["status"], "tx_hash:", relay.get("tx_hash"))

if relay["status"] != "confirmed":
    raise RuntimeError(f"relay did not confirm: {relay}")
```

### Direct-flow end-to-end example

```bash
API_KEY='rialto_live_example.redacted_secret'

curl -sS 'https://rialto-trade-api.rialto.xyz/quote?sell_token=WETH&buy_token=USDG&sell_amount=0.01&taker=<taker_wallet_address>&slippage_bps=50' \
  -H "Authorization: Bearer $API_KEY" -o quote.json

# In your app:
# - If quote.issues.balance is non-null, show an insufficient-balance error.
# - If quote.issues.allowance is non-null, approve quote.issues.allowance.spender.
# - Permit2: sign quote.permit2 as EIP-712, then splice the signature into
#   quote.tx.data at quote.tx.signature_offset.
# - Allowance/native: use quote.tx.data as-is.
# - Optional: eth_estimateGas / eth_call quote.tx from the taker.
# - Direct flow: submit quote.tx from the taker wallet.
# - Gasless flow: use POST /gasless/submit instead of sending quote.tx yourself.
```

## Requesting an Integrator API Key

Integrator keys are created through a wallet-signed onboarding flow. The wallet
that applies becomes the owner wallet for the integrator profile and must also
be the fee-recipient wallet.

The flow is:

1. Build the application payload and its `payload_hash`.
2. Call `POST /integrators/nonce` with action
   `create_integrator_application`.
3. Sign the exact `message` returned by the nonce endpoint with the owner
   wallet.
4. Submit the signed application to `POST /integrators/applications`.
5. Wait for Rialto approval if the application returns `status: "pending"`.
6. After approval, request another nonce for `create_integrator_api_key`, sign
   it, and call `POST /integrators/api-keys`.
7. Store the returned `api_key` immediately. It is shown once. Later profile
   reads only return a masked key.

### Payload hash format

Mutable integrator actions require a `payload_hash`.

The hash is:

```text
keccak256("field_name=<byte length>:<value>\n" for each field, in order)
```

Use UTF-8 byte length for each value. Optional fields are encoded as:

```text
none
some:<value>
```

### Endpoint: create integrator nonce

`POST /integrators/nonce`

Auth: public.

Request body:

| Field | Description |
| --- | --- |
| `chain_id` | Optional chain id. Use `chain_id` for RH. |
| `wallet` | Owner wallet that will sign. |
| `action` | One of `create_integrator_application`, `create_integrator_api_key`, `revoke_integrator_api_key`, `view_integrator_profile`. |
| `payload_hash` | Required for create/revoke actions. Not required for `view_integrator_profile`. |

Response includes `message`, `nonce`, `issued_at`, and `expiration_time`. Sign
the exact `message` string. Do not reconstruct it client-side.

### Endpoint: submit application

`POST /integrators/applications`

Auth: public.

Request body:

| Field | Description |
| --- | --- |
| `chain_id` | Optional chain id. |
| `owner_wallet` | Wallet that owns the integrator profile. |
| `display_name` | Human-readable partner/app name. |
| `slug` | Lowercase unique id, 3-64 chars, lowercase letters/digits/hyphens only. |
| `contact_email` | Optional. |
| `telegram_handle` | Optional. |
| `app_url` | Optional. |
| `fee_recipient` | Wallet that receives integrator fees. Must equal `owner_wallet`. |
| `requested_max_fee_bps` | Max fee cap requested for this key, in basis points. |
| `payload_hash` | Hash of the application fields. |
| `nonce`, `issued_at`, `expiration_time`, `signature` | Fields from `/integrators/nonce` plus the wallet signature. |

Response:

```json
{
  "integrator_id": 12,
  "slug": "example-wallet",
  "status": "pending",
  "message": "Application submitted for review."
}
```

If `status` is `pending`, the Rialto team will review your application as soon
as possible. If approved, the profile status becomes `active` and you can create
an API key for it. If the application response already returns `active`, you can
create a key immediately.

### Endpoint: create API key

`POST /integrators/api-keys`

Auth: public.

Request body:

| Field | Description |
| --- | --- |
| `chain_id` | Optional chain id. |
| `owner_wallet` | Integrator owner wallet. |
| `integrator_id` | Numeric id returned by the application endpoint. |
| `label` | Human-readable key label. |
| `payload_hash` | Hash of `action`, `chain_id`, `owner_wallet`, `integrator_id`, and `label`. |
| `nonce`, `issued_at`, `expiration_time`, `signature` | Fields from `/integrators/nonce` plus the wallet signature. |

Response:

```json
{
  "key_id": 34,
  "api_key": "rialto_live_example.redacted_secret",
  "masked_key": "rialto_live_example...cret",
  "prefix": "example",
  "scopes": ["quote:read", "swap:create", "swap:integrator"],
  "quote_rate_limit_per_minute": 60,
  "swap_rate_limit_per_minute": 10,
  "integrator_id": "example-wallet",
  "integrator_fee_recipient": "<taker_wallet_address>",
  "integrator_max_fee_bps": 50,
  "shown_once": true
}
```

**Note:** `api_key` is shown only once. Store it securely when this response is
returned. It cannot be retrieved again; `/integrators/me` only returns masked
key metadata.

### Endpoint: list profile and masked keys

`POST /integrators/me`

Auth: public.

Use action `view_integrator_profile` on `/integrators/nonce`, sign the returned
message, then submit `owner_wallet`, nonce fields, and signature. The response
lists all profiles for that owner and masked API keys.

### Endpoint: revoke API key

`POST /integrators/api-keys/{key_id}/revoke`

Auth: public.

Use action `revoke_integrator_api_key`. Compute the `payload_hash` from
`owner_wallet`, `integrator_id`, and `key_id`. Revoked keys are rejected by
trading endpoints.

### Python key-request example

This example submits an application. If the returned status is `active`, it also
creates a key. If the status is `pending`, save the returned `integrator_id` and
rerun the key-creation part after Rialto approves it.

```bash
python3 -m pip install requests eth-account eth-utils

export OWNER_PRIVATE_KEY='0xREDACTED_OWNER_PRIVATE_KEY'
python3 rialto_request_integrator_key.py
```

```python
import os
import time

import requests
from eth_account import Account
from eth_account.messages import encode_defunct
from eth_utils import keccak

API_BASE = "https://rialto-trade-api.rialto.xyz"
CHAIN_ID = chain_id
ACTION_CREATE_APPLICATION = "create_integrator_application"
ACTION_CREATE_API_KEY = "create_integrator_api_key"


def require_env(name):
    value = os.getenv(name)
    if not value:
        raise RuntimeError(f"missing {name}")
    return value


def optional(value):
    return f"some:{value}" if value is not None else "none"


def payload_hash(fields):
    canonical = ""
    for key, value in fields:
        value = str(value)
        canonical += f"{key}={len(value.encode('utf-8'))}:{value}\n"
    return "0x" + keccak(canonical.encode("utf-8")).hex()


def sign_message(private_key, message):
    signed = Account.sign_message(encode_defunct(text=message), private_key)
    return "0x" + bytes(signed.signature).hex()


def nonce(owner, action, hash_value=None):
    body = {"chain_id": CHAIN_ID, "wallet": owner, "action": action}
    if hash_value is not None:
        body["payload_hash"] = hash_value
    response = requests.post(f"{API_BASE}/integrators/nonce", json=body, timeout=30)
    response.raise_for_status()
    return response.json()


private_key = require_env("OWNER_PRIVATE_KEY")
owner = Account.from_key(private_key).address.lower()
display_name = "Example Partner"
slug = f"example-partner-{int(time.time())}"
contact_email = "ops@example.com"
telegram_handle = None
app_url = "https://example.com"
fee_recipient = owner
requested_max_fee_bps = 50

app_hash = payload_hash(
    [
        ("action", ACTION_CREATE_APPLICATION),
        ("chain_id", CHAIN_ID),
        ("owner_wallet", owner),
        ("display_name", display_name),
        ("slug", slug),
        ("contact_email", optional(contact_email)),
        ("telegram_handle", optional(telegram_handle)),
        ("app_url", optional(app_url)),
        ("fee_recipient", fee_recipient),
        ("requested_max_fee_bps", requested_max_fee_bps),
    ]
)
app_nonce = nonce(owner, ACTION_CREATE_APPLICATION, app_hash)
app_body = {
    "chain_id": CHAIN_ID,
    "owner_wallet": owner,
    "display_name": display_name,
    "slug": slug,
    "contact_email": contact_email,
    "telegram_handle": telegram_handle,
    "app_url": app_url,
    "fee_recipient": fee_recipient,
    "requested_max_fee_bps": requested_max_fee_bps,
    "payload_hash": app_hash,
    "nonce": app_nonce["nonce"],
    "issued_at": app_nonce["issued_at"],
    "expiration_time": app_nonce["expiration_time"],
    "signature": sign_message(private_key, app_nonce["message"]),
}
app_response = requests.post(
    f"{API_BASE}/integrators/applications",
    json=app_body,
    timeout=30,
)
app_response.raise_for_status()
application = app_response.json()
print("application:", application)

if application["status"] != "active":
    print("Application is pending approval. Create the API key after approval.")
    raise SystemExit(0)

integrator_id = application["integrator_id"]
label = "production-key"
key_hash = payload_hash(
    [
        ("action", ACTION_CREATE_API_KEY),
        ("chain_id", CHAIN_ID),
        ("owner_wallet", owner),
        ("integrator_id", integrator_id),
        ("label", label),
    ]
)
key_nonce = nonce(owner, ACTION_CREATE_API_KEY, key_hash)
key_body = {
    "chain_id": CHAIN_ID,
    "owner_wallet": owner,
    "integrator_id": integrator_id,
    "label": label,
    "payload_hash": key_hash,
    "nonce": key_nonce["nonce"],
    "issued_at": key_nonce["issued_at"],
    "expiration_time": key_nonce["expiration_time"],
    "signature": sign_message(private_key, key_nonce["message"]),
}
key_response = requests.post(
    f"{API_BASE}/integrators/api-keys",
    json=key_body,
    timeout=30,
)
key_response.raise_for_status()
created_key = key_response.json()
print("raw API key; store now:", created_key["api_key"])
print("masked key:", created_key["masked_key"])
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
3. The same quote includes one executable transaction that, on execution, pays
   both Rialto's treasury and **your wallet** atomically.
4. The taker signs/approves as needed, then either submits the quote's `tx`
   directly or signs Permit2 for gasless relay. Your fee lands in your wallet
   the moment the swap settles.

Your fee is **additive** — Rialto charges its standard fee, and your fee is taken
on top of it. Rialto does not take any cut of your portion.

### Getting set up

Integrator access is configured through the self-service flow in
[Requesting an Integrator API Key](#requesting-an-integrator-api-key), followed
by Rialto approval when required. The approved API key binds three things:

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
INTEGRATOR_KEY='rialto_live_integrator.redacted_secret'

curl -sS 'https://rialto-trade-api.rialto.xyz/quote?sell_token=WETH&buy_token=USDG&sell_amount=0.01&taker=<taker_wallet_address>&slippage_bps=50&swap_fee_bps=30' \
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
        "token": "<sell_token_address>",
        "symbol": "WETH",
        "decimals": 18,
        "bps": "50",
        "amount": "50000000000000",
        "amount_decimal": "0.00005",
        "recipient": "<rialto_fee_recipient_address>"
      },
      {
        "side": "source",
        "token": "<sell_token_address>",
        "symbol": "WETH",
        "decimals": 18,
        "bps": "30",
        "amount": "30000000000000",
        "amount_decimal": "0.00003",
        "recipient": "<integrator_fee_recipient_address>"
      }
    ]
  },
  "integrator_fee": {
    "bps": 30,
    "recipient": "<integrator_fee_recipient_address>",
    "id": "your-integrator-id"
  }
}
```

In this example Rialto's fee is 50 bps and your fee is 30 bps, for a combined
`total_bps` of 80. The `recipient` on the second line is **your** payout wallet.
The `token` and `amount` fields tell you exactly which token the fee is taken in
and how much — Rialto selects the most suitable token in the swap automatically,
so you do not need to choose a fee token.

### Executing an integrator-fee quote

Execute the quote exactly like any other quote: handle `issues`, sign Permit2 if
present, patch `tx.data` if needed, and submit the returned `tx` from the taker
wallet. You do not build any fee logic yourself. The fee recipient and cap are
key-bound server-side, and the fee split is encoded into the transaction Rialto
returns.

### Python example

This example requests an integrator quote with `swapFeeBps=50`, signs the
returned Permit2 payload if present, patches the signature into `tx.data`, and
sends the returned transaction. Keep the API key and private key in environment
variables; the values below are placeholders.

```bash
python3 -m pip install requests web3 eth-account

export INTEGRATOR_API_KEY='rialto_live_integrator.redacted_secret'
export RPC_URL='<your-rh-rpc-url>'
export PRIVATE_KEY='0xREDACTED_PRIVATE_KEY'
python3 rialto_integrator_quote.py
```

```python
import os
from urllib.parse import urlencode

import requests
from eth_account import Account
from eth_account.messages import encode_typed_data
from web3 import Web3

API_BASE = "https://rialto-trade-api.rialto.xyz"
CHAIN_ID = chain_id
SELL_TOKEN = "USDG"
BUY_TOKEN = "WEEK"
SELL_AMOUNT = "0.532262"
SLIPPAGE_BPS = 50
SWAP_FEE_BPS = 50

ERC20_ABI = [
    {
        "name": "approve",
        "type": "function",
        "stateMutability": "nonpayable",
        "inputs": [
            {"name": "spender", "type": "address"},
            {"name": "value", "type": "uint256"},
        ],
        "outputs": [{"name": "", "type": "bool"}],
    }
]


def require_env(name: str) -> str:
    value = os.getenv(name)
    if not value:
        raise RuntimeError(f"missing {name}")
    return value


def tx_fees(w3: Web3) -> dict:
    latest = w3.eth.get_block("latest")
    priority = w3.to_wei("0.01", "gwei")
    return {
        "maxPriorityFeePerGas": priority,
        "maxFeePerGas": int(latest["baseFeePerGas"]) * 2 + priority,
    }


def patch_permit2_signature(tx_data: str, signature_offset: int, signature: bytes) -> str:
    data = bytearray(bytes.fromhex(tx_data.removeprefix("0x")))
    if len(signature) != 65:
        raise RuntimeError("Permit2 signature must be 65 bytes")
    data[signature_offset : signature_offset + 65] = signature
    return "0x" + data.hex()


api_key = require_env("INTEGRATOR_API_KEY")
private_key = require_env("PRIVATE_KEY")
w3 = Web3(Web3.HTTPProvider(require_env("RPC_URL")))
taker = Account.from_key(private_key).address

params = {
    "sell_token": SELL_TOKEN,
    "buy_token": BUY_TOKEN,
    "sell_amount": SELL_AMOUNT,
    "taker": taker,
    "slippage_bps": SLIPPAGE_BPS,
    "chain_id": CHAIN_ID,
    "swapFeeBps": SWAP_FEE_BPS,
}

response = requests.get(
    f"{API_BASE}/quote?{urlencode(params)}",
    headers={"Authorization": f"Bearer {api_key}"},
    timeout=30,
)
response.raise_for_status()
quote = response.json()

print("quote_id:", quote["quote_id"])
print("buy_amount:", quote["buy_amount"])
print("min_buy_amount:", quote["min_buy_amount"])
print("integrator_fee:", quote.get("integrator_fee"))

issues = quote.get("issues") or {}
if issues.get("balance"):
    raise RuntimeError(f"insufficient balance: {issues['balance']}")

if issues.get("allowance"):
    allowance = issues["allowance"]
    token = Web3.to_checksum_address(quote["sell_token"])
    spender = Web3.to_checksum_address(allowance["spender"])
    amount = int(quote["sell_amount"])
    approve_tx = w3.eth.contract(address=token, abi=ERC20_ABI).functions.approve(
        spender, amount
    ).build_transaction(
        {
            "from": taker,
            "chainId": CHAIN_ID,
            "nonce": w3.eth.get_transaction_count(taker),
            **tx_fees(w3),
        }
    )
    approve_tx["gas"] = int(w3.eth.estimate_gas(approve_tx) * 1.2)
    signed_approval = Account.sign_transaction(approve_tx, private_key)
    approval_hash = w3.eth.send_raw_transaction(signed_approval.raw_transaction)
    receipt = w3.eth.wait_for_transaction_receipt(approval_hash)
    if receipt.status != 1:
        raise RuntimeError(f"approval reverted: {approval_hash.hex()}")

tx = quote["tx"]
data = tx["data"]
if quote.get("permit2"):
    typed_data = {
        "domain": quote["permit2"]["domain"],
        "types": quote["permit2"]["types"],
        "primaryType": quote["permit2"]["primaryType"],
        "message": quote["permit2"]["message"],
    }
    signed_permit = Account.sign_message(
        encode_typed_data(full_message=typed_data),
        private_key,
    )
    data = patch_permit2_signature(
        tx["data"],
        int(tx["signature_offset"]),
        bytes(signed_permit.signature),
    )

swap_tx = {
    "from": taker,
    "to": Web3.to_checksum_address(tx["to"]),
    "data": data,
    "value": int(tx.get("value", "0")),
    "chainId": CHAIN_ID,
    "nonce": w3.eth.get_transaction_count(taker),
    **tx_fees(w3),
}
gas_estimate = w3.eth.estimate_gas(swap_tx)
swap_tx["gas"] = int(gas_estimate * 1.3)

signed_swap = Account.sign_transaction(swap_tx, private_key)
swap_hash = w3.eth.send_raw_transaction(signed_swap.raw_transaction)
print("swap_tx:", swap_hash.hex())
```

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
- Execute the transaction returned by `/quote` — your fee is paid to your wallet
  atomically, on every swap.

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

- Integrator API keys are created through the wallet-signed onboarding flow and
  may require Rialto approval before activation.
- API keys may be scoped to quote-only, execution, or integrator access.
- API keys may have rate limits or expiry.
- Quotes depend on live liquidity and may become stale quickly.
- Partners should request a fresh quote if the user waits.
- In direct flow, submit transactions from the taker wallet.
- In gasless flow, the taker signs Permit2 and Rialto relays the transaction.
