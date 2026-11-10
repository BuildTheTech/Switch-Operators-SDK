# Operator Engine 1.1.7

This patch fixes JoltOTC routing on PulseChain. Version 1.1.6 contained the new
adapter address but still read the retired orderbook format. The engine now uses
the new JoltOTC book and explicit order ID/payment option calldata in slot 16.
It supports partial, full-only and multi-payment orders with fee rounding and
minimum-fill checks, and refreshes through the shared public-RPC failover pool.

- Orderbook: `0x4368b4dC185a17E77cCAcc389e267cC3bBD9C820`
- Adapter (slot 16): `0x2A1DE1578489353420FF0688Cc7782190fcC7e19`
- The old JoltOTC book is retired; no second Jolt adapter slot is added.
- Existing adapter access requirements still apply. GaslessSwitch V2 functionality
  and its 0.05% total operator payout cap are retained.

Read-only checks found six active orders and eight payment-option quotes matching
the deployed adapter. Tests cover cumulative fees, partial-fill minimums and dust,
full-only orders, mixed liquidity, transaction data and public-RPC failover.

## Gasless support retained from 1.1.6

GaslessSwitch V2 is supported by the desktop and headless engine on PulseChain
and Robinhood. The engine discovers gasless requests every five seconds in a
separate API poll, then retries during ordinary order scans while the order can
still execute. Activity shows detection, quote attempts, gas checks, submission,
settlement and expiry.

| Network | GaslessSwitch V2 |
| --- | --- |
| PulseChain (369) | `0xB5bd12E009D625d8b122d0Ff9EdeD8bA2cEdeaeB` |
| Robinhood (4663) | `0xc39cB531f71059fd427a63bF8a2f4b822D6B9b02` |

The contract charges a **0.1% protocol fee** and caps the **total operator payout
at 0.05%, including gas**. The engine accepts smaller payouts when they cover the
full padded gas budget and leave positive earnings. Desktop users can adjust
the separate gasless minimum net-profit setting (0–5 bps); its default is 0 bps.
Raising it can make gasless orders unprofitable. Ordinary limit-order settings
remain independent. The Docker worker uses the same zero-bps gasless default.

Settlement uses the original signed minimum and fee mode, with actual tax-aware
recipient and operator receipts checked before broadcast. The release also
includes public PulseChain RPC priority improvements, the SwitchX repeated-pool
fix, preserved quoted pool calldata, and the replacement JoltOTC book/adapter.

The API's `gaslessEligibility` estimates whether a **new** order is suitable for
gasless execution. Operators still verify the existing signed order locally
against their own adapter access, current gas and payout. A fresh API quote never
replaces the user's signed minimum. Eligibility is not a guaranteed fill.

## Downloads and upgrades

Download from this release and verify the file against `SHA256SUMS-1.1.7.txt`:

- **Windows x64:** `Switch-Operator-Engine-1.1.7-x64.exe`. Existing clients can
  use the signed in-app update after the feed is updated.
- **macOS Apple Silicon:** `arm64.dmg` or `arm64.zip`.
- **macOS Intel:** `x64.dmg` or `x64.zip`.
- **Linux x86-64:** `x86_64.AppImage` or `amd64.deb`.
- **Docker Linux amd64:** `Switch-Operator-Engine-1.1.7-Docker-amd64.tar.gz`.

Quit the old application before installing. Keep its application data, encrypted
signer, settings and databases. Do not run both versions with the same wallet.
macOS and Linux update notifications link to manual downloads; clients older
than 1.1.1 need a manual upgrade to gain these notifications. macOS builds remain
ad-hoc signed, not Apple Developer ID signed or notarized.

### Existing Docker installations

Run in your existing Compose directory. Keep the same `.env`, password secret,
project name and persistent volume. Download and verify the image before stopping
the old container:

```bash
curl -fsSLO https://github.com/BuildTheTech/Switch-Operators-SDK/releases/download/operator-engine-v1.1.7/Switch-Operator-Engine-1.1.7-Docker-amd64.tar.gz
curl -fsSLO https://github.com/BuildTheTech/Switch-Operators-SDK/releases/download/operator-engine-v1.1.7/SHA256SUMS-1.1.7.txt
grep 'Switch-Operator-Engine-1.1.7-Docker-amd64.tar.gz$' SHA256SUMS-1.1.7.txt | sha256sum -c -
docker load -i Switch-Operator-Engine-1.1.7-Docker-amd64.tar.gz
docker compose stop operator
```

Set the existing Compose service's `image` to `switch-operator-engine:1.1.7`,
retaining `pull_policy: never`, then run:

```bash
docker compose up -d operator
docker compose exec operator version
docker compose logs --tail 100 operator
```

No signer setup or database reset is needed. Never use `docker compose down -v`.
For a new installation, follow [DOCKER.md](https://github.com/BuildTheTech/Switch-Operators-SDK/blob/main/DOCKER.md).

The verified [1.1.4 release](https://github.com/BuildTheTech/Switch-Operators-SDK/releases/tag/operator-engine-v1.1.4)
remains available for rollback. Version 1.1.5 remains withdrawn. This release
publishes the Operator Engine; the new gasless npm SDK package is not published.
