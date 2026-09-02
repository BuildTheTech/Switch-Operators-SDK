# Switch Operator Engine 1.1.0 - Docker/VPS setup

The protected headless image runs the same current-contract-only PulseChain
and Robinhood operator workers as Operator Engine `1.1.0`. It can run either
chain or both simultaneously.

The image contains protected, Node/V8-version-pinned proprietary worker
bytecode and small module loaders. It does not contain the Switch Operator
Engine TypeScript source tree, source maps, declarations, build scripts, or
Electron/Chromium. Prisma's machine-generated database client remains ordinary
runtime JavaScript because Prisma performs native dynamic imports that cannot
run from V8 bytecode; it contains database plumbing rather than Switch routing
source. Bytecode materially raises the reverse-engineering cost, but no
executable distributed to an untrusted machine can be made impossible to
analyze.

## Requirements

- An x86-64 Ubuntu/Debian VPS with Docker Engine and Docker Compose v2
- At least 4 vCPU, 8 GB RAM, and 20 GB free disk space recommended
- An operator wallet with the on-chain operator role on the current deployment
- Purchased adapter access where required
- Native gas currency in that wallet

## 1. Download the setup files

```bash
mkdir -p switch-operator-engine/secrets
cd switch-operator-engine
curl -fsSLO https://github.com/BuildTheTech/Switch-Operators-SDK/releases/download/operator-engine-v1.1.0/docker-compose.yml
curl -fsSL https://github.com/BuildTheTech/Switch-Operators-SDK/releases/download/operator-engine-v1.1.0/operator-engine.env.example -o .env
curl -fsSLO https://github.com/BuildTheTech/Switch-Operators-SDK/releases/download/operator-engine-v1.1.0/Switch-Operator-Engine-1.1.0-Docker-amd64.tar.gz
curl -fsSLO https://github.com/BuildTheTech/Switch-Operators-SDK/releases/download/operator-engine-v1.1.0/SHA256SUMS-1.1.0.txt
grep 'Switch-Operator-Engine-1.1.0-Docker-amd64.tar.gz$' SHA256SUMS-1.1.0.txt | sha256sum -c -
docker load -i Switch-Operator-Engine-1.1.0-Docker-amd64.tar.gz
rm Switch-Operator-Engine-1.1.0-Docker-amd64.tar.gz
chmod 700 secrets
```

Edit `.env`. `SWITCH_NETWORKS=pulsechain` runs PulseChain only. Use
`SWITCH_NETWORKS=robinhood` for Robinhood only or
`SWITCH_NETWORKS=pulsechain,robinhood` for both.

Leaving the RPC URL fields empty uses the curated public RPC pools built into
Operator Engine `1.1.0`. A private node running on the VPS host can be addressed
as `http://host.docker.internal:PORT`.

## 2. Create the keystore password secret

This password encrypts the operator signer at rest. Store a backup in a
password manager. Losing it means the encrypted keystore cannot be recovered.

```bash
read -rsp "Keystore password (12+ characters): " SWITCH_KEYSTORE_PASSWORD
printf '%s' "$SWITCH_KEYSTORE_PASSWORD" > secrets/operator_password
unset SWITCH_KEYSTORE_PASSWORD
echo
chmod 600 secrets/operator_password
```

The password is mounted into the container as a read-only Docker secret. It is
not placed in `.env`, Compose environment variables, image layers, or process
arguments.

## 3. Encrypt the operator signer

After loading the release image, run setup. The private-key prompt does not echo
and the key is encrypted before anything is written to the persistent volume.

```bash
docker compose run --rm operator setup --network pulsechain
```

For Robinhood, use `--network robinhood`. If the same wallet operates both
networks, one hidden entry can create both encrypted keystores:

```bash
docker compose run --rm operator setup --network all
```

Setup refuses to overwrite a keystore. `--force` is available only for an
intentional signer replacement.

## 4. Start and monitor

```bash
docker compose up -d
docker compose logs -f operator
```

Check the sanitized runtime status:

```bash
docker compose exec operator status
docker inspect --format '{{json .State.Health}}' switch-operator-engine
```

Stop or restart safely:

```bash
docker compose stop
docker compose restart operator
```

The service handles `SIGTERM`, stops both workers, uses bounded automatic
worker recovery, and reports unhealthy until every configured network reaches
the ready state.

## Security and operations

- The encrypted keystore and local order databases live only in the named
  `switch-operator-engine-data` volume.
- The container runs as unprivileged UID/GID `10001`, drops every Linux
  capability, uses a read-only root filesystem, and enables
  `no-new-privileges`.
- Never put a raw private key in `.env`, Compose YAML, a shell command, or an
  RPC URL.
- Protect `secrets/operator_password` and all VPS administrator access.
  Docker/root access can inspect a running process and must be treated as
  signer-level access.
- Only sanitized lifecycle, fill, warning, and error summaries reach stdout.
  Detailed routing internals and source paths remain suppressed.
- The runtime checks the on-chain operator role and adapter entitlements before
  launching. It executes only the current Limit Order and PLS Flow deployments
  embedded in Operator Engine `1.1.0`.
- Back up both the Docker volume and password secret. Keep them separately.

Create an encrypted-volume backup:

```bash
docker compose stop
docker run --rm -v switch-operator-engine-data:/data -v "$PWD":/backup alpine \
  tar -czf /backup/operator-data-backup.tgz -C /data .
docker compose start
```

Verify the loaded image version and platform:

```bash
docker run --rm switch-operator-engine:1.1.0 version
docker image inspect switch-operator-engine:1.1.0 \
  --format '{{.Os}}/{{.Architecture}} {{.Config.User}}'
```
