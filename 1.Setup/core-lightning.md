---
title: Bitcoin Core + Core Lightning
parent: Setup
nav_order: 1
---

# Bitcoin Core + Core Lightning

The harness requires two binaries on `PATH`: `bitcoind` (Bitcoin Core) and `lightningd` (Core Lightning). The top-level `./install.sh` handles both automatically on WSL2/Linux (Ubuntu 22.04+).

This page explains **what each piece is for**, **how the installer obtains it**, and **how to install them manually** if you prefer not to run `./install.sh`.

---

## What these binaries do in this project

**Bitcoin Core (`bitcoind`)** is the full-node implementation of the Bitcoin protocol. In this project it runs in `regtest` mode — an isolated local blockchain with no connection to mainnet, no real funds, and manual block mining. All Lightning channels are anchored on this regtest chain. The MCP tools prefixed `btc_*` (e.g. `btc_getblockchaininfo`, `btc_sendtoaddress`) wrap `bitcoin-cli` calls against this node.

**Core Lightning (`lightningd`)** is Blockstream's implementation of the Lightning Network protocol. In this project every Lightning node is a separate `lightningd` process with its own data directory under `runtime/lightning/node-N/` and its own on-chain wallet. The MCP tools prefixed `ln_*` (e.g. `ln_getinfo`, `ln_openchannel`, `ln_pay`) wrap `lightning-cli` calls against a specific node.

Both daemons are POSIX-only. Core Lightning in particular depends on Unix domain sockets, fork semantics, and POSIX plugin process management, which is why the harness runs on Linux/WSL2 and not on native Windows.

---

## Automatic installation

`./install.sh` (at the repository root) downloads the official upstream binaries for both daemons and places `bitcoind` / `bitcoin-cli` / `lightningd` / `lightning-cli` on `PATH`:

```bash
./install.sh
```

It installs **pre-built binaries**, not source builds:

- **Bitcoin Core** — the official `bitcoin-core-<version>-x86_64-linux-gnu.tar.gz` release from `bitcoincore.org`, SHA-256 verified.
- **Core Lightning** — the official per-Ubuntu-version release tarball (`clightning-<version>-Ubuntu-<os-version>-amd64.tar.xz`) from the [ElementsProject/lightning](https://github.com/ElementsProject/lightning) GitHub releases page.

Binaries were chosen over source builds because (a) the Ubuntu-pinned CLN release is reproducible across machines, (b) build-from-source requires Rust + gRPC + protobuf + full autotools toolchain, and (c) the SHA-256-verified tarball is easier to audit than a long build log. See `ln-ai-network/scripts/README_INSTALL.md` for the complete installer breakdown.

After installation:

```bash
bitcoind --version
lightningd --version
```

should both print a version string.

---

## Manual installation

If `./install.sh` does not cover your platform (e.g. you are on a non-apt distribution), install the binaries manually.

### Bitcoin Core

Download the official release tarball from [bitcoincore.org/bin](https://bitcoincore.org/bin/), verify its SHA-256, and extract `bitcoind` + `bitcoin-cli` onto `PATH`:

```bash
wget https://bitcoincore.org/bin/bitcoin-core-<VERSION>/bitcoin-<VERSION>-x86_64-linux-gnu.tar.gz
sha256sum bitcoin-<VERSION>-x86_64-linux-gnu.tar.gz   # cross-check with SHA256SUMS on the page
tar -xzf bitcoin-<VERSION>-x86_64-linux-gnu.tar.gz
sudo install -m 0755 bitcoin-<VERSION>/bin/bitcoind /usr/local/bin/
sudo install -m 0755 bitcoin-<VERSION>/bin/bitcoin-cli /usr/local/bin/
```

On Ubuntu you can also use the official PPA:

```bash
sudo add-apt-repository ppa:bitcoin/bitcoin
sudo apt update
sudo apt install bitcoind
```

### Core Lightning

Download the pre-built binary for your Ubuntu version from the [latest CLN release](https://github.com/ElementsProject/lightning/releases):

```bash
CLN_VERSION=v25.02
UBUNTU_VERSION=22.04
TARBALL="clightning-${CLN_VERSION}-Ubuntu-${UBUNTU_VERSION}-amd64.tar.xz"
wget "https://github.com/ElementsProject/lightning/releases/download/${CLN_VERSION}/${TARBALL}"
sudo tar -xJf "$TARBALL" -C /usr/local --strip-components=2
```

For other distributions, or to pin to a specific commit for reproducibility, build from source: [github.com/ElementsProject/lightning/blob/master/doc/getting-started/getting-started/installation.md](https://github.com/ElementsProject/lightning/blob/master/doc/getting-started/getting-started/installation.md).

Verify:

```bash
lightningd --version
lightning-cli --version
```

---

## Regtest configuration — generated for you

The harness runs entirely on `regtest`. All configuration files are generated automatically by `scripts/startup/0.1.infra_boot.sh` from the templates under `ln-ai-network/config/`:

- `runtime/bitcoin/shared/bitcoin.conf` — Bitcoin Core config (RPC credentials, ZMQ ports, regtest flag).
- `runtime/lightning/node-N/config` — per-node Core Lightning config, rendered from `ln-ai-network/config/cln/lightning.conf.tpl`.

You do not need to edit these files manually. Override settings via the project's `.env` instead — the boot script reads `env.sh`, which reads `.env`, and threads the values into the config templates.

---

## What the harness creates when you run `./start.sh`

1. Starts `bitcoind` in daemon mode on `regtest` (RPC port 18443, P2P port 18444).
2. Creates the `shared-wallet` Bitcoin wallet.
3. Starts one `lightningd` process per node, base port 9735 incremented per node (node 1 → 9736, node 2 → 9737, …).
4. Connects each node to every other node as Lightning peers (full mesh for regtest convenience).
5. Funds each node's on-chain wallet with regtest coins.
6. Mines 101 blocks to cross the coinbase-maturity threshold so the funding transactions are spendable.
7. Opens a channel between nodes 1 and 2 (plus each additional node), mines 6 blocks to confirm.

All data lives under `ln-ai-network/runtime/` and is cleaned up by `./stop.sh` (which archives the per-node Lightning directories rather than deleting them, so prior state survives a restart).

---

## Port reference

| Service | Default port | Environment variable |
|---------|--------------|----------------------|
| Bitcoin Core RPC | 18443 | `BITCOIN_RPC_PORT` |
| Bitcoin Core P2P | 18444 | `BITCOIN_P2P_PORT` |
| Bitcoin Core ZMQ raw block | 28332 | (fixed in template) |
| Bitcoin Core ZMQ raw tx | 28333 | (fixed in template) |
| Lightning node N | `9735 + N` | `LIGHTNING_BASE_PORT` |
| Web UI | 8008 | `UI_PORT` |
| MCP server | stdio | — |

Override any port in `ln-ai-network/.env` and restart the stack.
