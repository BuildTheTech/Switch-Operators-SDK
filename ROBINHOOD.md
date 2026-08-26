# Switch Limit Order Operators - Robinhood Chain

> Integration guide for operators (fillers) of Switch Limit Orders on Robinhood Chain.

Operators monitor the Switch orderbook for signed limit orders and fill them
on-chain for profit. This guide covers Robinhood-specific contracts, routes,
fee modes, tax-token handling, native ETH orders, and operational safeguards.

**API:** `https://quote.switch.win` | **Network:** `robinhood` |
**Chain ID:** `4663`

For PulseChain, use the matching [PulseChain operator guide](PULSECHAIN.md).

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
   - [Route Amount Scaling](#route-amount-scaling-for-sell-tax-input-feeonoutputtrue)
10. [On-Chain Structs](#on-chain-structs)
11. [Revert Errors](#revert-errors)
12. [Reference](#reference)
13. [Support](#support)

---

## Overview

Switch Limit Orders use gasless EIP-712 signatures. ERC-20 tokens remain in
the maker's wallet until an order is filled. Native ETH orders are escrowed as
WETH by the native-flow contract and validated through ERC-1271.

As an operator, you:

1. Monitor the orderbook for active orders.
2. Verify on-chain fillability, authorization, and routing constraints.
3. Fill through DEX routing (`fillOrder`) or your own liquidity
   (`directFillOrder`).

The contract enforces the maker's `minAmountOut`. Any valid input or output
surplus selected by the fill mode is operator profit.

---

## Architecture

```text
MAKER
  |-- ERC-20: signs EIP-712 order; tokens remain in wallet
  `-- Native ETH: deposits into native-flow contract; ETH is wrapped to WETH

SWITCH BACKEND (quote.switch.win)
  |-- validates and stores orders
  |-- indexes fill and cancellation events
  `-- exposes the public order and network-config APIs

OPERATOR
  |-- polls active orders and reads each order's source contract
  |-- checks operator role, nonce, balances, allowances, taxes, and profit
  |-- fillOrder(order, signature, routes, excessOnInput)
  `-- directFillOrder(order, signature, outputAmount, directToMaker)

ON-CHAIN
  |-- SwitchLimitOrder: validates EIP-712/ERC-1271 and enforces minAmountOut
  |-- Native ETH flow: owns WETH-backed native-input orders
  `-- SwitchRouter: executes approved adapter routes
```

---

## Contract Addresses

| Contract | Address |
| --- | --- |
| **SwitchRouter** | `0x8b2bBdF41C1486b3482bD0e9603d72f012EE8599` |
| **Legacy SwitchRouterViewV2** (bound to the legacy Router) | `0xFF6b56d3F444eB5b7FA1db047F57140C84810376` |
| **SwitchLimitOrder** | `0x1E05115387f314398bbb1A808B25308E71150396` |
| **Native ETH flow** | `0x8170a3B0e2FD2e4333E0Ca9c9414B2D3dd6aF689` |
| **SwitchDirectFillQuoter** | `0x77b246C127c3c501Ec2836A2B53B555208b30B44` |
| **Limit-order adapter** (index 3) | `0xb13CC4C37e1C609617C51B5dCDf8e4Ae5721Faa4` |
| **Legacy limit-order adapter** (legacy Router index 3) | `0x412F625072c10e58C619D1e0b3C95cd3d5689871` |
| **WETH** | `0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73` |

Chain: **Robinhood Chain** (`4663`) | Fee denominator: `10000` basis points.

These are current defaults, not permanent constants. Fetch the
[config endpoint](#config-endpoint) at startup. Every order contains a
`limitOrderContract`; always call `fillOrder`, `directFillOrder`,
`canFillOrder`, and `isNonceUsed` on that per-order address.

Each SwitchLimitOrder exposes its immutable `SWITCH_ROUTER()`. Resolve that
router from the order contract and use adapters registered on that router. The
fill transaction target is the order's SwitchLimitOrder, never a swap quote's
`tx.to` address.

Core contract state and adapter registry verified August 25, 2026:

- `SwitchLimitOrder.getFee()` returns `30` bps (`0.30%`).
- `SwitchRouter.MIN_FEE()` returns `10` bps and `MAX_FEE()` returns `100` bps.
- The current LO is exempt from the Router's regular-swap fee, so a limit-order
  fill pays the LO's 30 bps fee rather than an additional Router fee.
- `MAX_OPERATOR_EXCESS_BPS()` returns `500` (5%).
- `operatorGateEnabled()` returns `true`.

Public operators must evaluate and fill only orders whose
`limitOrderContract` is the current SwitchLimitOrder above. Legacy addresses
remain in config for cancellation, indexing, and migration history; public
operator access is not granted on them.

Read these values on-chain instead of treating them as permanent constants.

---

## Order Lifecycle

```text
ACTIVE --> FILLED       operator calls fillOrder or directFillOrder
ACTIVE --> CANCELLED    maker invalidates nonce or native-flow owner cancels
ACTIVE --> EXPIRED      block.timestamp exceeds a nonzero deadline
```

Standard makers cancel by calling `invalidateNonce` on the order's
`limitOrderContract`. Native ETH owners call `cancelOrder` on the order's
`sourceContract`. There is no hosted REST cancellation endpoint; the backend
indexes the on-chain event.

An order can be filled only once. If the fill transaction reverts, its nonce
remains unused.

---

## Fetching Active Orders

No API key is required.

```http
GET https://quote.switch.win/limit-orders?network=robinhood&status=ACTIVE
```

### Query Parameters

| Parameter | Type | Default | Description |
| --- | --- | --- | --- |
| `status` | string | `ACTIVE` | `ACTIVE`, `FILLED`, `CANCELLED`, or `EXPIRED` |
| `maker` | address | - | Filter by signing maker; native orders use the flow address |
| `recipient` | address | - | Filter by output recipient |
| `owner` | address | - | Filter by the real owner, including native-flow orders |
| `partnerAddress` | address | - | Filter by partner |
| `tokenIn` | address | - | Filter by input token |
| `tokenOut` | address | - | Filter by output token |
| `pair` | string | - | Lowercase `tokenIn:tokenOut` pair key |
| `limit` | number | `100` | Page size, from 1 through 500 |
| `offset` | number | `0` | Page offset |

### Additional Endpoints

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/limit-orders/pairs?network=robinhood` | Active pairs and order counts |
| `GET` | `/limit-orders/stats?network=robinhood` | Network order statistics |
| `GET` | `/limit-orders/:maker/:nonce?network=robinhood` | Single order lookup |
| `GET` | `/limit-orders/config?network=robinhood` | Current contracts and EIP-712 domain |
| `GET` | `/swap/adapters?network=robinhood` | Current router adapter indices and addresses |

### Response

Amounts are integer strings in each token's native decimals. This abbreviated
example sells `0.01` WETH for at least `30` USDG (USDG has 6 decimals):

```json
{
  "total": 1,
  "limit": 100,
  "offset": 0,
  "orders": [
    {
      "maker": "0x...",
      "recipient": "0x...",
      "tokenIn": "0x0bd7d308f8e1639fab988df18a8011f41eacad73",
      "tokenOut": "0x5fc5360d0400a0fd4f2af552add042d716f1d168",
      "amountIn": "10000000000000000",
      "minAmountOut": "30000000",
      "deadline": 1798761600,
      "nonce": 1790000000000,
      "feeOnOutput": false,
      "unwrapOutput": false,
      "partnerAddress": "0x0000000000000000000000000000000000000000",
      "signature": "0x...",
      "status": "ACTIVE",
      "pairKey": "0x0bd7d308f8e1639fab988df18a8011f41eacad73:0x5fc5360d0400a0fd4f2af552add042d716f1d168",
      "tokenInSellTaxBps": 0,
      "tokenInBuyTaxBps": 0,
      "tokenOutSellTaxBps": 0,
      "tokenOutBuyTaxBps": 0,
      "limitOrderContract": "0x1e05115387f314398bbb1a808b25308e71150396",
      "sourceContract": "0x1e05115387f314398bbb1a808b25308e71150396",
      "orderType": "limitOrder",
      "contractVersion": "v2"
    }
  ]
}
```

Tax fields are basis points (`100` = 1%). A zero value means no tax was
detected when the order was created. Revalidate tax behavior before execution;
token rules can change.

Always use `limitOrderContract` from the order. `sourceContract`, `orderType`,
and `contractVersion` identify the order's origin and are especially important
for native-flow orders and future deployments.

### Config Endpoint

Retrieve all supported limit-order and native-flow contracts at startup:

```http
GET https://quote.switch.win/limit-orders/config?network=robinhood
```

Current response:

```json
{
  "limitOrderContract": "0x1e05115387f314398bbb1a808b25308e71150396",
  "allLimitOrderContracts": [
    "0x1e05115387f314398bbb1a808b25308e71150396",
    "0x752c50ddd3b426cae3d7a995f313ac74ac6b0230"
  ],
  "allPlsFlowContracts": [
    "0x8170a3b0e2fd2e4333e0ca9c9414b2d3dd6af689",
    "0x029ffc6af9112ea078f1d6f4a98826ddb2136cf6"
  ],
  "plsFlowContract": "0x8170a3b0e2fd2e4333e0ca9c9414b2d3dd6af689",
  "limitOrderContractVersions": {
    "0x1e05115387f314398bbb1a808b25308e71150396": "v2",
    "0x752c50ddd3b426cae3d7a995f313ac74ac6b0230": "v1"
  },
  "plsFlowContractVersions": {
    "0x8170a3b0e2fd2e4333e0ca9c9414b2d3dd6af689": "v2",
    "0x029ffc6af9112ea078f1d6f4a98826ddb2136cf6": "v1"
  },
  "eip712Domain": {
    "name": "SwitchLimitOrder",
    "version": "2",
    "chainId": 4663,
    "verifyingContract": "0x1E05115387f314398bbb1A808B25308E71150396"
  }
}
```

The API retains the legacy field names `plsFlowContract` and
`allPlsFlowContracts`; on Robinhood they refer to the native ETH flow.

### Polling Strategy

- Poll every 10 to 30 seconds and page until all active orders are processed.
- Refresh config periodically. Fill only `limitOrderContract` and accept native
  orders only from `plsFlowContract`. Use the `all*Contracts` arrays for
  cancellation, historical indexing, and migration visibility—not as a public
  operator fill allowlist.
- Use `/limit-orders/pairs` to focus routing work on active pairs.
- Re-evaluate same-pair orders after a fill because pool state changed.
- Treat API status as an index hint; confirm the nonce on-chain before sending.

---

## Evaluating an Order

### Pre-Flight Checks

For every fill:

1. Confirm `deadline == 0 || deadline > latestBlock.timestamp`.
2. Confirm `isNonceUsed(order.maker, order.nonce) == false` on the per-order LO.
3. Call `canFillOrder(order, signature)` on the same LO.
4. For ERC-20 orders, verify maker balance and allowance:
   - `feeOnOutput == false`: allowance to the per-order LO.
   - `feeOnOutput == true`: allowance to the LO's `SWITCH_ROUTER()`.
5. Confirm your operator wallet has `OPERATOR_ROLE` on the current LO when
   `operatorGateEnabled()` is true.
6. Resolve the router from `SWITCH_ROUTER()` and use only its approved adapters.
7. Read the current fee and simulate the exact transaction at the latest block.

```ts
const OPERATOR_ROLE = ethers.id("OPERATOR_ROLE");

for (const loAddress of [config.limitOrderContract]) {
  const lo = new ethers.Contract(loAddress, LO_ABI, provider);
  if (await lo.operatorGateEnabled()) {
    const allowed = await lo.hasRole(OPERATOR_ROLE, operator.address);
    if (!allowed) throw new Error(`Operator role missing on ${loAddress}`);
  }
}
```

`canFillOrder` covers the contract's signature, deadline, nonce, balance, and
allowance rules. It does not prove DEX liquidity, tax compatibility, operator
profit, or that the selected adapter data is valid.

### Native ETH Flow Orders

A native order has its `maker` set to a contract from
`allPlsFlowContracts`, its `tokenIn` set to WETH, and normally uses the empty
signature `0x`. The native-flow contract is the ERC-1271 maker; `recipient` is
the user who deposited ETH.

- Skip a user-wallet allowance check.
- Read the order tuple from `sourceContract.getOrder(nonce)` and use the
  on-chain tuple, especially its `recipient`.
- Confirm the target LO reports the flow nonce as unused.
- Confirm the flow's actual WETH balance can cover the fill.

```ts
const nativeFlow = new ethers.Contract(apiOrder.sourceContract, [
  "function getOrder(uint256 nonce) view returns (tuple(address originalMaker,address tokenOut,uint256 amountIn,uint256 minAmountOut,uint256 deadline,uint256 createdAt,bool feeOnOutput,bool unwrapOutput,bool active,address recipient,address partnerAddress))",
], provider);

const isNative = config.allPlsFlowContracts.some(
  (address: string) => address.toLowerCase() === apiOrder.maker.toLowerCase(),
);

if (isNative) {
  const order = await nativeFlow.getOrder(apiOrder.nonce);
  if (order.recipient.toLowerCase() !== apiOrder.recipient.toLowerCase()) {
    throw new Error("Native-flow API tuple differs from on-chain tuple");
  }
}
```

For the current v2 native flow, use the on-chain order tuple and
`isNonceUsed(flowAddress, nonce)` on the current LO as authoritative checks.
The legacy storage name `totalLockedWPLS` is retained by the ABI even though
the Robinhood asset is WETH.

For native-flow route fills with `excessOnInput=false`, route the full
executable input: the full `amountIn` for output-fee orders, or the maximum
post-input-fee amount for input-fee orders. Enforce this in the operator even
if a v1 deployment does not reject partial consumption itself; otherwise WETH
can remain stranded in the flow.

### Profitability Check

1. Quote executable routes across Robinhood venues.
2. Deduct the current protocol fee from the correct side.
3. Model all transfer taxes and round conservatively.
4. If either side is taxed, require every leg of every route hop to use a
   [tax-safe adapter](#first-hop-adapter-restriction-for-sell-tax-inputs).
5. Include approval, fill, L1-data, and replacement-transaction gas costs.
6. Add a reorg/slippage safety margin and require a minimum net profit.
7. Run `staticCall` using the same signer, calldata, block state, and value as
   the transaction you intend to send.

Switch exposes execution contracts and order data, not an operator routing
edge. Route discovery, quoting, and risk thresholds remain operator-owned.

---

## Filling an Order

### Route Fill (`fillOrder`)

You provide adapter routing instructions; the per-order LO invokes its router.

```solidity
function fillOrder(
    LimitOrder calldata order,
    bytes calldata signature,
    RouteAllocation[] calldata routes,
    bool excessOnInput
) external;
```

Profit is either unrouted input (`excessOnInput=true`) or output above the
maker requirement (`excessOnInput=false`).

The protected LO caps operator-retained route surplus at 5%: input-side profit
is capped at 5% of executable input and output-side profit at 5% of the signed
`minAmountOut`. Any additional improvement is delivered or refunded to the
maker.

### Direct Fill (`directFillOrder`)

You supply output tokens and receive the maker's input without a DEX route.

```solidity
function directFillOrder(
    LimitOrder calldata order,
    bytes calldata signature,
    uint256 outputAmount,
    bool directToMaker
) external;
```

`outputAmount` must leave the maker at least the protected floor after output
fees and transfer taxes. The contract obtains a direct-adapter quote through
`directFillQuoter()` and requires the maker to receive the greater of the
signed `minAmountOut` and 95% of that quote. A missing or zero quote fails
closed. For an untaxed output-fee order, first calculate and then gross up the
protected floor as shown in the direct-fill example below.

For a taxed output, calculate against observed balance deltas and add a safety
margin. All post-fee output above the protected floor goes to the maker. Your
direct-fill profit is input received minus output spent and gas.

### Approval Requirements

| Party | `feeOnOutput=false` | `feeOnOutput=true` |
| --- | --- | --- |
| Maker | Approves the order's LO | Approves the LO's router |
| Operator, direct fill | Approves the order's LO for tokenOut | Approves the order's LO for tokenOut |
| Operator, route fill | No token approval for the route itself | Input-side excess may require special LO allowance; simulate |

Public operators should reject orders pointing to an older LO. Approve the
current LO for direct-fill output tokens.

### Always Simulate First

```ts
try {
  await lo.fillOrder.staticCall(order, signature, routes, excessOnInput);
} catch (error) {
  console.error("Fill simulation reverted", error);
  return;
}

const tx = await lo.fillOrder(order, signature, routes, excessOnInput);
await tx.wait();
```

A simulation is short-lived evidence, not a guarantee. Set route minimums and
transaction replacement policy for state changes between simulation and mine.

### Example: Route Fill

```ts
import { ethers } from "ethers";
import SwitchLimitOrderABI from "./abi/SwitchLimitOrderABI.json";

const provider = new ethers.JsonRpcProvider(
  process.env.ROBINHOOD_RPC_URL ?? "https://rpc.mainnet.chain.robinhood.com",
);
const signer = new ethers.Wallet(process.env.OPERATOR_PRIVATE_KEY!, provider);

// Use the address carried by this order; do not hardcode the default LO.
const lo = new ethers.Contract(
  apiOrder.limitOrderContract,
  SwitchLimitOrderABI,
  signer,
);

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

const UNISWAP_V2_ADAPTER = "0x7a14d7A8509a66209D4332843b983b29bF5604A4";
const routes = [{
  amountIn: routeAmountIn,
  hops: [{
    tokenIn: order.tokenIn,
    tokenOut: order.tokenOut,
    legs: [{
      adapter: UNISWAP_V2_ADAPTER,
      amountIn: routeAmountIn,
      fee: 0,
      data: "0x",
    }],
  }],
}];

await lo.fillOrder.staticCall(order, apiOrder.signature, routes, excessOnInput);
const receipt = await (
  await lo.fillOrder(order, apiOrder.signature, routes, excessOnInput)
).wait();
```

### Example: Direct Fill

```ts
const feeBps = BigInt(await lo.getFee());
const quoter = new ethers.Contract(await lo.directFillQuoter(), [
  "function quoteExactInput(uint256,address,address) view returns (uint256)",
], provider);
const outToken = order.unwrapOutput ? await lo.WNATIVE() : order.tokenOut;
const quoteInput = order.feeOnOutput
  ? order.amountIn
  : order.amountIn - (order.amountIn * feeBps) / 10_000n;
const fairQuote = await quoter.quoteExactInput(quoteInput, order.tokenIn, outToken);
const marketFloor = (fairQuote * 9_500n + 9_999n) / 10_000n;
const protectedFloor = order.minAmountOut > marketFloor
  ? order.minAmountOut
  : marketFloor;

let outputAmount = protectedFloor;

if (order.feeOnOutput) {
  outputAmount =
    (protectedFloor * 10_000n + (10_000n - feeBps) - 1n) /
    (10_000n - feeBps);
}

// For taxed input/output tokens, use balance-delta-aware estimates, oversend
// as needed, and rely on the exact staticCall below as the final gate.

const outputToken = new ethers.Contract(order.tokenOut, ERC20_ABI, signer);
await (await outputToken.approve(await lo.getAddress(), outputAmount)).wait();

const directToMaker =
  !order.feeOnOutput &&
  !order.unwrapOutput &&
  apiOrder.tokenOutBuyTaxBps > 0;

await lo.directFillOrder.staticCall(
  order,
  apiOrder.signature,
  outputAmount,
  directToMaker,
);
await (
  await lo.directFillOrder(
    order,
    apiOrder.signature,
    outputAmount,
    directToMaker,
  )
).wait();
```

---

## Profit Strategies & `excessOnInput`

This flag applies only to route fills. Direct-fill profit is the difference
between input received and output spent.

### How `feeOnOutput` Affects Token Flow

`feeOnOutput` is signed by the maker and cannot be changed by an operator.

- `false`: the LO charges the protocol fee on input before routing.
- `true`: the router sources input under the output-fee path, then the LO
  charges the fee from output before satisfying the maker.

This choice affects maker allowances, executable route input, tax exposure,
and which `excessOnInput` modes are practical.

### `excessOnInput` Choices

| `feeOnOutput` | `excessOnInput` | Profit asset | Operator guidance |
| --- | --- | --- | --- |
| `false` | `true` | tokenIn | Normal input-surplus mode; route less than executable input |
| `false` | `false` | tokenOut | Normal output-surplus mode; route the executable input |
| `true` | `false` | tokenOut | Normal output-fee mode |
| `true` | `true` | tokenIn | Usually avoid; may need additional maker allowance to the LO |

`feeOnOutput=true` plus `excessOnInput=true` is not universally impossible,
but the LO may need to pull the surplus itself while the maker normally
approved only the router. Use it only when the exact deployment, allowance,
and simulation prove the path is valid.

For a native-flow order with output-side surplus, always route the full
executable input as described in [Native ETH Flow Orders](#native-eth-flow-orders).

### Decision Tree

```ts
function chooseExcessOnInput(order: Order, taxes: Taxes): boolean {
  const inputTaxed = taxes.tokenInSellTaxBps > 0;
  const outputTaxed = taxes.tokenOutBuyTaxBps > 0;

  // The supported both-tax strategy is input-side excess with an input fee.
  if (inputTaxed && outputTaxed) {
    if (order.feeOnOutput) throw new Error("Both-tax order must use input fee");
    return true;
  }

  // Output-fee orders normally use output surplus.
  if (order.feeOnOutput) return false;

  // Avoid taking the taxed asset as profit when one side is taxed.
  if (inputTaxed) return false;
  if (outputTaxed) return true;

  const priority: Record<string, number> = {
    "0x0bd7d308f8e1639fab988df18a8011f41eacad73": 1, // WETH / native ETH
    "0x5fc5360d0400a0fd4f2af552add042d716f1d168": 2, // USDG
    "0x0339f5459fc690ac85f1782e15782a151b4a9e1b": 3, // WALLET
    "0x58f693a30f124e59b125f7c7b837b0f6bbaf5a45": 4, // SEEDCOIN
    "0x020bfc650a365f8bb26819deaabf3e21291018b4": 5, // CASHCAT
  };

  const inputRank = priority[order.tokenIn.toLowerCase()] ?? Infinity;
  const outputRank = priority[order.tokenOut.toLowerCase()] ?? Infinity;
  if (inputRank !== outputRank) return inputRank < outputRank;
  return true;
}
```

This is a starting policy. Replace token ranking, liquidity limits, inventory
targets, and minimum profit with your own risk model.

---

## Tax Token Handling

Some Robinhood tokens deduct value during `transfer` or `transferFrom`.
Orders carry four detected tax fields:

```json
{
  "tokenInSellTaxBps": 300,
  "tokenInBuyTaxBps": 0,
  "tokenOutSellTaxBps": 0,
  "tokenOutBuyTaxBps": 500
}
```

Values are basis points. Treat them as discovery metadata and confirm actual
balance deltas in a simulation because a token can change its rules.

### Transfer Tax by Fill Path

Route fill behavior depends on fee mode and adapter implementation:

- With `feeOnOutput=false`, the LO collects input, charges the input fee, and
  makes the executable amount available to the router.
- With `feeOnOutput=true`, the router sources input from the maker and the LO
  charges output after the route.
- The maker must receive at least `minAmountOut` after all output-side fees and
  taxes.
- A tax-safe adapter measures actual balance deltas and avoids callback or
  forwarding assumptions that fail for taxed tokens.

Direct fills normally move output through the LO. For eligible input-fee
orders, `directToMaker=true` removes that intermediate output transfer and can
avoid an extra output-tax hit.

Do not rely only on a nominal quote. Compare balances before and after the
exact simulated call and round every gross-up conservatively.

### Tax Token Decision Matrix

| Tax condition | Preferred signed mode | Route rule | Preferred profit side |
| --- | --- | --- | --- |
| Input sell tax only | Usually `feeOnOutput=true` | Every leg tax-safe | Output (`excessOnInput=false`) |
| Output buy tax only | `feeOnOutput=false` | Every leg tax-safe | Input (`excessOnInput=true`) |
| Both sides taxed | `feeOnOutput=false` | Every leg tax-safe | Input (`excessOnInput=true`) |
| Neither side taxed | Either | Any compatible approved adapter | Strategy-dependent |

Both-tax orders are supported on Robinhood. They are not frontend-blocked;
use the input-fee, input-surplus strategy and verify the complete route.

### `directToMaker` Decision (Direct Fills Only)

| `feeOnOutput` | `unwrapOutput` | Output taxed | `directToMaker` |
| --- | --- | --- | --- |
| `false` | `false` | Yes | `true` |
| `false` | `false` | No | Usually `false`; no tax benefit |
| `false` | `true` | Any | `false`; contract must unwrap |
| `true` | Any | Any | `false`; contract must charge output fee |

```ts
function shouldDirectToMaker(order: Order, outputBuyTaxBps: number): boolean {
  return !order.feeOnOutput && !order.unwrapOutput && outputBuyTaxBps > 0;
}
```

This flag does not relax `minAmountOut`; simulate the recipient's actual
balance increase.

### Profit Adjustments

1. Gross up taxed transfers using ceiling division, never floating point.
2. For direct-pair adapters, quote the amount the pair actually receives, not
   the nominal transfer amount. For Flap, quote the full adapter input because
   its Portal quote already models the taxable transfer.
3. Require the recipient's net output to meet `minAmountOut`.
4. Avoid receiving a taxed token as profit when the other side is practical.
5. Add uncertainty margin for dynamic, allowlisted, or mutable tax logic.

For a simple known tax, a conservative gross-up helper is:

```ts
function grossForNet(net: bigint, taxBps: bigint): bigint {
  const denominator = 10_000n - taxBps;
  return (net * 10_000n + denominator - 1n) / denominator;
}
```

### First-Hop Adapter Restriction for Sell-Tax Inputs

On Robinhood this is intentionally stricter than the PulseChain rule: when
`tokenInSellTaxBps > 0`, **every leg of every hop** must use a tax-safe
adapter. Restricting only the first hop is insufficient for the production
routing policy.

Current tax-safe adapters:

| Index | Venue | Adapter |
| ---: | --- | --- |
| 0 | UniswapV2 | `0x7a14d7A8509a66209D4332843b983b29bF5604A4` |
| 4 | SwapHoodV2 | `0x6D8746f02e52944c13824fA691c6f4186E463354` |
| 7 | SheriffV2 | `0xBDB3EB0355981500f58C9bc77c3E61762844A146` |
| 10 | CatnipV2 | `0x5b2Ca358d56490Dc86224D502522314De7707237` |
| 11 | PancakeSwapV2 | `0x3B6e71A59553143937Fef74a7B50AFD24528786E` |
| 13 | RobinSwapV2 | `0xF694145F104c3C24f723301e160D2Ccf0Db31FE6` |
| 16 | GigaV2 | `0xa379c7D17F7fEe735773879D4069886B117AB54a` |
| 18 | Flap | `0x6af2A4475C44d5833575150Bf7C3D3FE6Bf4F344` |
| 20 | RamsesV2 | `0x5fe3b873c222e76f7630b40052f07ee06196E6d3` |

```ts
const TAX_SAFE_ADAPTERS = new Set([
  "0x7a14d7a8509a66209d4332843b983b29bf5604a4",
  "0x6d8746f02e52944c13824fa691c6f4186e463354",
  "0xbdb3eb0355981500f58c9bc77c3e61762844a146",
  "0x5b2ca358d56490dc86224d502522314de7707237",
  "0x3b6e71a59553143937fef74a7b50afd24528786e",
  "0xf694145f104c3c24f723301e160d2ccf0db31fe6",
  "0xa379c7d17f7fee735773879d4069886b117ab54a",
  "0x6af2a4475c44d5833575150bf7c3d3fe6bf4f344",
  "0x5fe3b873c222e76f7630b40052f07ee06196e6d3",
]);

function assertTaxSafeRoute(routes: Route[], eitherSideTaxed: boolean): void {
  if (!eitherSideTaxed) return;
  for (const route of routes) {
    for (const hop of route.hops) {
      for (const leg of hop.legs) {
        if (!TAX_SAFE_ADAPTERS.has(leg.adapter.toLowerCase())) {
          throw new Error(`Tax-unsafe adapter: ${leg.adapter}`);
        }
      }
    }
  }
}
```

GIGA V2 and Ramses V2 use direct-pair execution, so tax-token input goes
straight from the router to the pair and the swap uses the actual amount
received. Flap is the deliberate exception to the direct-pair family: only its
Portal may execute the curve pair. Flap tokens exempt the maker/router to
adapter transfer, and the Portal quote already includes the taxable transfer
in both directions. Do not subtract or gross up the declared token tax again
for a Flap leg.

V3, V4, Algebra, and other callback-style adapters are not tax-safe for these
orders even if a nominal quoter returns a value.

### Last-Hop Adapter Restriction for Buy-Tax Outputs

When `tokenOutBuyTaxBps > 0`, apply the same route-wide restriction: every leg
of every hop must be one of the eight tax-safe adapters above. Restricting only
the last hop is not enough.

The route-wide policy makes split and multi-hop validation deterministic and
prevents an intermediate adapter from introducing an extra taxed transfer or
assuming nominal input/output balances. If both sides are taxed, there is one
restriction, not two independent exceptions: the entire route is tax-safe.

### Route Amount Scaling for Sell-Tax Input (`feeOnOutput=true`)

For an output-fee order with taxed input, route amounts are gross pre-tax
amounts. Quote using the estimated net pool input, then gross the route amount
back up without exceeding `order.amountIn`.

```ts
function scaleRouteForSellTax(
  quotedNetInput: bigint,
  sellTaxBps: bigint,
  maxGrossInput: bigint,
): bigint {
  if (sellTaxBps === 0n) return quotedNetInput;
  const denominator = 10_000n - sellTaxBps;
  const gross = (quotedNetInput * 10_000n + denominator - 1n) / denominator;
  return gross > maxGrossInput ? maxGrossInput : gross;
}
```

Example: for a 5% tax and a quote requiring `950` net units, route `1000`
gross units. Routing `950` gross would deliver only `902.5` units. Always
confirm actual received amounts with `staticCall` because token tax formulas
may not be linear.

Do not apply this scaling helper to Flap. The Flap adapter receives its planned
input in full, then the Portal performs the taxable transfer and returns a
post-tax quote. Quote and route that full adapter input; applying the generic
sell-tax reduction or gross-up would count the tax twice.

---

## On-Chain Structs

### `LimitOrder`

```solidity
struct LimitOrder {
    address maker;
    address tokenIn;
    address tokenOut;
    uint256 amountIn;
    uint256 minAmountOut;
    uint256 deadline;
    uint256 nonce;
    bool feeOnOutput;
    address recipient;
    bool unwrapOutput;
    address partnerAddress;
}
```

Use field order exactly as shown when hashing or encoding. JavaScript numbers
are unsafe for token amounts, deadlines, and nonces; use `bigint` or decimal
strings.

### Routing Structs

```solidity
struct RouteAllocation {
    uint256 amountIn;
    HopAllocation[] hops;
}

struct HopAllocation {
    address tokenIn;
    address tokenOut;
    HopAdapterAllocation[] legs;
}

struct HopAdapterAllocation {
    address adapter;
    uint256 amountIn;
    uint24 fee;
    bytes data;
}
```

The `data` field is required. Use `0x` when an adapter needs no extra payload.
V3/Algebra use their supported fee value; V4 requires exact pool-key data and
the adapter's `swapWithData` path. Current Robinhood V4 support is limited to
hookless, static-fee pools.

### Route Examples

Single-hop UniswapV2:

```ts
const routes = [{
  amountIn: input,
  hops: [{
    tokenIn: WETH,
    tokenOut: USDG,
    legs: [{ adapter: uniswapV2, amountIn: input, fee: 0, data: "0x" }],
  }],
}];
```

Multi-hop through USDG:

```ts
const routes = [{
  amountIn: wethInput,
  hops: [
    {
      tokenIn: WETH,
      tokenOut: USDG,
      legs: [{ adapter: swapHoodV2, amountIn: wethInput, fee: 0, data: "0x" }],
    },
    {
      tokenIn: USDG,
      tokenOut: CASHCAT,
      legs: [{ adapter: catnipV2, amountIn: usdgOutput, fee: 0, data: "0x" }],
    },
  ],
}];
```

Split one hop across venues:

```ts
const legs = [
  { adapter: uniswapV2, amountIn: total * 60n / 100n, fee: 0, data: "0x" },
  { adapter: swapHoodV2, amountIn: total * 40n / 100n, fee: 0, data: "0x" },
];
```

For a taxed pair, validate all legs in all three examples with
`assertTaxSafeRoute`.

---

## Revert Errors

### SwitchLimitOrder Errors

| Selector | Error | Likely cause | Operator action |
| --- | --- | --- | --- |
| `0x8baa579f` | `InvalidSignature()` | Wrong tuple, recipient, domain, or signature | Re-read order and native-flow tuple |
| `0x1fb09b80` | `NonceAlreadyUsed()` | Filled or cancelled | Remove from active queue |
| `0xc56873ba` | `OrderExpired()` | Deadline passed | Remove from active queue |
| `0xbb2875c3` | `InsufficientOutput()` | Maker net output is too low | Requote or increase output |
| `0x2c5211c6` | `InvalidAmount()` | Zero amount | Reject order |
| `0x2977da44` | `ExcessiveFee()` | Fee outside supported range | Refresh contract configuration |
| `0xb6972a87` | `RouteInputExceedsMax()` | Route total exceeds executable input | Reduce or correct route input |
| `0x672215de` | `InvalidTokens()` | Zero or identical tokens | Reject order |
| `0x90b8ec18` | `TransferFailed()` | Balance, allowance, tax, or token rule changed | Repeat pre-flight and simulate |
| `0xae5e3e00` | `OperatorOnly()` | Caller lacks role while gate is enabled | Obtain role on this LO |
| `0x42e89a78` | `NativeFlowInputNotFullyConsumed()` | Native-flow output-surplus route leaves input unconsumed | Route the full executable input |
| `0x54346455` | `RouteTokenInMismatch()` | Route does not begin with the signed input token | Rebuild the route |
| `0xa587e83b` | `RouteTokenOutMismatch()` | Route does not end with the signed output token | Rebuild the route |
| `0xdcd56106` | `InvalidDirectFillQuoter()` | Admin supplied an incompatible quoter | Stop direct fills and report configuration |
| `0x0e4c7aa9` | `DirectFillQuoteUnavailable()` | Protected direct quote is unavailable or zero | Defer direct fill; do not bypass the floor |
| `0xca8ecf0e` | `DirectFillPriceTooLow()` | Maker would receive less than the protected floor | Increase `outputAmount` or skip |
| `0xb68f3df5` | `OperatorManagerOnly()` | Caller cannot manage operator access | Use admin or configured operator manager |

For a native-flow order with `InvalidSignature`, first compare the API order
to `sourceContract.getOrder(nonce)`. The empty signature is validated through
ERC-1271 and the on-chain recipient is part of the signed digest.

Newer deployments may also reject partial native-flow consumption explicitly.
Regardless of version, enforce full executable input for native output-surplus
fills in the operator.

### SwitchRouter Errors (during route fills)

| Selector | Error | Likely cause | Operator action |
| --- | --- | --- | --- |
| `0x025dbdd4` | `InsufficientFee()` | Fee below router minimum | Refresh `getFee()` and router limits |
| `0x2977da44` | `ExcessiveFee()` | Fee above router maximum | Refresh `getFee()` and router limits |
| `0x4a2ab023` | `FinalAmountOutTooLow()` | Route output below minimum | Requote route |
| `0x5725cad2` | `EmptySplit()` | Empty route allocation | Add a nonempty route |
| `0xdceb8b7a` | `SplitMixedTokenIn()` | Split starts with different tokens | Fix route construction |
| `0x45a68c8c` | `SplitMixedTokenOut()` | Split ends with different tokens | Fix route construction |
| `0xaf458c07` | `ZeroInput()` | Route, hop, or leg input is zero | Remove zero allocations |
| `0xbc6f88c5` | `MsgValueMismatch()` | Native value differs from route input | Correct `msg.value` |
| `0x906478dc` | `PathNeedsBeginWithWPLS()` | Native-input route does not begin with wrapped native | Begin with Robinhood WETH |
| `0x037ccaee` | `PathNeedsEndWithWPLS()` | Native-output route does not end with wrapped native | End with Robinhood WETH |
| `0x0c48343e` | `AdapterNotApproved(address)` | Adapter is not registered | Refresh adapter registry |
| `0xb19ef58a` | `EmptyHop()` | Hop has no legs | Add at least one leg |
| `0x73620122` | `FeeNotSupported()` | Adapter/path does not support selected mode | Change route or fee mode |

The deployed router retains the legacy `WPLS` error names; on Robinhood their
wrapped-native requirement refers to WETH.

### Matching Selectors in Code

```ts
const ERROR_BY_SELECTOR: Record<string, string> = {
  "0x8baa579f": "InvalidSignature",
  "0x1fb09b80": "NonceAlreadyUsed",
  "0xc56873ba": "OrderExpired",
  "0xbb2875c3": "InsufficientOutput",
  "0x2c5211c6": "InvalidAmount",
  "0x2977da44": "ExcessiveFee",
  "0xb6972a87": "RouteInputExceedsMax",
  "0x672215de": "InvalidTokens",
  "0x90b8ec18": "TransferFailed",
  "0xae5e3e00": "OperatorOnly",
  "0x42e89a78": "NativeFlowInputNotFullyConsumed",
  "0x54346455": "RouteTokenInMismatch",
  "0xa587e83b": "RouteTokenOutMismatch",
  "0xdcd56106": "InvalidDirectFillQuoter",
  "0x0e4c7aa9": "DirectFillQuoteUnavailable",
  "0xca8ecf0e": "DirectFillPriceTooLow",
  "0xb68f3df5": "OperatorManagerOnly",
  "0x025dbdd4": "InsufficientFee",
  "0x4a2ab023": "FinalAmountOutTooLow",
  "0x5725cad2": "EmptySplit",
  "0xdceb8b7a": "SplitMixedTokenIn",
  "0x45a68c8c": "SplitMixedTokenOut",
  "0xaf458c07": "ZeroInput",
  "0xbc6f88c5": "MsgValueMismatch",
  "0x906478dc": "PathNeedsBeginWithWPLS",
  "0x037ccaee": "PathNeedsEndWithWPLS",
  "0x0c48343e": "AdapterNotApproved",
  "0xb19ef58a": "EmptyHop",
  "0x73620122": "FeeNotSupported",
};

function matchErrorSelector(revertData?: string): string | undefined {
  if (!revertData || revertData.length < 10) return undefined;
  return ERROR_BY_SELECTOR[revertData.slice(0, 10).toLowerCase()];
}
```

---

## Reference

### View Functions

```solidity
function canFillOrder(LimitOrder calldata order, bytes calldata signature) external view returns (bool);
function isNonceUsed(address maker, uint256 nonce) external view returns (bool);
function getFee() external view returns (uint256);
function MAX_OPERATOR_EXCESS_BPS() external view returns (uint256);
function directFillQuoter() external view returns (address);
function SWITCH_ROUTER() external view returns (address);
function WNATIVE() external view returns (address);
function operatorGateEnabled() external view returns (bool);
function hasRole(bytes32 role, address account) external view returns (bool);
function domainSeparator() external view returns (bytes32);
```

### Events

```solidity
event OrderFilled(
    address indexed maker,
    address indexed filler,
    uint256 indexed nonce,
    address tokenIn,
    address tokenOut,
    uint256 amountIn,
    uint256 makerAmountOut,
    uint256 fillerProfit,
    bool excessOnInput,
    address partnerAddress,
    bool directFill
);

event NonceCancelled(address indexed maker, uint256 indexed nonce);
```

Native-flow cancellation is emitted by the order's source flow. Public fill
workers should index current contracts for actionable orders; archival systems
may index every contract returned by config.

### ABIs

The npm package is [`@switch-win/sdk`](https://www.npmjs.com/package/@switch-win/sdk).
Use version `1.2.8` or newer for the protected deployments, current ABIs, and
Robinhood adapter metadata:

```bash
npm install @switch-win/sdk@^1.2.8
```

The package/repository supplies the SwitchLimitOrder and SwitchRouter ABIs.
Use the ABI matching `contractVersion` and retain the native-flow ABI for
`getOrder` and `cancelOrder` reads.

### EIP-712 Domain

The current default domain is:

```json
{
  "name": "SwitchLimitOrder",
  "version": "2",
  "chainId": 4663,
  "verifyingContract": "0x1E05115387f314398bbb1A808B25308E71150396"
}
```

Use the config endpoint and the target order's contract. Do not reuse the
PulseChain chain ID or verifying contract.

### Routing Venues

Current Robinhood router adapters:

| Index | Venue | Adapter | Tax-safe |
| ---: | --- | --- | :---: |
| 0 | UniswapV2 | `0x7a14d7A8509a66209D4332843b983b29bF5604A4` | Yes |
| 1 | UniswapV3 | `0xbcA08f296d9Ba0dc19Aa0E05D355365cE29A3205` | No |
| 2 | UniswapV4 | `0xB9885d3C55e79499bf887F2fBe445e01A8cFFf1c` | No |
| 3 | SwitchLimitOrders | `0xb13CC4C37e1C609617C51B5dCDf8e4Ae5721Faa4` | No |
| 4 | SwapHoodV2 | `0x6D8746f02e52944c13824fA691c6f4186E463354` | Yes |
| 5 | SwapHoodV3 | `0x9645dE0AcB48F0AAefdBEb423F0558457907DE98` | No |
| 6 | Up33 | `0x388179D2FB0ABcE9b03068916aF8a3c4dfD023c8` | No |
| 7 | SheriffV2 | `0xBDB3EB0355981500f58C9bc77c3E61762844A146` | Yes |
| 8 | SheriffAlgebra | `0xaC4da986100724983042Ec28c28db243E2f828CB` | No |
| 9 | AeonAlgebra | `0x20615954FB87360139e7DdDB519359498EbD1904` | No |
| 10 | CatnipV2 | `0x5b2Ca358d56490Dc86224D502522314De7707237` | Yes |
| 11 | PancakeSwapV2 | `0x3B6e71A59553143937Fef74a7B50AFD24528786E` | Yes |
| 12 | PancakeSwapV3 | `0xD2Adac87bab4f0f99CF2a21c552c88d1C9825cCC` | No |
| 13 | RobinSwapV2 | `0xF694145F104c3C24f723301e160D2Ccf0Db31FE6` | Yes |
| 14 | RobinSwapV3 | `0x798f77D63b46b0E019de206E111e5ea5CC16BEc8` | No |
| 15 | SushiSwapV3 | `0xca3EA0Fd6E31f94c81B6586836790adE638313ED` | No |
| 16 | GigaV2 | `0xa379c7D17F7fEe735773879D4069886B117AB54a` | Yes |
| 17 | GigaV3 | `0xcAa612CDe3d3FbE97Be97eB5f79BC91597432d55` | No |
| 18 | Flap | `0x6af2A4475C44d5833575150Bf7C3D3FE6Bf4F344` | Yes |
| 19 | RamsesV3 | `0xdBf182774C60932c6fe1Bf3FFaB8Ca28CCb0dC17` | No |
| 20 | RamsesV2 | `0x5fe3b873c222e76f7630b40052f07ee06196E6d3` | Yes |

PancakeSwap V3 was deployed in
[`0xa82a...cfa8`](https://robinhoodchain.blockscout.com/tx/0xa82a53e8e32e7a26be720d62e157c820e8c25ccbb3826e604435480606e0cfa8)
and inserted at index `12` in router update
[`0xae59...94b5`](https://robinhoodchain.blockscout.com/tx/0xae5949bad45dfbdcd930a2fd39e5fb22914b51704f8c7b137f996a154e8d94b5).

RobinSwap V2 was deployed in
[`0x2263...d1cc`](https://robinhoodchain.blockscout.com/tx/0x22633d0c9be2f117474bb7993d037c3b9b9f568a9e91b738fbd32cc0b5ddd1cc)
and inserted immediately before RobinSwap V3 in router update
[`0x093a...96fb`](https://robinhoodchain.blockscout.com/tx/0x093a2f30536b362f238bd23c79c8c4cd1d6e0f561aba9b76bb108713759896fb).

Adapter registration can change. Refresh the router registry and adapter
metadata before route construction.

For PancakeSwap V3, select from fee tiers `100`, `500`, `2500`, and `10000`.
For Ramses V3, select from tick spacings `1`, `5`, `10`, `50`, `100`, and `200`
and store the winner in the route leg's `fee` field; do not treat that value as
a fixed fee percentage. GIGA V2 and Ramses V2 both use each selected pair's
live `getAmountOut` implementation and pair-owned fee. Operators should not
hard-code either venue's V2 fee percentage.

### Uniswap Infrastructure

| Component | Address |
| --- | --- |
| V2 Factory | `0x8bcEaA40B9AcdfAedF85AdF4FF01F5Ad6517937f` |
| V3 Factory | `0x1f7d7550B1b028f7571E69A784071F0205FD2EfA` |
| V3 QuoterV2 | `0x33e885eD0Ec9bF04EcfB19341582aADCb4c8A9E7` |
| V3 SwapRouter02 | `0xCaf681a66D020601342297493863E78C959E5cb2` |
| V4 PoolManager | `0x8366a39cc670b4001a1121b8f6a443a643e40951` |
| V4 PositionDescriptor | `0x9639443158e8c5efa35bd45287bf2effd3d8dc06` |
| V4 PositionManager | `0x58daec3116aae6d93017baaea7749052e8a04fa7` |
| V4 Quoter | `0x8dc178efb8111bb0973dd9d722ebeff267c98f94` |
| V4 StateView | `0xf3334192d15450cdd385c8b70e03f9a6bd9e673b` |
| V4 UniversalRouter | `0x8876789976decbfcbbbe364623c63652db8c0904` |
| Permit2 | `0x000000000022D473030F116dDEE9F6B43aC78BA3` |

These are venue infrastructure contracts, not the target for a limit-order
fill transaction.

### Curated Tokens

| Symbol | Decimals | Address |
| --- | ---: | --- |
| WETH | 18 | `0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73` |
| USDG | 6 | `0x5fc5360D0400a0Fd4f2af552ADD042D716F1d168` |
| VIRTUAL | 18 | `0xc6911796042b15d7Fa4F6CDe69e245DdCd3d9c31` |
| CASHCAT | 18 | `0x020bfC650A365f8BB26819deAAbF3E21291018b4` |
| WALLET | 18 | `0x0339f5459FC690aC85F1782e15782A151b4A9E1b` |
| SEEDCOIN | 18 | `0x58f693A30F124E59b125F7c7b837b0F6bbAF5a45` |
| JUGGERNAUT | 18 | `0xD7321801CAae694090694Ff55A9323139F043B88` |
| HOODRAT | 18 | `0x8e62F281f282686fCa6dCB39288069a93fC23F1c` |
| DIH | 18 | `0x17bb0C898254406b1Ea2e8E99B0C263e26c9E4a4` |
| KITSU | 18 | `0x8d4dFaaA4198b6486E0293Fec914C2B6a821D4DC` |
| WEN | 18 | `0xA80eb66b3E0CF66ccB46f8b8C9e7ff5803eEb820` |
| REPE | 18 | `0x5266eeafF092D6136AB63D18B975A60a0Cc0C8f7` |
| TENDIES | 18 | `0x45242320DBB855EeA8Fd36804C6487E10E97FCF9` |
| GME | 18 | `0x7e86381A763F0Ecca2bDF27C54eAC403ddD48123` |
| 4663 | 18 | `0xd4052415613B34Af236024B895574c467f65b6dD` |
| MARIAN | 18 | `0x01637b14B7378B99dE75A64d50656d98488D9a4d` |
| Index | 18 | `0x56910D4409F3a0C78C64DD8D0545FF0705389870` |
| VEX | 18 | `0x8Ff92566f2e81BDd68EDfAa8cde73942A723796b` |
| HOODIE | 18 | `0xC72c01AAB5f5678dc1d6f5C6d2B417d91D402Ba3` |
| WISHBONE | 18 | `0x77581054581B9c525E7dd7a0155DE43867532d03` |
| VLAD | 18 | `0x31BE8f7485e36928C9De86566c62da82d4B6BF81` |
| AEON | 18 | `0xd4c93eD1843606f92CccA078941f3d52A585982f` |

Native ETH uses 18 decimals and is represented as WETH inside routes. The
default user-facing list is native ETH, WETH, USDG, VIRTUAL, CASHCAT, and Index.

### Trusted Intermediates

The current preferred intermediate order is:

1. WETH
2. USDG
3. VIRTUAL
4. CASHCAT
5. HOODRAT
6. TENDIES
7. JUGGERNAUT
8. MARIAN
9. WALLET

Treat this as route-search guidance, not a guarantee of liquidity or safety.
Apply pool allowlists, depth checks, price-impact caps, and tax restrictions.

### RPC and Explorer

- Public RPC: `https://rpc.mainnet.chain.robinhood.com`
- Explorer: `https://robinhoodchain.blockscout.com`
- Recommended production RPC profile: sync-aware proxy on `127.0.0.1:8575`
- Direct local Nitro RPC: `127.0.0.1:8576`

The local Robinhood-RPC profile supports transaction forwarding to the official
sequencer. A configuration with `FORWARDING_TARGET=null` is read-only and
cannot broadcast operator fills. Use redundant read endpoints and verify the
chain ID before signing.

---

## Support

- Website: [switch.win](https://switch.win)
- Telegram: [@BrandonDavisR2R](https://t.me/BrandonDavisR2R),
  [@bttscott](https://t.me/bttscott), and
  [@shanebtt](https://t.me/shanebtt)

---

## License

MIT

---

Last updated: July 19, 2026.
