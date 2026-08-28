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

Switch Operator Engine `1.0.4` is available as both a desktop application and
a protected headless Docker image for x86-64 Linux servers. The Docker image
can operate PulseChain, Robinhood Chain, or both networks simultaneously.

- [Docker/VPS setup guide](DOCKER.md)
- Image archive: `Switch-Operator-Engine-1.0.4-Docker-amd64.tar.gz`
- [Operator Engine 1.0.4 release](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.0.4)

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
