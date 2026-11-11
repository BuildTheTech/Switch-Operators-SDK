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

Switch Operator Engine `1.1.8` is available as both a desktop application and
a protected headless Docker image for x86-64 Linux servers. The Docker image
can operate PulseChain, Robinhood Chain, or both networks simultaneously.

- [Docker/VPS setup guide](DOCKER.md)
- Image archive: `Switch-Operator-Engine-1.1.8-Docker-amd64.tar.gz`
- [Operator Engine 1.1.8 release and checksums](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.1.8)

Desktop downloads cover Windows x64, macOS Apple Silicon and Intel, and Linux
x86-64 (AppImage/Debian). Version 1.1.8 removes duplicate pool searches
for PulseChain gasless input payouts, reserves tax-check capacity for gasless
orders on public RPCs, and suppresses stale quote/retry messages after an order
closes. Fixed OTC fills retain amount-specific sizing. Exact settlement, signed
minimum output and gas checks remain required.
It retains the corrected JoltOTC integration and GaslessSwitch V2 on both chains,
separate gasless profit settings in the desktop app, independent gasless discovery,
priority retries, tax-aware settlement checks and visible gasless execution activity.
The 0.05% operator payout cap includes gas; ordinary limit-order profit settings
remain separate. It retains the patched Electron 44.10.2 / Node.js 24.22.2 runtimes.

See the [1.1.8 changes and downloads](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.1.8)
and [Docker upgrade instructions](DOCKER.md#upgrade-an-existing-installation-and-retain-rollback).
Public RPCs, prices and competition can still prevent a fill; a successful API
eligibility preview is not a fill guarantee.

Windows supports signed in-app updates. Starting with 1.1.1, macOS and Linux
also check the signed feed and notify users about releases, with a link to
official downloads for manual installation. Older Mac/Linux clients, including
1.0.2, need one manual upgrade to gain notifications. Quit the old app before
installing, verify the download checksum, and retain existing application data.
macOS builds are ad-hoc signed, not Apple-notarized.

Version 1.1.8 is the current stable release. Version 1.1.7 remains available for rollback; withdrawn 1.1.5 stays unpublished.
Back up settings, encrypted wallet storage and databases before changing versions;
never run two versions against the same operator wallet simultaneously.

The image contains version-pinned protected runtime bytecode, not the Operator
Engine TypeScript source tree, source maps, declarations, or build scripts.

## Shared rules

- Include `network=pulsechain` or `network=robinhood` on every orderbook and
  config request.
- Fetch `/limit-orders/config` at startup. Public operators must quote and fill
  ordinary orders only on the response's current `limitOrderContract` and `plsFlowContract`.
  Operator Engine 1.1.7 additionally pins the GaslessSwitch contracts listed in its
  [release guide](OPERATOR_ENGINE_1.1.7.md); their fee and payout rules differ.
  Historical arrays are for cancellation, indexing, and trusted drain tooling.
- After applying that allowlist, fill, simulate, and check nonces against each
  order's own `limitOrderContract`.
- Read the current fee and protection configuration on-chain. Both networks
  use a 30 bps fee for ordinary limit orders, a 5% route-surplus cap, and a
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
