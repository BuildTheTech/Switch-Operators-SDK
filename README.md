# Switch Operators SDK

Integration guides for operators (fillers) of Switch Limit Orders.

Operators monitor the Switch orderbook for signed limit orders, evaluate
profitability, and fill orders on-chain. Choose the guide for the target
network:

| Network | Chain ID | Native currency | Operator guide |
| --- | ---: | --- | --- |
| PulseChain | `369` | PLS | [PulseChain operator guide](PULSECHAIN.md) |
| Robinhood Chain | `4663` | ETH | [Robinhood operator guide](ROBINHOOD.md) |

Both guides use the same structure and execution model. Their contract
addresses, native currency, routing venues, token data, fee configuration, and
RPC requirements are chain-specific.

## Operator Engine

Switch Operator Engine `1.1.5` is available as both a desktop application and
a protected headless Docker image for x86-64 Linux servers. The Docker image
can operate PulseChain, Robinhood Chain, or both networks simultaneously.

- [Docker/VPS setup guide](DOCKER.md)
- Image archive: `Switch-Operator-Engine-1.1.5-Docker-amd64.tar.gz`
- [Operator Engine 1.1.5 release and checksums](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.1.5)

Desktop downloads cover Windows x64, macOS Apple Silicon and Intel, and Linux
x86-64 (AppImage/Debian). Version 1.1.5 retains patched Electron 44.10.2, bundled
Node.js to 24.22.2, and the headless Node.js base to 24.22.2, including the
October Brotli security fix. It patches dependencies and enables the backend
CPU serialization optimization for PulseChain operator quotes. Quote and
transaction bytes, route coverage, RPC limits, and 1.1.3 payout checks are
preserved. End-to-end quote time still depends on the pair, routing work, and RPCs.

Windows supports signed in-app updates. Starting with 1.1.1, macOS and Linux
also check the signed feed and notify users about releases, with a link to
official downloads for manual installation. Older Mac/Linux clients, including
1.0.2, need one manual upgrade to gain notifications. Quit the old app before
installing, verify the download checksum, and retain existing application data.
macOS builds are ad-hoc signed, not Apple-notarized.

[Version 1.1.4 remains available for rollback](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.1.4).
Back up settings, encrypted wallet storage and databases before changing versions;
never run two versions against the same operator wallet simultaneously.

Version 1.1.5 adds dedicated GaslessSwitch orders on PulseChain and Robinhood. Gasless swaps use a separate minimum net profit setting (default 0 bps, requiring strictly positive earnings after gas) and a bounded priority retry queue while ordinary limit-order sweeps continue. The total gasless operator payout remains capped on-chain at 0.05%, including gas. Ordinary limit-order profit settings remain separate.

The image contains version-pinned protected runtime bytecode, not the Operator
Engine TypeScript source tree, source maps, declarations, or build scripts.

## Shared rules

- Include `network=pulsechain` or `network=robinhood` on every orderbook and
  config request.
- Fetch `/limit-orders/config` at startup. Public operators must quote and fill
  only the response's current `limitOrderContract`, `plsFlowContract`, and recognized `gaslessContract` when present.
  Historical arrays are for cancellation, indexing, and trusted drain tooling.
- After applying that allowlist, fill, simulate, and check nonces against each
  order's own `limitOrderContract`.
- Read the current fee and protection configuration on-chain. Ordinary limit orders on both networks currently use a 30 bps fee, a 5% route-surplus cap, and a
  fail-closed 95%-of-market direct-fill floor.
- Treat all route and tax information as time-sensitive and simulate the exact
  transaction immediately before broadcast.

## SDK

The network-aware helper package is published as
[`@switch-win/sdk`](https://www.npmjs.com/package/@switch-win/sdk):

```bash
npm install @switch-win/sdk@^1.2.8 ethers
```

Use `@switch-win/sdk` version `1.2.8` or newer for the protected deployment
addresses, current ABIs, fee constants, and Robinhood adapter metadata. This repository contains the operator
integration documentation; it is not a separate npm package.

## Support

- **Website:** [switch.win](https://switch.win)
- **Telegram:** [@BrandonDavisR2R](https://t.me/BrandonDavisR2R),
  [@bttscott](https://t.me/bttscott), and
  [@shanebtt](https://t.me/shanebtt)

## License

MIT

## Gasless swap orders

GaslessSwitch uses the same signed order tuple with a distinct EIP-712 domain, **GaslessSwitch / 1**, bound to chain and contract. Users approve GaslessSwitch itself. Read its fee and operator cap on-chain: a 0.1% protocol fee and a maximum **total** operator payout of 0.05% of actual output after the protocol fee, including gas. Surplus beyond that cap goes to the signed recipient; an operator cannot change the signed minimum or collect the full slippage tolerance.

- PulseChain: `0xC28C70527740A85E98f7C8f800694f05c43814C5`
- Robinhood: `0xC9daFAA2ef5482Eab0014fAc7c44E7F8C9005A3F`

Identify these orders by their contract, not a short deadline or maker address. Revisit them while valid instead of waiting for a complete long-order sweep. Engine 1.1.5 allows at most two gasless evaluations at once, uses eight seconds between completed attempts during long cycles, and retains transaction serialization, pending-fill deduplication and full-gas checks. Quote/RPC time and the ordinary cycle interval are additional. Stop routing filled/cancelled or expired signatures. Do not submit a loss-making fill simply because its configured minimum is zero.
