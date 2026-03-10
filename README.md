# Switch Operators SDK

> **Integration guide for operators (fillers) of Switch Limit Orders on PulseChain**

Operators monitor the Switch orderbook for signed limit orders and fill them on-chain for profit. This document covers contract interaction, profit strategies, fee modes, tax token handling, and the order lifecycle.

**API:** `https://quote.switch.win` &nbsp;|&nbsp; **Chain:** PulseChain (369)

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Contract Addresses](#contract-addresses)
4. [Order Lifecycle](#order-lifecycle)
5. [Fetching Active Orders](#fetching-active-orders)
6. [Evaluating an Order](#evaluating-an-order)
7. [Filling an Order](#filling-an-order)
8. [Profit Strategies & `excessOnInput`](#profit-strategies--excessoninput)
9. [Tax Token Handling](#tax-token-handling)
   - [First-Hop Adapter Restriction](#first-hop-adapter-restriction-for-sell-tax-inputs)
   - [Last-Hop Adapter Restriction](#last-hop-adapter-restriction-for-buy-tax-outputs)
   - [Route Amount Scaling](#route-amount-scaling-for-sell-tax-input-feeoutputtrue)
10. [On-Chain Structs](#on-chain-structs)
11. [Revert Errors](#revert-errors)
12. [Reference](#reference)
13. [Support](#support)

---

## Overview

Switch Limit Orders use **gasless EIP-712 signatures**. Users sign orders off-chain; tokens remain in the maker's wallet until filled.

As an operator, you:

1. **Monitor** the orderbook for active orders
2. **Evaluate** whether an order can be filled profitably
3. **Fill** on-chain — via DEX routing (`fillOrder`) or from your own liquidity (`directFillOrder`)

The contract enforces `minAmountOut` for the maker — everything above that is yours.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│  MAKER (user wallet)                                                             │
│  ├── Signs EIP-712 order off-chain (gasless)                                     │
│  ├── Approves SwitchLimitOrder or SwitchRouter (one-time per token + fee mode)   │
│  └── Tokens stay in wallet until fill                                            │
├──────────────────────────────────────────────────────────────────────────────────┤
│  SWITCH BACKEND  (quote.switch.win)                                                │
│  ├── Stores signed orders, validates signatures, indexes on-chain events         │
│  └── Provides REST API for querying orders                                       │
├──────────────────────────────────────────────────────────────────────────────────┤
│  OPERATOR (your bot)                                                             │
│  ├── Polls GET /limit-orders for active orders                                   │
│  ├── Evaluates profitability                                                     │
│  ├── Route Fill: fillOrder(order, signature, routes, excessOnInput)              │
│  └── Direct Fill: directFillOrder(order, signature, outputAmount, directToMaker) │
├──────────────────────────────────────────────────────────────────────────────────┤
│  ON-CHAIN CONTRACTS                                                              │
│  ├── SwitchLimitOrder — verifies signature, enforces minAmountOut, distributes   │
│  └── SwitchRouter     — executes multi-path DEX routing                          │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Contract Addresses

> **⚠️ Do not hardcode the SwitchRouter address.** The router contract may be redeployed from time to time. When executing swaps (via `fillOrder` or `directFillOrder`), always use the `tx.to` address returned by the `/bestPath` API response. This ensures your operator bot automatically picks up router upgrades without code changes.

| Contract | Address |
|---|---|
| **SwitchRouter** | `0x99999d19eC98F936934e029e63D1C0A127a15207` |
| **SwitchLimitOrder** | `0x79925587bE77C25b292C0ecA6FEdD3A3f07916F9` |
| **SwitchPLSFlow** | `0x79D1Ce697509D75D79c6cA8f9232ee6ca6Df379a` |

Chain: **PulseChain** (ID `369`) &nbsp;|&nbsp; Fee denominator: `10000` (basis points)

---

## Order Lifecycle

```
ACTIVE ──▶ FILLED       (operator calls fillOrder or directFillOrder)
ACTIVE ──▶ CANCELLED    (maker calls invalidateNonce on-chain)
ACTIVE ──▶ EXPIRED      (block.timestamp > deadline, if deadline > 0)
```

An order can only be filled **once** — the nonce is marked used atomically. If your tx reverts, the nonce stays unused.

---

## Fetching Active Orders

No API key required.

```
GET https://quote.switch.win/limit-orders?status=ACTIVE
```

### Query Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `status` | string | `"ACTIVE"` | `ACTIVE`, `FILLED`, `CANCELLED`, `EXPIRED` |
| `maker` | address | — | Filter by maker |
| `tokenIn` | address | — | Filter by input token |
| `tokenOut` | address | — | Filter by output token |
| `pair` | string | — | Pair key (`tokenIn:tokenOut`, lowercased) |
| `limit` | number | `100` | Page size (1–500) |
| `offset` | number | `0` | Page offset |

### Additional Endpoints

| Method | Path | Description |
|---|---|---|
| `GET` | `/limit-orders/pairs` | Active pairs with order counts |
| `GET` | `/limit-orders/stats` | Summary statistics |
| `GET` | `/limit-orders/:maker/:nonce` | Single order lookup |

### Response

```json
{
  "total": 42,
  "limit": 100,
  "offset": 0,
  "orders": [
    {
      "maker": "0x...",
      "tokenIn": "0xA1077a294dDE1B09bB078844df40758a5D0f9a27",
      "tokenOut": "0x95B303987A60C71504D99Aa1b13B4DA07b0790ab",
      "amountIn": "1000000000000000000000",
      "minAmountOut": "500000000000000000000000",
      "deadline": 1717257600,
      "nonce": 1717171717,
      "feeOnOutput": false,
      "recipient": "0x...",
      "unwrapOutput": false,
      "partnerAddress": "0x0000000000000000000000000000000000000000",
      "signature": "0x...",
      "pairKey": "0xa1077...:0x95b30...",
      "status": "ACTIVE",
      "tokenInSellTaxBps": 0,
      "tokenInBuyTaxBps": 0,
      "tokenOutSellTaxBps": 500,
      "tokenOutBuyTaxBps": 500,
      "createdAt": "2025-01-01T00:00:00.000Z",
      "updatedAt": "2025-01-01T00:00:00.000Z"
    }
  ]
}
```

> **Tax fields** are in basis points (100 = 1%). `0` = no tax detected. Populated automatically at order creation.

### Polling Strategy

- Poll every 10–30 seconds
- Use `/limit-orders/pairs` to focus on pairs with active orders
- After filling, re-evaluate same-pair orders (pool state changed)

---

## Evaluating an Order

### Pre-Flight Checks

1. **Not expired:** `deadline == 0 || deadline > block.timestamp`
2. **Nonce unused:** `isNonceUsed(maker, nonce)` returns `false`
3. **Maker has balance:** `ERC20(tokenIn).balanceOf(maker) >= amountIn`
4. **Maker has allowance:**
   - `feeOnOutput == false`: allowance to SwitchLimitOrder
   - `feeOnOutput == true`: allowance to SwitchRouter

Or use `canFillOrder(order, signature)` — performs all checks in one call (does not check DEX liquidity).

### PLSFlow Orders (Native PLS)

Orders where `maker == 0x79D1Ce697509D75D79c6cA8f9232ee6ca6Df379a` are PLSFlow orders (native PLS sold as WPLS). For these:

- **Skip** maker allowance checks (PLSFlow has infinite approval)
- Check `WPLS.balanceOf(PLSFlow)` instead of maker balance
- Fill identically to standard orders

### Profitability Check

1. Quote the pair through your own routing (Switch does **not** provide a routing API — this is where your competitive edge comes from)
2. Account for protocol fee (currently 30 bps, via `getFee()`)
3. Account for tax tokens if applicable (see [Tax Token Handling](#tax-token-handling))
4. **If input has sell tax**, first hop must use V2 adapters only (see [First-Hop Adapter Restriction](#first-hop-adapter-restriction-for-sell-tax-inputs))
5. **If output has buy tax**, last hop must use V2 adapters only (see [Last-Hop Adapter Restriction](#last-hop-adapter-restriction-for-buy-tax-outputs))
5. Ensure profit > gas cost

---

## Filling an Order

### Route Fill (`fillOrder`)

You provide DEX routing instructions; the SwitchRouter executes swaps.

```solidity
function fillOrder(
    LimitOrder calldata order,
    bytes calldata signature,
    RouteAllocation[] calldata routes,
    bool excessOnInput
) external;
```

Profit comes from routing efficiency — either unrouted input (`excessOnInput=true`) or output surplus (`excessOnInput=false`). See [Profit Strategies](#profit-strategies--excessoninput).

### Direct Fill (`directFillOrder`)

You provide output tokens yourself and receive the maker's input tokens. No routing needed — lowest gas, zero slippage, no DEX dependency.

```solidity
function directFillOrder(
    LimitOrder calldata order,
    bytes calldata signature,
    uint256 outputAmount,
    bool directToMaker
) external;
```

`outputAmount` must be ≥ `minAmountOut`. For `feeOnOutput=true` orders, include fee: `outputAmount = minAmountOut × 10000 / (10000 - fee)`. For tax output tokens with `directToMaker=false`, oversend to cover transfer tax. Works with both `feeOnOutput` modes transparently.

**`directToMaker`** — When `true` and the order has `feeOnOutput=false` and `unwrapOutput=false`, the output is transferred directly from the filler to the recipient in a **single transfer** instead of the usual filler → contract → recipient (2 transfers). This saves one buy tax hit for tax output tokens. Silently ignored when `feeOnOutput=true` (output needs fee deduction at the contract) or `unwrapOutput=true` (output needs unwrapping). For non-tax tokens, pass `false` — both paths behave identically.

Your profit = value of input received − value of output sent − gas.

### Approval Requirements

| Who | `feeOnOutput=false` | `feeOnOutput=true` |
|---|---|---|
| **Maker** approves | SwitchLimitOrder | SwitchRouter |
| **Operator** approves (direct fill only) | SwitchLimitOrder for tokenOut | SwitchLimitOrder for tokenOut |

### Always Simulate First

```ts
try {
  await contract.fillOrder.staticCall(order, signature, routes, excessOnInput);
} catch (err) {
  console.error("Simulation reverted:", err.reason);
  return; // See Revert Errors for diagnosis
}
const tx = await contract.fillOrder(order, signature, routes, excessOnInput);
```

### Example: Route Fill

```ts
import { ethers } from "ethers";
import SwitchLimitOrderABI from "./abi/SwitchLimitOrderABI.json";

const LIMIT_ORDER_ADDRESS = "0x79925587bE77C25b292C0ecA6FEdD3A3f07916F9";
const provider = new ethers.JsonRpcProvider("https://rpc.pulsechain.com");
const signer = new ethers.Wallet(OPERATOR_PRIVATE_KEY, provider);
const contract = new ethers.Contract(LIMIT_ORDER_ADDRESS, SwitchLimitOrderABI, signer);

const order = {
  maker: apiOrder.maker,
  tokenIn: apiOrder.tokenIn,
  tokenOut: apiOrder.tokenOut,
  amountIn: BigInt(apiOrder.amountIn),
  minAmountOut: BigInt(apiOrder.minAmountOut),
  deadline: BigInt(apiOrder.deadline),
  nonce: BigInt(apiOrder.nonce),
  feeOnOutput: apiOrder.feeOnOutput,
  recipient: apiOrder.recipient,
  unwrapOutput: apiOrder.unwrapOutput,
  partnerAddress: apiOrder.partnerAddress,
};

const routes = [{
  amountIn: routeAmountIn,
  hops: [{
    tokenIn: order.tokenIn,
    tokenOut: order.tokenOut,
    legs: [{ adapter: PULSEX_V2_ADAPTER, amountIn: routeAmountIn }],
  }],
}];

// Simulate, then send
await contract.fillOrder.staticCall(order, apiOrder.signature, routes, excessOnInput);
const tx = await contract.fillOrder(order, apiOrder.signature, routes, excessOnInput);
await tx.wait();
```

### Example: Direct Fill

```ts
// One-time approval
await outputToken.approve(LIMIT_ORDER_ADDRESS, ethers.MaxUint256);

// Calculate output amount
const fee = await contract.getFee(); // e.g. 30 = 0.30%
let outputAmount = order.minAmountOut;
if (order.feeOnOutput) {
  outputAmount = (order.minAmountOut * 10000n) / (10000n - fee);
}

// Use directToMaker=true for tax output tokens with feeOnOutput=false
// This sends output filler → recipient directly (1 transfer, 1 tax hit)
const directToMaker = !order.feeOnOutput && !order.unwrapOutput
  && apiOrder.tokenOutBuyTaxBps > 0;

// Simulate, then send
await contract.directFillOrder.staticCall(order, apiOrder.signature, outputAmount, directToMaker);
const tx = await contract.directFillOrder(order, apiOrder.signature, outputAmount, directToMaker);
await tx.wait();
```

---

## Profit Strategies & `excessOnInput`

> Applies to **route fills** only. For direct fills, your profit is always the input tokens received.

### How `feeOnOutput` Affects Token Flow

The `feeOnOutput` flag is set by the maker (baked into the EIP-712 signature). It cannot be changed.

**`feeOnOutput=false` (default):** LO contract pulls all `amountIn` from maker → deducts fee from input → approves remainder to Router → Router swaps → output goes to maker. You have full flexibility — choose `excessOnInput=true` or `false`.

**`feeOnOutput=true`:** Router pulls tokens directly from maker to DEX pools via `goSwitchFrom` → output goes to LO contract → fee deducted from output → remainder to maker. Minimizes transfers (critical for tax input tokens). You can only use `excessOnInput=false`.

### `excessOnInput` Choices

| `excessOnInput` | You profit in | How | `feeOnOutput=false` | `feeOnOutput=true` |
|---|---|---|---|---|
| `true` | tokenIn | Route **less** than `amountIn`; unrouted input sent to you | ✅ | ❌ Reverts |
| `false` | tokenOut | DEX output exceeds `minAmountOut`; surplus sent to you | ✅ | ✅ |

**Why `true` reverts with `feeOnOutput=true`:** The LO contract would need to pull excess tokens from the maker, but the maker only approved the Router.

### Decision Tree

Apply in order — first match wins:

```ts
function chooseExcessOnInput(
  order: { feeOnOutput: boolean; tokenIn: string; tokenOut: string },
  inputSellTaxBps: number,
  outputBuyTaxBps: number,
): boolean {
  // 1. feeOnOutput forces false
  if (order.feeOnOutput) return false;

  // 2. Avoid receiving tax tokens as profit
  if (inputSellTaxBps > 0 && outputBuyTaxBps > 0) return true;
  if (inputSellTaxBps > 0) return false;  // avoid taxed input
  if (outputBuyTaxBps > 0) return true;   // avoid taxed output

  // 3. Prefer more valuable token (customize to your strategy)
  const TOKEN_PRIORITY: Record<string, number> = {
    "0xa1077a294dde1b09bb078844df40758a5d0f9a27": 1,  // WPLS
    "0xefd766ccb38eaf1dfd701853bfce31359239f305": 2,  // DAI
    "0x15d38573d2feeb82e7ad5187ab8c1d52810b1f07": 2,  // USDC
    "0x0cb6f5a34ad42ec934882a05265a7d5f59b51a2f": 2,  // USDT
    "0x95b303987a60c71504d99aa1b13b4da07b0790ab": 3,  // PLSX
    "0x02dcdd04e3f455d838cd1249292c58f3b79e3c3c": 4,  // WETH
    "0xb17d901469b9208b17d916112988a3fed19b5ca1": 5,  // WBTC
    "0x2b591e99afe9f32eaa6214f7b7629768c40eeb39": 6,  // eHEX
    "0x57fde0a71132198bbec939b98976993d8d89d225": 6,  // pHEX
  };

  const inPri = TOKEN_PRIORITY[order.tokenIn.toLowerCase()] ?? Infinity;
  const outPri = TOKEN_PRIORITY[order.tokenOut.toLowerCase()] ?? Infinity;

  if (inPri < outPri) return true;
  if (outPri < inPri) return false;

  // 4. Default
  return true;
}
```

---

## Tax Token Handling

Some PulseChain tokens charge a fee on every `transfer()`. Orders include pre-computed tax data:

```json
{
  "tokenInSellTaxBps": 300,
  "tokenInBuyTaxBps": 0,
  "tokenOutSellTaxBps": 0,
  "tokenOutBuyTaxBps": 500
}
```

Values in basis points (100 = 1%). `0` = no tax detected.

- **`tokenInSellTaxBps`** — Tax when tokenIn is **sent**
- **`tokenInBuyTaxBps`** — Tax when tokenIn is **received**
- **`tokenOutSellTaxBps`** — Tax when tokenOut is **sent**
- **`tokenOutBuyTaxBps`** — Tax when tokenOut is **received**

### Transfer Tax by Fill Path

**Route fill, `feeOnOutput=false`:**
```
tokenIn: maker → LO contract (1 sell tax) → approved to router → pool (no additional tax)
tokenOut: pool → maker (1 buy tax)
```

**Route fill, `feeOnOutput=true`:**
```
tokenIn: maker → pool via goSwitchFrom (1 sell tax)
tokenOut: pool → LO contract (1 buy tax) → maker (2nd buy tax!)
```

> **Route scaling required:** The sell tax is applied during the `goSwitchFrom` transfer, so the pool receives less than the route's `amountIn`. If tokenIn has a sell tax, route amounts must be **pre-tax** values. See [Route Amount Scaling](#route-amount-scaling-for-sell-tax-input-feeoutputtrue) below.

**Direct fill, `directToMaker=false` (default):**
```
tokenIn: maker → LO contract (1 sell tax) → filler (1 buy tax)
tokenOut: filler → LO contract (1 sell tax) → maker (1 buy tax)
```
For tax output tokens, oversend so the contract receives ≥ `minAmountOut` + fee after tax.

**Direct fill, `directToMaker=true` (`feeOnOutput=false` only):**
```
tokenIn: maker → LO contract (1 sell tax) → filler (1 buy tax)
tokenOut: filler → maker directly (1 buy tax)  ← saves 1 tax hit!
```
Output goes directly from filler to the recipient in a single transfer, avoiding the sell tax from the intermediate filler → contract hop. Only works when `feeOnOutput=false` and `unwrapOutput=false`.

### Tax Token Decision Matrix

| Scenario | Maker's `feeOnOutput` | Why |
|---|---|---|
| Input has sell tax | `true` | 1 transfer (not 2) — avoids double sell tax |
| Output has buy tax | `false` | Output goes directly to maker (1 buy tax, not 2) |
| Both taxed | Order unlikely | Frontend blocks this |
| Neither taxed | `false` (default) | Maximum operator flexibility |

### `directToMaker` Decision (Direct Fills Only)

| `feeOnOutput` | `unwrapOutput` | Output taxed? | Use `directToMaker` | Why |
|---|---|---|---|---|
| `false` | `false` | Yes | **`true`** | Saves 1 buy tax (1 transfer vs 2) |
| `false` | `false` | No | `false` | No benefit — both paths identical |
| `false` | `true` | — | `false` (ignored) | Output needs unwrapping at contract |
| `true` | — | — | `false` (ignored) | Output needs fee deduction at contract |

```ts
function shouldDirectToMaker(order: LimitOrder, tokenOutBuyTaxBps: number): boolean {
  return !order.feeOnOutput && !order.unwrapOutput && tokenOutBuyTaxBps > 0;
}
```

### Profit Adjustments

1. **Input sell tax** → available routing input = `amountIn × (10000 - sellTaxBps) / 10000`
2. **Output buy tax** → ensure `outputAfterTax ≥ minAmountOut`
3. **Choose profit token wisely** — prefer `excessOnInput=true` when output is taxed, `false` when input is taxed

### First-Hop Adapter Restriction for Sell-Tax Inputs

> **Critical.** Using the wrong adapter type on the first hop causes `IIA` (Insufficient Input Amount) reverts from V3 pools.

When `tokenInSellTaxBps > 0`, the **first hop** of your route **must** use a UniswapV2-style adapter (e.g. PulseXV2, SushiV2, 9inchV2). Do **not** use V3, Phux, Tide, or PulseXStable adapters for the first hop.

**Why:** UniswapV3 pools use a callback payment model. During `pool.swap()`, the pool calls back to the adapter, which then transfers tokens from the router to the pool. This creates **two transfers** — router → adapter → pool — and the sell tax is applied on **each transfer**, so the pool receives far less than expected and reverts with "IIA".

UniswapV2-style adapters work differently: tokens are transferred to the pair first, and the adapter measures `balanceOf(pair) - reserves` to determine the actual received amount. This correctly handles tax tokens with only one tax hit.

**Subsequent hops are unrestricted.** After the first hop, intermediates (like WPLS, USDC, etc.) are not tax tokens, so any adapter type works fine for hop 2, hop 3, etc.

```
✅  TaxToken → [V2 adapter] → WPLS → [V3 adapter] → PLSX     (works)
✅  TaxToken → [V2 adapter] → PLSX                             (works)
❌  TaxToken → [V3 adapter] → WPLS → ...                       (reverts IIA)
❌  TaxToken → [Curve-style adapter] → WPLS → ...              (reverts)
```

**V2-style adapters (safe for first hop with tax tokens):**

| Adapter | Type |
|---|---|
| UniswapV2 | V2 direct pair |
| SushiV2 | V2 direct pair |
| PulseXV1 | V2 direct pair |
| PulseXV2 | V2 direct pair |
| 9inchV2 | V2 direct pair |
| DextopV2 | V2 direct pair |

**Unsafe for first hop with tax tokens:**

| Adapter | Type | Why |
|---|---|---|
| UniswapV3 | V3 callback | Double sell tax via callback |
| 9mmV3 | V3 callback | Double sell tax via callback |
| 9inchV3 | V3 callback | Double sell tax via callback |
| pDexV3 | V3 callback | Double sell tax via callback |
| DextopV3 | V3 callback | Double sell tax via callback |
| Phux | Curve-style | Not designed for tax tokens |
| Tide | Curve-style | Not designed for tax tokens |
| PulseXStable | Curve-style | Not designed for tax tokens |

```ts
function isFirstHopSafe(adapterAddress: string, tokenInSellTaxBps: number): boolean {
  if (tokenInSellTaxBps === 0) return true; // no tax — all adapters OK

  // V2-style adapters (safe for tax token first hop)
  const V2_ADAPTERS = new Set([
    "0x...", // UniswapV2  — replace with actual adapter addresses
    "0x...", // SushiV2
    "0x...", // PulseXV1
    "0x...", // PulseXV2
    "0x...", // 9inchV2
    "0x...", // DextopV2
  ].map(a => a.toLowerCase()));

  return V2_ADAPTERS.has(adapterAddress.toLowerCase());
}
```

### Last-Hop Adapter Restriction for Buy-Tax Outputs

> **Critical.** Using the wrong adapter type on the last hop causes inflated quotes and `transfer amount exceeds balance` reverts when the output token has a buy tax.

When `tokenOutBuyTaxBps > 0`, the **last hop** of your route **must** use a UniswapV2-style adapter. Do **not** use V3 or PulseXStable adapters for the last hop.

**Why:** UniswapV3 and PulseXStable adapters route output through an intermediate step — the pool sends tokens to the adapter, then the adapter forwards them to the recipient. This creates **two transfers** (pool → adapter → recipient), and the buy tax is applied on **each transfer**, so the recipient receives far less than the quoted amount.

UniswapV2-style adapters work differently: the pair sends tokens **directly to the recipient** in a single transfer, so the buy tax is only applied once.

**Intermediate hops are unrestricted.** Middle hops use non-tax intermediates (WPLS, PLSX, USDC, etc.), so any adapter type works. Only the final hop — where the output token is delivered — needs to be restricted.

```
✅  WPLS → [V3 adapter] → PLSX → [V2 adapter] → TaxToken     (works)
✅  WPLS → [V2 adapter] → TaxToken                             (works)
❌  WPLS → [V3 adapter] → TaxToken                             (double buy tax)
❌  WPLS → [V2 adapter] → PLSX → [V3 adapter] → TaxToken      (double buy tax on last hop)
```

**Safe for last hop with buy-tax output:**

| Adapter | Type | Why safe |
|---|---|---|
| UniswapV2 | V2 direct pair | Pair → recipient (1 transfer) |
| SushiV2 | V2 direct pair | Pair → recipient (1 transfer) |
| PulseXV1 | V2 direct pair | Pair → recipient (1 transfer) |
| PulseXV2 | V2 direct pair | Pair → recipient (1 transfer) |
| 9inchV2 | V2 direct pair | Pair → recipient (1 transfer) |
| DextopV2 | V2 direct pair | Pair → recipient (1 transfer) |

**Unsafe for last hop with buy-tax output:**

| Adapter | Type | Why unsafe |
|---|---|---|
| UniswapV3 | V3 callback | Pool → adapter → recipient (2 buy-tax hits) |
| 9mmV3 | V3 callback | Pool → adapter → recipient (2 buy-tax hits) |
| 9inchV3 | V3 callback | Pool → adapter → recipient (2 buy-tax hits) |
| pDexV3 | V3 callback | Pool → adapter → recipient (2 buy-tax hits) |
| DextopV3 | V3 callback | Pool → adapter → recipient (2 buy-tax hits) |
| PulseXStable | Curve-style | Pool → adapter → recipient (2 buy-tax hits) |
| Phux | Curve-style | Vault → adapter → recipient (2 buy-tax hits) |
| Tide | Curve-style | Vault → adapter → recipient (2 buy-tax hits) |

> **Note:** When both input has sell tax AND output has buy tax, both restrictions apply simultaneously. First hop → V2 only, last hop → V2 only, middle hops → unrestricted.

```ts
function isLastHopSafe(adapterAddress: string, tokenOutBuyTaxBps: number): boolean {
  if (tokenOutBuyTaxBps === 0) return true; // no buy tax — all adapters OK

  // V2-style adapters (safe for buy-tax output on last hop)
  const V2_ADAPTERS = new Set([
    "0x...", // UniswapV2  — replace with actual adapter addresses
    "0x...", // SushiV2
    "0x...", // PulseXV1
    "0x...", // PulseXV2
    "0x...", // 9inchV2
    "0x...", // DextopV2
  ].map(a => a.toLowerCase()));

  return V2_ADAPTERS.has(adapterAddress.toLowerCase());
}
```

### Route Amount Scaling for Sell-Tax Input (`feeOnOutput=true`)

> **Critical.** Getting this wrong causes `InsufficientOutput` reverts even when prices are favorable.

When `feeOnOutput=true` and `tokenIn` has a sell tax, route `amountIn` values must be **pre-tax (gross)** amounts, not the post-tax amounts you quoted at.

**Why:** `goSwitchFrom` transfers tokens directly from maker to pool. Sell tax reduces the amount in transit. If you set the route to the post-tax amount, the pool receives even less.

**Example (5% sell tax, 1000 token order):**

```
Quote at effective input:  1000 × 0.95 = 950 → DEX says 2000 WPLS out ✓

WRONG route.amountIn = 950:  950 × 0.95 = 902.5 arrives at pool → ~1900 WPLS → REVERT
RIGHT route.amountIn = 1000: 1000 × 0.95 = 950 arrives at pool → 2000 WPLS ✓
```

**Formula:**

```ts
function scaleRouteForSellTax(quotedInput: bigint, sellTaxBps: number, maxAmount: bigint): bigint {
  if (sellTaxBps === 0) return quotedInput;
  const preTax = (quotedInput * 10000n) / (10000n - BigInt(sellTaxBps));
  return preTax > maxAmount ? maxAmount : preTax;
}
```

| `feeOnOutput` | Sell tax on input? | Scale up route amounts? |
|---|---|---|
| `true` | Yes | **Yes** |
| `true` | No | No |
| `false` | Yes/No | No — LO contract pulls first, approves to router (no transfer tax) |

---

## On-Chain Structs

### `LimitOrder`

```solidity
struct LimitOrder {
    address maker;         // Signer / token owner
    address tokenIn;       // ERC-20 to sell
    address tokenOut;      // ERC-20 to buy
    uint256 amountIn;      // Maximum input amount
    uint256 minAmountOut;  // Minimum output the maker must receive
    uint256 deadline;      // Unix timestamp expiry (0 = no expiry)
    uint256 nonce;         // Unique per-maker (prevents double-fill)
    bool    feeOnOutput;   // true = fee from output, false = fee from input
    address recipient;     // Output recipient (usually maker's address)
    bool    unwrapOutput;  // true = unwrap WPLS to native PLS for recipient
    address partnerAddress; // Partner address for fee sharing (zero = no partner)
}
```

### Routing Structs

```solidity
struct RouteAllocation {
    uint256 amountIn;           // Total input for this route split
    HopAllocation[] hops;       // Sequential hops
}

struct HopAllocation {
    address tokenIn;
    address tokenOut;
    HopAdapterAllocation[] legs; // DEX legs within this hop
}

struct HopAdapterAllocation {
    address adapter;            // On-chain adapter contract
    uint256 amountIn;           // Input for this adapter
}
```

Routes can have multiple hops (multi-hop) and hops can have multiple legs (split across DEXes).

### Route Examples

**Single-hop:**

```ts
const routes = [{
  amountIn: ethers.parseUnits("950", 18),
  hops: [{
    tokenIn: WPLS, tokenOut: PLSX,
    legs: [{ adapter: PULSEX_V2_ADAPTER, amountIn: ethers.parseUnits("950", 18) }],
  }],
}];
```

**Multi-hop (WPLS → USDC → PLSX):**

```ts
const routes = [{
  amountIn: ethers.parseUnits("950", 18),
  hops: [
    { tokenIn: WPLS, tokenOut: USDC, legs: [{ adapter: pulseXV2, amountIn: wplsAmount }] },
    { tokenIn: USDC, tokenOut: PLSX, legs: [{ adapter: uniV3, amountIn: usdcAmount }] },
  ],
}];
```

**Split route (60/40 across DEXes within one hop):**

```ts
legs: [
  { adapter: pulseXV2, amountIn: (total * 60n) / 100n },
  { adapter: uniV3,    amountIn: (total * 40n) / 100n },
]
```

---

## Revert Errors

| Error | Meaning | Action |
|---|---|---|
| `InvalidSignature()` | Signature doesn't recover to maker | Skip order |
| `NonceAlreadyUsed()` | Already filled or cancelled | Remove from active list |
| `OrderExpired()` | Past deadline | Remove from active list |
| `InsufficientOutput()` | Output < `minAmountOut` after fees | Re-evaluate routes / increase `outputAmount` |
| `InvalidAmount()` | `amountIn` or `minAmountOut` is zero | Skip order |
| `ExcessiveFee()` | Fee exceeds maximum (shouldn't happen) | Contact Switch team |
| `RouteInputExceedsMax()` | Route total > available after fee | Reduce route input |
| `InvalidTokens()` | `tokenIn == tokenOut` or zero address | Skip order |
| `TransferFailed()` | Transfer failed (revoked allowance, etc.) | Re-check pre-flight |

---

## Reference

### View Functions

```solidity
function canFillOrder(LimitOrder calldata order, bytes calldata signature) external view returns (bool);
function isNonceUsed(address maker, uint256 nonce) external view returns (bool);
function getFee() external view returns (uint256);         // basis points, e.g. 30 = 0.30%
function domainSeparator() external view returns (bytes32);
```

### Events

```solidity
event OrderFilled(
    address indexed maker, address indexed filler, uint256 indexed nonce,
    address tokenIn, address tokenOut, uint256 amountIn,
    uint256 makerAmountOut, uint256 fillerProfit, bool excessOnInput
);
event NonceCancelled(address indexed maker, uint256 indexed nonce);
```

### ABIs

Available in [Switch-SDK](https://github.com/BuildTheTech/Switch-SDK):

- `abi/SwitchLimitOrderABI.json`
- `abi/SwitchRouterABI.json`

### EIP-712 Domain

```json
{
  "name": "SwitchLimitOrder",
  "version": "2",
  "chainId": 369,
  "verifyingContract": "0x79925587bE77C25b292C0ecA6FEdD3A3f07916F9"
}
```

---

## Support

- **Website:** [switch.win](https://switch.win)
- **Telegram:** [@BrandonDavisR2R](https://t.me/BrandonDavisR2R) · [@bttscott](https://t.me/bttscott) · [@shanebtt](https://t.me/shanebtt)

---

## License

MIT

---

*Last updated: March 2026*
