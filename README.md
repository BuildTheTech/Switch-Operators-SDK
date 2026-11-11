# Switch Operators SDK

Integration guides for operators (fillers) of Switch Limit Orders and GaslessSwitch swaps.

**Unreleased local documentation:** the new [GaslessSwitch V2 guide](GASLESS.md)
covers the dedicated gasless contracts, separate payout policy, tax support,
priority retry scheduling, and the [quote API's eligibility response](GASLESS.md#api-eligibility-response).
This work does not publish Operator Engine
1.1.5 or a Docker image. The public download instructions below remain on 1.1.4;
editing a guide or `.env` cannot add gasless support to that older binary.

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

Switch Operator Engine `1.1.4` is available as both a desktop application and
a protected headless Docker image for x86-64 Linux servers. The Docker image
can operate PulseChain, Robinhood Chain, or both networks simultaneously.

- [Docker/VPS setup guide](DOCKER.md)
- Image archive: `Switch-Operator-Engine-1.1.4-Docker-amd64.tar.gz`
- [Operator Engine 1.1.4 release and checksums](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.1.4)

Desktop downloads cover Windows x64, macOS Apple Silicon and Intel, and Linux
x86-64 (AppImage/Debian). Version 1.1.4 updates Electron to 44.10.2, bundled
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

[Version 1.1.3 remains available for rollback](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.1.3).
Back up settings, encrypted wallet storage and databases before changing versions;
never run two versions against the same operator wallet simultaneously.

The image contains version-pinned protected runtime bytecode, not the Operator
Engine TypeScript source tree, source maps, declarations, or build scripts.

## Shared rules

- Include `network=pulsechain` or `network=robinhood` on every orderbook and
  config request.
- Fetch `/limit-orders/config` at startup. Gasless-aware public operators may
  fill the current `limitOrderContract` and separately validated `gaslessContract`;
  accept native-flow makers only from the current `plsFlowContract`.
  Historical arrays are for cancellation, indexing, and trusted drain tooling.
- After applying that allowlist, fill, simulate, and check nonces against each
  order's own `limitOrderContract`.
- Read each target's fee and protection configuration on-chain. Ordinary limit
  orders use a 30 bps fee, 5% route-surplus cap, and a 95%-of-market direct-fill
  floor. GaslessSwitch V2 uses a 10 bps fee, a 5 bps total operator payout cap
  including gas, and route fills only. Do not apply LO profit thresholds to it.
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
