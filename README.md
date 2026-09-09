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

Switch Operator Engine `1.1.1` is available as both a desktop application and
a protected headless Docker image for x86-64 Linux servers. The Docker image
can operate PulseChain, Robinhood Chain, or both networks simultaneously.

- [Docker/VPS setup guide](DOCKER.md)
- Image archive: `Switch-Operator-Engine-1.1.1-Docker-amd64.tar.gz`
- [Operator Engine 1.1.1 release and checksums](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.1.1)

Desktop downloads cover Windows x64, macOS Apple Silicon and Intel, and Linux
x86-64 (AppImage/Debian). Version 1.1.1 improves Robinhood public-RPC handling,
retains V4 routing and exact execution verification, and removes detailed
diagnostic logging from public builds.

Windows supports signed in-app updates. Starting with 1.1.1, macOS and Linux
also check the signed feed and notify users about releases, with a link to
official downloads for manual installation. Older Mac/Linux clients, including
1.0.2, need one manual upgrade to gain notifications. Quit the old app before
installing, verify the download checksum, and retain existing application data.
macOS builds are ad-hoc signed, not Apple-notarized.

[Version 1.1.0 remains available for rollback](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.1.0).
Back up settings, encrypted wallet storage and databases before changing versions;
never run two versions against the same operator wallet simultaneously.

The image contains version-pinned protected runtime bytecode, not the Operator
Engine TypeScript source tree, source maps, declarations, or build scripts.

## Shared rules

- Include `network=pulsechain` or `network=robinhood` on every orderbook and
  config request.
- Fetch `/limit-orders/config` at startup. Public operators must quote and fill
  only the response's current `limitOrderContract` and `plsFlowContract`.
  Historical arrays are for cancellation, indexing, and trusted drain tooling.
- After applying that allowlist, fill, simulate, and check nonces against each
  order's own `limitOrderContract`.
- Read the current fee and protection configuration on-chain. Both networks
  currently use a 30 bps limit-order fee, a 5% route-surplus cap, and a
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
