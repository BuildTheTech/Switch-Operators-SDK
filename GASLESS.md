# GaslessSwitch V2 operators

**Unreleased integration guide.** The public Operator Engine and Docker
downloads remain version 1.1.4. This document describes a gasless-aware worker
or a custom integration; it does not claim that 1.1.4 implements this flow.
Operator Engine 1.1.5 remains unpublished. No environment setting can patch
an older binary into a gasless-aware one.

Gasless swaps share the off-chain `/limit-orders` orderbook and the signed
`LimitOrder` tuple with ordinary limit orders. They have a separate contract,
domain, fee, payout cap, approval target and urgency. Classify by the validated
settlement address, never the token symbol, order label or deadline alone.

## Contracts and startup checks

Public config endpoints verified November 9, 2026:

| Network | Chain | Current GaslessSwitch V2 | Router |
| --- | ---: | --- | --- |
| PulseChain | 369 | `0xB5bd12E009D625d8b122d0Ff9EdeD8bA2cEdeaeB` | `0x2dFc8B6e13fF7F04e37Ef97006084805D65a6F19` |
| Robinhood | 4663 | `0xc39cB531f71059fd427a63bF8a2f4b822D6B9b02` | `0x8b2bBdF41C1486b3482bD0e9603d72f012EE8599` |

Fetch `/limit-orders/config?network=pulsechain` or `network=robinhood`. Check
`gaslessContract`, `gaslessApprovalSpender`, `gaslessSettlementVersion=2`,
`gaslessTaxTokensSupported=true`, `gaslessProtocolFeeBps=10`,
`gaslessMaxOperatorPayoutBps=5`, and `gaslessEip712Domain`.
The name is **GaslessSwitch**, version **2**, with that network's chain ID and
gasless verifying contract. Do not reuse the ordinary LO domain.

Verify on-chain:

- Chain ID, recent block, contract bytecode, `settlementVersion()==2`.
- `SWITCH_ROUTER()` and `WNATIVE()` match the trusted deployment.
- `getFee()==10`, `MAX_OPERATOR_EXCESS_BPS()==5`, and `paused()==false`.
- `operatorGateEnabled()==true`, and your wallet has
  `hasRole(keccak256("OPERATOR_ROLE"), operator)==true` on GaslessSwitch itself.
- Router `FEE_EXEMPT(gaslessContract)==true` and each route adapter is approved.

GaslessSwitch is attached to the same operator-access administration pipeline.
Holding a role on an ordinary LO alone is not proof of permission to fill this
contract. If registration has not propagated, report it rather than sending a
transaction that will revert.

The public fill allowlist is the current ordinary `limitOrderContract` plus
the separately validated current `gaslessContract`. Native-flow makers must
come from the current `plsFlowContract`. Historical `all*` arrays support
indexing, cancellation and authorized drain work, not automatic public fills.
Gasless V1 addresses (`0xC28C70527740A85E98f7C8f800694f05c43814C5` on PulseChain,
`0xC9daFAA2ef5482Eab0014fAc7c44E7F8C9005A3F` on Robinhood) use a different
domain/version and settlement behavior. Do not send new V2 fills to them.

The adjacent Switch-SDK checkout includes `@switch-win/sdk/gasless` and
`@switch-win/sdk/gasless/operators` as **unreleased** helpers. They provide
config/domain validation, `classifySwitchOrder`, `selectGaslessPriorityOrders`,
`assertGaslessOperator`, `encodeGaslessFill`, payout cap math and gas budgeting.
The complete ABI is `abi/GaslessSwitchV2ABI.json`. Do not assume the currently
published npm package already exports those helpers.

## Fee modes, approvals, and execution

The maker always approves **GaslessSwitch V2**, including output-fee orders.
V2 executes signed routes through its authorized execution library; the maker
does not need a second Router allowance. The ordinary LO approval table in the
network guides applies only to ordinary LOs.

| Signed `feeOnOutput` | Required `excessOnInput` | Routed amount | Payout asset |
| --- | --- | --- | --- |
| `true` | `false` | Full signed input | Output token |
| `false` | `true` | At most actual received input minus input protocol fee | Input token |

The flags cannot be chosen independently for gasless. Mismatched flags revert.
There is no `directFillOrder` entry point and no PLSFlow funding path.
Native input must first be wrapped by the user. Native output uses WPLS/WETH
in the signed tuple with `unwrapOutput=true`.

The fill ABI is:

```solidity
function fillOrder(
    LimitOrder calldata order,
    bytes calldata signature,
    RouteAllocation[] calldata routes,
    bool excessOnInput
) external returns (uint256 makerOutput, uint256 operatorPayout, uint256 capBase);
```

Build the order from **all original signed fields**, including recipient,
partner, fee side, unwrap flag, nonce, and deadline. Changing the amount or
minimum to match a new quote invalidates the signature. Send the fill to the
order's validated `limitOrderContract`; a quote's `selectedTx.to` identifies
the Router and is not a fill transaction target.

Before filling check the nonce, deadline, real maker balance, allowance and
`canFillOrder(order, signature)` on GaslessSwitch. `canFillOrder` does not test
liquidity, gas, route metadata, taxes, or operator profitability. Simulate the
exact final fill calldata immediately before broadcast.

## Total payout and profitability

The protocol charges 0.1%. The operator's **total** nominal payout is capped
at 0.05% of the realized after-fee base, including gas reimbursement. It is
not a 0.05% profit after gas and not a percentage of the excess itself.
Do not apply an ordinary LO 50 bps (0.5%) net-profit floor to gasless work.

Output-fee fill:

```text
fee       = floor(actual settlement output * 10 / 10000)
afterFee  = actual settlement output - fee
surplus   = afterFee - signed minimum
payout    = min(surplus, floor(afterFee * 5 / 10000))
recipient = actual balance received after the final token transfer
```

Input-fee fill:

```text
received  = actual maker -> settlement input balance delta
fee       = floor(received * 10 / 10000)
available = received - fee
surplus   = available - sum(route.amountIn)
payout    = min(surplus, floor(available * 5 / 10000))
refund    = surplus - payout   # returned to the maker
```

The signed minimum is enforced on the actual recipient output. Slippage gives
room for price changes, not an enlarged operator cap. If only 0.01% is left,
the payout is automatically smaller. Fill it only if that smaller actual
payout covers the full submitted gas budget and leaves positive earnings.
Do not try to force the full cap when doing so would miss the maker's minimum.

Value **actual after-transfer-tax operator receipts** in native currency
through an executable conversion route. Missing valuation is a failed check,
not permission to substitute an optimistic USD price. Quote the whole payout
at its actual size; don't extrapolate from a one-token price in a shallow pool.

Budget the gas limit you will actually submit, not just the unpadded estimate.
As of November 10, 2026, both quote APIs apply this conservative recommendation:

```text
gasLimit = ceil(estimatedGas * 1.30)
gasPrice = max(API quote gas price, preview price, ceil(fresh raw gas price * 1.30))
budget   = ceil(gasLimit * gasPrice * 1.10)
reserve  = budget * 2
stressedPayout = floor(actual payout in native * 0.90)
eligible only when stressedPayout > reserve
```

The API also checks real registered-operator native funds against `reserve`,
at least 50 bps of user-selected slippage, and preview freshness. It returns
`gaslessEligibility` with native economics, reason codes and the same
`eligible` flag as `gaslessExecution`. A ready simulation can be ineligible.
This recommendation does not alter the contract cap or charge the reserve.
`operatorLivenessVerified: false` explicitly distinguishes funded registration
from an online worker. The public engine binary has not been updated or
published by this docs change; every worker must still simulate the exact
signed order and independently check its own profitability before filling.

The API price is already boosted; do not multiply it again. Your own additional
minimum incentive can be stricter, but cannot make the contract pay more.
Keep gasless and ordinary LO profit settings separate. Persist pending wallet
transactions, serialize wallet nonces, and recalculate profitability before a
gas-price replacement. A replacement can make a previously profitable order
unprofitable. Never leave a known losing fill queued just to complete the order.

`fillOrder`'s returned `operatorPayout` and the `OrderFilled.fillerProfit`
event are nominal amounts before payout transfer tax. Trace the actual balance
delta when a tax can apply. The receipt's actual gas cost is known only after
settlement; distinguish nominal payout, actual token receipt, gas budget,
actual gas spent and final net earnings in logs and reporting.

## API eligibility response

Both hosted APIs now return `gaslessEligibility` on successful swap quotes;
the same summary appears under `gaslessExecution.eligibility`. Request a real
check with `gasless=true`, `fee=10`, the correct fee side, `network`, actual
maker/recipient, raw amount and user slippage (basis points; 50 means 0.5%).
The unpublished `@switch-win/sdk/gasless` client handles those parameters and
exposes `decision.eligibility` for both accepted and rejected quotes.

| Field | Use |
| --- | --- |
| `status` | `eligible`, `ineligible`, `unavailable` or `not_checked`; only eligible can pass a frontend preflight. |
| `eligible` / `recommendation` | Conservative server recommendation, mirrored by the preview flag. |
| `reasonCode` / `reason` | Explain the fallback instead of treating every rejection as a stale route. |
| `economics.operatorPayoutNative` | Measured payout valued through executable routing, formatted as PLS/ETH. |
| `economics.estimatedGasCostNative` | Gas estimate at the checked gas price. |
| `economics.gasBudgetNative` | Padded execution budget. |
| `economics.conservativeGasBudgetNative` | Twice that padded budget; payout must exceed it after a 10% haircut. |
| `economics.requiredOperatorPayoutNative` | Minimum native payout needed to pass the strict integer comparison. |
| `operator.nativeBalanceWei` / `gasAvailable` | Checked registered candidate's native funds, not a promise about your wallet. |
| `checkedAt` / `expiresAt` | Millisecond timestamps; refresh an expired check. |
| `operatorLivenessVerified` | Currently false; funding and registration do not prove an online worker. |

Raw wei fields remain decimal strings; `netAfterGasBudgetNativeWei` may be
negative. Missing economics are unknown, not zero-cost execution. A
`gaslessExecution.status` of ready means the simulation completed and can
still have `eligible: false`.

`GAS_NOT_COVERED` and `INSUFFICIENT_SAFETY_MARGIN` normally mean a larger swap
or Regular. `SLIPPAGE_BUFFER_TOO_SMALL` requires an explicit user tolerance
change or Regular. `STALE_PREVIEW` needs refresh. `EXECUTION_UNVERIFIED`,
`PAYOUT_UNVALUED`, `NO_APPROVED_OPERATOR`, and `OPERATOR_GAS_UNAVAILABLE` can
be retried with backoff. `NOT_REQUESTED` means no gasless check ran. Never
weaken the fee/cap/gas checks to turn a rejection into an acceptance.

For an existing signed order, always simulate its **original signed minimum**
with your wallet, allowance, gas budget and current executable routes. A new
frontend quote can have a different minimum. Its eligibility result is neither
fill authorization nor a reason by itself to discard a still-live signed order
that your exact simulation can profitably fill. The client rechecks the
conservative reserve against current gas before allowing a new signature.
This documentation does not update or publish the 1.1.4 worker binary.

## Quotes and tax-token support

The hosted quote endpoint can provide a checked frontend preview with
`gasless=true`, `fee=10`, the selected fee mode, actual maker/recipient and
slippage. The preview executes the deployed settlement at a pinned block and
measures settlement outputs and gas. It is a preflight estimate, not a fill
authorization or proof of the real maker's allowance/signature. A custom
operator still simulates the **existing signed order's minimum**, which may
differ from a fresh quote's newly computed minimum.

Do not broadcast the API's regular `goSwitch` transaction or blindly copy its
nominal full-input routes into a gasless input-fee fill. Quote routes for the
actual executable spend. With input fees, reserve protocol fee and a capped
payout from measured input, then quote that spend. With output fees route the
full input. A low-level encoder verifies bounds; it does not make arbitrary
scaled route bytes executable.

Preserve every adapter's fee and `bytes data` metadata, including explicit OTC
order IDs and state-sensitive route data. Requote and rebuild after pool/order
state changes; arbitrary rescaling of calldata can invalidate OTC and other
specialized routes. Quote all legs consistently at the same recent block.

Tax mode selection matches regular swaps and LOs:

- Sell-tax input only: output fee / output payout; direct maker-to-first-pool
  execution avoids an extra input custody transfer.
- Buy-tax output: input fee / input payout; route output directly to the
  recipient where possible.
- Both directions taxed: input fee, measured received input, tax-compatible
  routing, and exact simulation including the payout transfer.
- No directional tax: use the frontend's known fee-token priority hierarchy.

Support tax tokens when their route and actual transfers pass. VOUCH, SWITCH,
and any other token must be checked by address and live behavior; symbol names
do not establish tax safety. Never apply reported tax rates a second time to
an adapter quote that already includes tax. Use each network's actual tax-safe
adapter registry; V3/V4/OTC coverage is not permission to use a venue for every
tax token. If a token's mutable behavior cannot be verified, do not broadcast.

## Priority discovery and retries

Gasless frontend orders normally live for three minutes; the hosted submission
policy accepts a live deadline up to ten minutes. Do not wait for a complete
ordinary LO cycle before discovering, quoting, or retrying them.

Use a dedicated bounded queue alongside ordinary limit-order work:

1. Poll active orders frequently (for example every five seconds), page
   results, validate the current contracts, deduplicate by contract + maker +
   nonce, and enqueue gasless orders by earliest deadline. The list endpoint
   now supports `orderType=GASLESS`, `LIMIT_ORDER`, or `ALL` on both networks;
   absent/ALL retains the shared book. For example,
   `/limit-orders?network=pulsechain&status=ACTIVE&orderType=GASLESS&limit=500`.
   Still paginate and apply the current-contract allowlist: the filter includes
   historical gasless contracts. It does not implement retry scheduling for you.
2. Reserve a routing/RPC slot for this queue, and bound normal LO concurrency
   so priority work can actually obtain one. Priority over an already saturated
   public-RPC queue alone is insufficient. Continue ordinary LO work fairly.
3. Coalesce simultaneous discovery and timer events. One evaluation per order
   at a time; retain its queue entry through a transient failure. Recheck all
   queued live orders, not just one leader per token pair.
4. Retry an unprofitable price, gas shortfall, stale simulation, `InsufficientOutput`,
   timeout, or transient public-RPC failure within the order window. A bounded
   two-to-ten-second backoff with jitter is a reasonable starting point; respect
   endpoint limits and total request budgets. Each retry needs a fresh pinned
   quote and exact simulation. Do not recycle a partially completed quote.
5. Abort expired/filled/cancelled work, check the chain's deadline and nonce
   before broadcast, and reserve time for simulation and confirmation. Do not
   defer a three-minute order behind a multi-minute full LO scan.
6. Serialize submission across both queues for the same wallet. Persist pending
   transactions so restart/replacement cannot duplicate a fill or misuse a nonce.

Use pagination responsibly: the API limits page size to 500 and its public
read rate is finite. Polling only the first page can miss live orders on a busy
book. Non-snapshot offset pages can move during inserts; deduplicate overlapping
pages and revisit recent pages. Back off on 429 responses. Large integrations
should coordinate a filtered/cursor-based feed rather than assuming this
shared endpoint is an unlimited low-latency stream.

Public RPCs are supported, but need bounded parallelism, pinned-block
consistency, a health-checked endpoint pool, realistic per-call deadlines and
backoff. Fall back after transport/rate-limit failures while rebuilding the
whole pinned quote if necessary. Do not classify transport failure as an
empty market, silently omit a venue, extend work past order expiry, or remove
tax checks to meet a latency target. Missing audit/valuation data means retry,
not broadcast. A private RPC is an option rather than a requirement.

Log useful lifecycle events with the contract, maker/nonce, pair and remaining
seconds: detected, queued, quoting, quote complete, checking execution gas,
retry scheduled with reason, submitted, filled, expired, or cancelled. Distinguish
gas shortfall from an LO profit-floor rejection. Rate-limit repeat messages;
do not expose raw keys, signed payloads, private RPC credentials, or giant
routing dumps in user logs.

## Docker and desktop rollout

The currently documented public release remains **1.1.4**. Do not advertise a
new download, set an imaginary gasless environment variable on that image, or
publish the 1.1.5 draft as part of these documentation changes.

When an approved gasless-aware release is available, Docker operators need its
new runtime image, not just updated Compose settings. Verify the release
checksums, retain the existing encrypted keystore/password secret/data volume,
stop the old worker, load the approved image, update its tag, and recreate the
service. Do not run old and new workers against the same wallet concurrently.
Desktop operators install the approved build and retain their application data.
Verify the gasless contract, role, separate payout policy, RPC health and
detection/retry logs after startup. Existing LO settings continue separately.

See [DOCKER.md](DOCKER.md) for the current released setup and rollback process,
[PULSECHAIN.md](PULSECHAIN.md) and [ROBINHOOD.md](ROBINHOOD.md) for ordinary LO
details, and the Switch-SDK repository's `GASLESS.md` for the maker/frontend flow.
