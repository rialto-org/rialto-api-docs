# Prop AMM Integration Guide

How to onboard an on-chain proprietary AMM ( "propAMMs") as a liquidity source on
the Rialto Router. This is for **on-chain market makers** that expose an on-chain
price/quote function and settle atomically in the same transaction
---

## 1. How Rialto sources liquidity

Rialto's router quotes every candidate pool **on-chain** at request time and picks
the best route net of gas, then settles the chosen route atomically through the
`RialtoRouter` settlement contract. A prop AMM plugs in as one more candidate pool:

- **Quote time (read-only):** Rialto calls your `getAmountOut(...)` **view**
  function. Your quote competes on output net of gas.
- **Settle time (one tx):** if your pool wins a leg, `RialtoRouter` calls your
  `swapExactIn(...)` via a generic `RawCall` action, with on-chain slippage and
  output checks.

The model is an **on-chain, batch-sampleable quote** plus an **atomic settlement
entrypoint** — everything stays on-chain and trustless, with no signing service or
quote API to run (unlike off-chain RFQ).

---

## 2. What you implement

Deploy one contract per token **pair** implementing the **canonical Rialto
prop-AMM interface** below — really just two functions: a `getAmountOut` quote and
a `swapExactIn` settle. Rialto calls them by their 4-byte selector, so match these
signatures — or, if your AMM already exposes different ones, put a thin adapter in
front (see §2.1). `token0()`/`token1()` are **optional**; Rialto never calls them
(the pair and ordering come from our config).

```solidity
interface IPropPair {
    /// OPTIONAL — Rialto never calls these; pair & ordering come from our config.
    /// Included for convention only.
    function token0() external view returns (address);
    function token1() external view returns (address);

    /// QUOTE (read-only view). Exact-in only.
    /// zeroForOne = true  -> selling token0, receiving token1
    /// zeroForOne = false -> selling token1, receiving token0
    /// Returns the token-out amount you would deliver for `amountIn` right now.
    function getAmountOut(bool zeroForOne, uint256 amountIn)
        external view returns (uint256 amountOut);

    /// SETTLE (state-changing). Called by RialtoRouter only.
    /// Pull `amountIn` of the input token, send the output to `to`, revert if
    /// you can't deliver at least `amountOutMin`.
    function swapExactIn(
        bool zeroForOne,
        uint256 amountIn,
        uint256 amountOutMin,
        address to,
        uint256 deadline
    ) external payable returns (uint256 amountOut);
}
```

`getAmountOut` is sampled at quote time; `swapExactIn` settles. The trade direction
(`zeroForOne`) is fixed by our **config** ordering, not by `token0()`/`token1()`:
`zeroForOne = true` ⇒ sell config-token0 → buy config-token1.

### 2.1 If your AMM already uses different signatures

This interface is a convention we standardize on, not a protocol constraint — we
set the ABI on our side. But don't just rename your functions and assume it works,
because two things are coupled to the exact shape:

- The settle **selector is allowlisted on-chain** per pool, and the router patches
  the real input amount into the calldata at a **fixed offset** that assumes
  `swapExactIn(bool, uint256 amountIn, …)` (with `amountIn` as the 2nd argument). A
  different argument layout moves that offset.
- The quote/settle selectors are currently a **single shared interface** across all
  prop-AMM pools.

So if your AMM already exposes different names or argument orders, the clean path is
a **thin adapter contract** that exposes exactly `getAmountOut` / `swapExactIn` and
forwards to your AMM (≈30 lines — see the Appendix skeleton). If adapters are a
dealbreaker across several pairs, talk to us — we can make the selectors and the
input-amount offset configurable per pool on our side instead.

---

## 3. Quoting — `getAmountOut`

| Param | Type | Meaning |
| --- | --- | --- |
| `zeroForOne` | `bool` | `true` = sell `token0()`, buy `token1()`; `false` = the reverse |
| `amountIn` | `uint256` | exact input amount, in the input token's smallest units (wei) |
| **returns** | `uint256` | output amount you'd deliver **now**, smallest units of the output token |

Requirements:

- **`view` / read-only.** `getAmountOut` must not write state.
- **Exact-in only.** There is no exact-out path today.
- **Unsupported sizes.** For an `amountIn` or pair you can't fill, return `0` or
  revert — don't return a price you won't honor.
- **Self-consistent with settlement.** The price `getAmountOut` returns is what
  Rialto routes on. Occasional slips are expected from price volatility in the
  block-or-two between quote and settle, but we generally expect your quotes and
  swaps to agree — don't quote more favorably than you intend to fill. Quote your
  *currently executable* price; if it's oracle-driven, make sure the oracle you
  quote from is the one you settle against.
- **Gas.** Rialto assigns your leg a gas estimate when ranking routes net-of-gas,
  calibrated from observed on-chain usage (target ≈ p90 + ~10% headroom). Give us a
  realistic gas figure for your `swapExactIn` so we set it correctly — an
  unrealistically cheap estimate makes your route look better than it settles.

---

## 4. Settlement — `swapExactIn`

`RialtoRouter` settles a winning leg by calling `swapExactIn` through a generic
`RawCall` action. The exact runtime behavior:

1. **Input delivery.** For ERC-20 input, the router `approve`s your contract for
   exactly `amountIn` immediately before the call, so you pull it with
   `transferFrom(router, you, amountIn)`. The router **revokes the approval to 0**
   right after. For native-ETH input, the router calls `swapExactIn{value: amountIn}`
   (so the function is `payable`).
2. **Honor the `amountIn` parameter.** It's the authoritative input amount for this
   fill — pull and price against exactly the `amountIn` passed into the call; don't
   assume a fixed or pre-agreed size.
3. **Output delivery.** Send the output token to **`to`** (which is the
   `RialtoRouter` address). The router measures its **own balance delta** of the
   output token and requires `received >= amountOutMin`, reverting the whole swap
   otherwise. So you must actually transfer the output to `to` within the call.
4. **`deadline`.** Unix seconds. Enforcing it on your side is optional — Rialto
   already bounds the swap's validity window — so honor it or ignore it as you
   prefer; treat `0` as "no deadline".
5. **Refunds.** If you pull less than `amountIn`, the router refunds the unspent
   input — but the cleanest contract pulls and uses exactly `amountIn`.

| Param | Type | Meaning |
| --- | --- | --- |
| `zeroForOne` | `bool` | same convention as the quote |
| `amountIn` | `uint256` | exact input to pull (router-supplied, authoritative) |
| `amountOutMin` | `uint256` | revert if you can't deliver at least this |
| `to` | `address` | recipient of the output — **always the RialtoRouter** |
| `deadline` | `uint256` | Unix-seconds expiry |
| **returns** | `uint256` | actual output delivered |

**Approval/transfer pattern (ERC-20):**

```solidity
// Router, before the call:    IERC20(tokenIn).approve(you, amountIn);
// You, inside swapExactIn:     IERC20(tokenIn).transferFrom(msg.sender, address(this), amountIn);
//                              ... your pricing/inventory logic ...
//                              IERC20(tokenOut).transfer(to, amountOut);   // to == router
// Router, after the call:      IERC20(tokenIn).approve(you, 0);
```

> Native ETH is optional/advanced — most prop pairs are ERC-20/ERC-20 (e.g.
> WETH/USDC). If you want a native-ETH side, accept `msg.value` on input and send
> ETH to `to` on output. Two recommendations:
> - Pick **one convention and use it across all your pairs** — either `ETH/token`
>   everywhere or `WETH/token` everywhere; don't mix native-ETH and WETH pairs.
> - Represent native ETH with the **`0xEeee…EEeE` sentinel** (preferred).
>
> Tell us which and we'll confirm the wiring.

---

## 5. Onboarding checklist — what we need from you

To list a pair, send us:

- [ ] **Pair contract address** (deployed, verified) implementing `IPropPair`.
- [ ] **Chain** (today: Arbitrum One, `42161`).
- [ ] **`token0()` / `token1()`** addresses and symbols, in that order.
- [ ] Confirmation that **`swapExactIn` sends output to `to`** and reverts below
      `amountOutMin` / past `deadline`.
- [ ] A realistic **gas estimate** for your `swapExactIn` (we calibrate the per-leg
      estimate from observed gas, ≈ p90 + 10% headroom).
- [ ] Whether you want a **native-ETH** side (default: ERC-20/ERC-20 only).

We then add the config entry and the owner allowlists, and your liquidity starts
competing on the next router cycle. We'll quote a few sizes against your pool
and compare to your expected fills before going live.

---

## Appendix A — reference interface + skeleton

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract MyPropPair {
    address public immutable token0;
    address public immutable token1;

    constructor(address _token0, address _token1) {
        token0 = _token0;
        token1 = _token1;
    }

    function getAmountOut(bool zeroForOne, uint256 amountIn)
        external view returns (uint256 amountOut)
    {
        // Your pricing. Must be a pure view; return 0 for sizes you won't fill.
        amountOut = _price(zeroForOne, amountIn);
    }

    function swapExactIn(
        bool zeroForOne,
        uint256 amountIn,
        uint256 amountOutMin,
        address to,
        uint256 deadline
    ) external payable returns (uint256 amountOut) {
        if (deadline != 0) require(block.timestamp <= deadline, "expired");
        (address tin, address tout) = zeroForOne ? (token0, token1) : (token1, token0);

        IERC20(tin).transferFrom(msg.sender, address(this), amountIn);   // router approved us
        amountOut = _price(zeroForOne, amountIn);
        require(amountOut >= amountOutMin, "slippage");
        IERC20(tout).transfer(to, amountOut);                            // to == RialtoRouter
    }

    function _price(bool zeroForOne, uint256 amountIn) internal view returns (uint256) {
        // oracle / curve / inventory logic
    }
}
```

---

## Appendix B — Security model

What Rialto's settlement contract guarantees around your `swapExactIn` call:

- **Allowlisted, single-function.** The router can only `RawCall` the exact
  `(yourPair, swapExactIn.selector)` we allowlist — no other function, no arbitrary
  calldata. A pair address allowlisted as a call target cannot also be a routable
  token, and vice-versa.
- **Output-verified.** Settlement reverts unless the router's measured output-token
  balance increases by at least `amountOutMin`. You cannot be "paid" without
  delivering.
- **Approvals are scoped and revoked.** The router approves exactly `amountIn` and
  resets to 0 around each call, so a stale/oversized allowance can't be drained
  later.
- **You hold your own inventory and risk.** Rialto never custodies your funds; your
  contract prices and fills from its own balances. Your `getAmountOut` is your
  price; your `swapExactIn` is your fill.
