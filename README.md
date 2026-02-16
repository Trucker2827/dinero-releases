# Dinero (DNR)

**Real Money for Free People**

Dinero is a ground-up cryptocurrency built from scratch in C++. Not a fork of Bitcoin, not a copy of someone else's code. Every line of consensus, networking, mining, and wallet code was written specifically for Dinero. It takes the best ideas from Bitcoin's 15 years of evolution — Taproot, compact blocks, modern key derivation — and combines them with Utreexo, a breakthrough in how nodes store and verify the UTXO set. The result is a cryptocurrency that any computer can fully validate without storing gigabytes of chain state.

Genesis Block: 2025

---

## Downloads

**[Latest Release: v5.2.0](https://github.com/Trucker2827/dinero-releases/releases/tag/v5.2.0)**

| Platform | Type | Download |
|----------|------|----------|
| **macOS** | Desktop Wallet (GUI) | [dinero-qt-v5.2.0-macos-arm64.zip](https://github.com/Trucker2827/dinero-releases/releases/download/v5.2.0/dinero-qt-v5.2.0-macos-arm64.zip) |
| **macOS** | CLI Tools (daemon + miner) | [dinero-v5.2.0-macos-arm64.tar.gz](https://github.com/Trucker2827/dinero-releases/releases/download/v5.2.0/dinero-v5.2.0-macos-arm64.tar.gz) |
| **Linux** | Desktop Wallet (GUI) | [dinero-qt-v5.2.0-linux-x86_64.tar.gz](https://github.com/Trucker2827/dinero-releases/releases/download/v5.2.0/dinero-qt-v5.2.0-linux-x86_64.tar.gz) |
| **Linux** | CLI Tools (daemon + miner) | [dinero-v5.2.0-linux-x86_64.tar.gz](https://github.com/Trucker2827/dinero-releases/releases/download/v5.2.0/dinero-v5.2.0-linux-x86_64.tar.gz) |

### What's in each package

**CLI Tools** contain:
- `dinerod` — Full node daemon (the backbone of the network)
- `dinero-miner` — Standalone CPU miner (SHA-256)
- `dinero-cli` — Command-line RPC client for interacting with the daemon

**Desktop Wallet (dinero-qt)** contains:
- Full GUI wallet with send/receive, address book, transaction history
- Embedded daemon — automatically starts and manages `dinerod` in the background
- Built-in solo miner — mine directly from the wallet GUI
- QR code support for easy address sharing

### Verify your download

Every release includes `SHA256SUMS.txt`. Verify the integrity of your download:

```bash
# macOS
shasum -a 256 -c SHA256SUMS.txt

# Linux
sha256sum -c SHA256SUMS.txt
```

---

## Quick Start

### Desktop Wallet (Recommended for most users)

**macOS:**
```bash
unzip dinero-qt-v5.2.0-macos-arm64.zip
open dinero-qt.app
```

**Linux:**
```bash
tar xzf dinero-qt-v5.2.0-linux-x86_64.tar.gz
cd dinero-qt-v5.2.0-linux-x86_64
./dinero-qt
```

The wallet automatically starts the daemon, connects to the network, and begins syncing. Create a new wallet, write down your 12-word seed phrase, and you're ready to go.

### CLI (For server operators and advanced users)

**macOS:**
```bash
tar xzf dinero-v5.2.0-macos-arm64.tar.gz
cd dinero-v5.2.0-macos-arm64
./dinerod --listen --rpc
```

**Linux:**
```bash
tar xzf dinero-v5.2.0-linux-x86_64.tar.gz
cd dinero-v5.2.0-linux-x86_64
./dinerod --listen --rpc
```

The daemon creates its data directory at `~/.dinero` and connects to seed nodes automatically.

### Start mining

Once the daemon is synced:

```bash
# Start mining with 4 threads to your address
./dinero-miner --address dnr1your_address_here --threads 4
```

Or use the built-in miner in dinero-qt — just go to the Mining tab, enter your address, and click Start.

---

## Why Dinero?

### Built from scratch, not a fork

Most altcoins are forks of Bitcoin Core with a few parameters changed. Dinero is written from the ground up. Every component — consensus engine, P2P networking, block validation, wallet, miner — is original code. This means:

- **No inherited technical debt** from 15 years of Bitcoin Core refactoring
- **Clean architecture** designed for modern C++17 from day one
- **Purpose-built** for the features we want, not constrained by backwards compatibility
- **Auditable** — the codebase is focused and readable, not 500K+ lines of accumulated history

### Utreexo: Run a full node on anything

This is Dinero's most important technical innovation. Here's the problem it solves:

In Bitcoin, every full node must store the entire UTXO set — every unspent coin in existence. Today that's over 7 GB and growing. This is why most people don't run full nodes. They trust someone else to validate transactions for them, which undermines the entire point of a decentralized currency.

**Utreexo** (invented by Tadge Dryja at MIT) replaces the UTXO set with a cryptographic accumulator — a compact hash-based data structure that can prove any coin exists (or doesn't) using a small proof instead of the entire database.

**What this means in practice:**

| | Bitcoin | Dinero (with Utreexo) |
|---|---------|----------------------|
| UTXO storage | ~7 GB and growing | ~1 KB (just the accumulator roots) |
| Full validation | Need the entire UTXO database | Proofs are included with transactions |
| Node requirements | SSD recommended, significant disk space | Runs on anything — Raspberry Pi, phone, old laptop |
| Sync from scratch | Download and index the whole UTXO set | Verify proofs as you go |

With Utreexo, every Dinero node is a fully-validating node. There are no "light clients" that trust others — every node checks every rule. This is what decentralization actually looks like.

**How it works (simplified):**

1. The Utreexo accumulator is a forest of perfect Merkle trees
2. When a coin is created, it's added as a leaf to the forest
3. When a coin is spent, its inclusion is proven via a Merkle path, then it's deleted
4. The entire state of all coins is compressed into a handful of root hashes
5. Each block includes the Utreexo roots in its header — everyone agrees on the state
6. State transitions are verified using cryptographic proofs, not database lookups

This isn't theoretical — it's live in Dinero today. Every block header commits to the Utreexo forest roots. Every transaction includes the proofs needed to verify it. No trust required.

### Taproot-only (BIP86 / P2TR)

Dinero uses Taproot (P2TR — Pay-to-Taproot) exclusively. There are no legacy address types, no P2PKH, no P2SH, no P2WPKH. Every address starts with `dnr1` and uses Schnorr signatures on the secp256k1 curve.

**Why Taproot-only matters:**

- **Better privacy** — All transactions look the same on-chain. You can't tell a simple payment from a complex smart contract by looking at it. In Bitcoin, different address types (1..., 3..., bc1q..., bc1p...) leak information about the wallet software and transaction type.

- **Schnorr signatures** — More efficient than ECDSA. Schnorr enables key aggregation (multiple signers produce a single signature that looks like any other), batch validation (verify many signatures faster than one-by-one), and cleaner math.

- **Simpler codebase** — Supporting one address type instead of four means less code, fewer edge cases, fewer bugs. The entire script validation engine is focused on one thing and does it well.

- **Future-ready** — Taproot includes a built-in upgrade mechanism via the script tree (tapscript). Future features like covenants, vaults, or advanced smart contracts can be added without changing the address format.

**Key derivation follows BIP86:**
```
m/86'/1447'/account'/change/index
```
- `86'` — BIP86 purpose (Taproot)
- `1447'` — Dinero's registered coin type
- Deterministic — your entire wallet is recoverable from the 12-word seed phrase

### BIP39 Mnemonic Seed (12-word backup)

Your wallet is protected by a 12-word seed phrase following the BIP39 standard. Write these 12 words on paper, store them safely, and you can recover your entire wallet — all addresses, all keys, all funds — on any device, forever.

```
example: abandon ability able about above absent absorb abstract absurd abuse access accident
```

The seed phrase generates a master key, which deterministically derives all your addresses. No more backing up wallet files. No more lost coins because of a hard drive failure. Twelve words is all you need.

### SHA-256 Proof of Work

Dinero uses the same SHA-256 mining algorithm as Bitcoin. This is a deliberate choice:

- **Battle-tested** — SHA-256 has been securing Bitcoin since 2009 with zero cryptographic breaks
- **ASIC-compatible** — The same hardware that mines Bitcoin can mine Dinero. No proprietary algorithms that lock miners into one coin
- **CPU-mineable at launch** — With a new chain, difficulty starts low enough that anyone with a computer can mine blocks and earn DNR
- **Real security** — Proof of Work means that attacking the chain requires real energy expenditure. No staking tricks, no validator committees, no trusted parties

### Bulletproof range proofs

Dinero includes Bulletproofs — a zero-knowledge proof system that proves a transaction amount is within a valid range (non-negative, not overflowing) without revealing the actual amount. This is a building block for confidential transactions and advanced privacy features.

### PSBT support (Hardware wallet ready)

Partially Signed Bitcoin Transactions (PSBTs) allow transaction construction to be separated from signing. This means:

- **Hardware wallets** — Create a transaction on your computer, sign it on a hardware device that never exposes the private key
- **Multi-signature** — Multiple parties can collaborate to sign a transaction, each adding their signature independently
- **Air-gapped signing** — Build the transaction online, transfer to an offline machine for signing, broadcast from the online machine

---

## Architecture

### Network

| Parameter | Value |
|-----------|-------|
| P2P Port | `20999` |
| RPC Port | `20998` |
| Address prefix | `dnr1` (Bech32m) |
| Coin type (BIP44) | `1447'` |
| Block time target | ~2 minutes |
| Mining algorithm | SHA-256d (double SHA-256) |
| Max supply | 265,428,000 DNR |
| Block reward | 100 DNR (halving every 1,314,000 blocks / ~5 years) |
| Smallest unit | 1 una = 0.00000001 DNR |
| Coinbase maturity | 100 confirmations |
| Consensus | Nakamoto consensus (longest valid chain) |
| UTXO model | Utreexo accumulator (forest of Merkle trees) |

### Software components

```
dinero-qt (Desktop Wallet)
├── GUI — wallet, send/receive, mining controls, address book
├── Embedded dinerod — full node runs inside the app
├── Solo miner — mine from the wallet GUI
└── QR codes — scan/generate for easy payments

dinerod (Full Node Daemon)
├── Consensus engine — block validation, Utreexo verification
├── P2P networking — peer discovery, block relay, transaction propagation
├── RPC server — JSON-RPC interface for wallets and tools
├── Mining service — internal CPU miner
├── Block relay manager — efficient block announcement and download
└── Wallet service — key management, transaction tracking

dinero-miner (Standalone CPU Miner)
├── SHA-256 mining engine — multi-threaded
├── RPC mode — connects to local dinerod
└── Stratum support — connect to mining pools

dinero-cli (Command-line client)
└── RPC client — send commands to running dinerod
```

### Data directory

Default location: `~/.dinero`

```
~/.dinero/
├── chainstate/     — Block headers, chain state (RocksDB)
├── blocks/         — Block data
├── utreexo/        — Utreexo accumulator forest
├── peers/          — Known peer addresses
├── wallets/        — Wallet databases
└── dinero.conf     — Configuration file (optional)
```

---

## Configuration

Create `~/.dinero/dinero.conf` (optional):

```ini
# Enable network listening and RPC
listen=1
rpcbind=127.0.0.1:20998

# Connect to known nodes
addnode=172.93.160.131:20999
addnode=173.249.195.59:20999

# Performance
maxconnections=50
```

### Command-line options

```bash
# Start daemon with custom data directory
./dinerod --datadir /path/to/data --listen --rpc

# Start daemon on testnet
./dinerod --testnet

# Start mining with the standalone miner
./dinero-miner --address dnr1youraddress --threads 4 --rpcport 20998
```

---

## RPC Interface

The daemon exposes a JSON-RPC interface on port 20998. Examples:

```bash
# Get current block height
./dinero-cli getblockcount

# Get block hash at height
./dinero-cli getblockhash 1000

# Get network info
./dinero-cli getpeerinfo

# Get mining status
./dinero-cli mining.status

# Start mining (4 threads)
./dinero-cli mining.start 4 dnr1youraddress

# Stop mining
./dinero-cli mining.stop

# Create a new wallet
./dinero-cli createwallet "mywallet"

# Get a new receiving address
./dinero-cli getnewaddress

# Send DNR
./dinero-cli sendtoaddress dnr1recipientaddress 10.0

# Get wallet balance
./dinero-cli getbalance
```

---

## Mining Guide

### Option 1: Mine from the Desktop Wallet (easiest)

1. Open dinero-qt
2. Go to the **Mining** tab
3. Your wallet address is auto-filled (or enter a different one)
4. Set thread count (start with half your CPU cores)
5. Click **Start Mining**

The wallet shows hash rate, blocks found, and mining status in real-time.

### Option 2: Standalone CPU Miner

```bash
# Start dinerod first
./dinerod --listen --rpc

# In another terminal, start mining
./dinero-miner --address dnr1youraddress --threads 4
```

The standalone miner connects to dinerod via RPC and submits solutions. It automatically detects the daemon and starts working.

### Option 3: Run dinerod's internal miner

```bash
# Start daemon with mining enabled
./dinerod --listen --rpc

# Use CLI to start mining
./dinero-cli mining.start 4 dnr1youraddress

# Check status
./dinero-cli mining.status

# Stop mining
./dinero-cli mining.stop
```

### Mining tips

- **Thread count**: Start with half your CPU cores. More threads = more hash power, but also more heat and power usage. Leave some cores free for the system.
- **Solo mining**: With low network difficulty, solo mining is practical. You get the full 100 DNR block reward.
- **Block time**: Target is ~2 minutes. With low network hash rate, you may find blocks faster.
- **Difficulty adjustment**: Adjusts every block based on recent block times. If you're the only miner, difficulty drops to match your hash rate.

---

## Running a Full Node

Running a full node strengthens the network. You validate every block and every transaction independently — no trust required.

```bash
# Start a full node
./dinerod --listen --rpc --datadir ~/.dinero

# Or use the desktop wallet — it runs a full node automatically
open dinero-qt.app
```

### Server deployment (headless Linux)

```bash
# Download and extract
tar xzf dinero-v5.2.0-linux-x86_64.tar.gz
cd dinero-v5.2.0-linux-x86_64

# Create systemd service (optional)
sudo tee /etc/systemd/system/dinerod.service << 'EOF'
[Unit]
Description=Dinero Full Node
After=network.target

[Service]
Type=simple
User=dinero
ExecStart=/opt/dinero/dinerod --listen --rpc --datadir /opt/dinero/data
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl enable dinerod
sudo systemctl start dinerod
```

### Minimum requirements

- **CPU**: Any modern processor (x86_64 or ARM64)
- **RAM**: 512 MB (Utreexo keeps memory usage minimal)
- **Disk**: 1 GB (Utreexo eliminates the need for a large UTXO database)
- **Network**: Broadband internet connection

Thanks to Utreexo, these requirements stay small even as the chain grows. A Raspberry Pi can run a fully-validating Dinero node.

---

## Comparison with Bitcoin

| Feature | Bitcoin | Dinero |
|---------|---------|--------|
| Codebase | Fork-based (15+ years of history) | Written from scratch (2025) |
| UTXO storage | ~7 GB UTXO set on disk | ~1 KB Utreexo accumulator |
| Full node disk | 600+ GB (full chain) | Minimal (Utreexo proofs) |
| Address types | 4 types (P2PKH, P2SH, P2WPKH, P2TR) | 1 type (P2TR only) |
| Signatures | ECDSA + Schnorr | Schnorr only |
| Key derivation | BIP44/49/84/86 | BIP86 only (Taproot) |
| Mining algorithm | SHA-256 | SHA-256 (same) |
| Max supply | 21M BTC | 265M DNR |
| Block time | ~10 min | ~2 min |
| Seed phrase | BIP39 (12/24 words) | BIP39 (12 words) |
| PSBT | Supported | Supported |
| Privacy baseline | Mixed (address type leaks info) | Uniform (all txs look identical) |

---

## Utreexo Deep Dive

For those who want to understand the technical details:

### The UTXO problem

Every cryptocurrency based on the UTXO model (Bitcoin, Litecoin, etc.) has the same fundamental scaling problem: the set of all unspent transaction outputs grows over time and must be stored by every full node. In Bitcoin, this is over 80 million UTXOs consuming 7+ GB of SSD space, and it's required for basic transaction validation.

This creates a centralization pressure. As the UTXO set grows:
- Home users stop running nodes (too much disk, too slow to sync)
- Validation concentrates in data centers and exchanges
- The network becomes less decentralized over time

### The Utreexo solution

Utreexo replaces the UTXO database with a **dynamic hash-based accumulator** — specifically, a forest of perfect Merkle trees. Instead of storing every UTXO, nodes store only the roots of these trees (a handful of 32-byte hashes).

**Adding a UTXO** (when a coin is created):
- The UTXO is hashed and inserted as a new leaf
- Trees of equal height are merged (paired and hashed together)
- Only the root hashes change

**Removing a UTXO** (when a coin is spent):
- A Merkle proof (the sibling hashes along the path from leaf to root) proves the UTXO exists
- The leaf is deleted and the tree structure updates
- Again, only root hashes change

**Block header commitment:**
- Every Dinero block header includes the Utreexo forest roots
- This means every node can independently verify the state without a database
- If the roots don't match after applying a block's transactions, the block is invalid

### State transition proofs

Each block in Dinero carries a **Utreexo transition proof** — the cryptographic evidence needed to verify that all spent coins existed and all state changes are correct. This means:

1. A new node doesn't need to download the UTXO set to start validating
2. Each block is self-contained: it includes everything needed to verify itself
3. Pruning is natural: once a block is validated, the proofs can be discarded

### Security guarantees

Utreexo's security rests on SHA-256 collision resistance — the same assumption that secures Bitcoin's Proof of Work. Specifically:

- **Inclusion proof**: An adversary cannot forge a Merkle proof for a UTXO that doesn't exist without finding a SHA-256 collision
- **Exclusion proof**: An adversary cannot hide an existing UTXO (pretend it was already spent) without the forest roots being wrong, which every node independently verifies
- **Binding**: The forest roots in the block header commit to the exact set of all unspent outputs. Any modification produces different roots.

### Full Technical Paper

For the complete engineering story — the 128-byte header format, the two-pass validation algorithm, the state transition proof system, and the ten consensus-critical bugs we found and fixed — read the full whitepaper:

**[Utreexo in Production: Engineering a Cryptographic Accumulator for Live Consensus](UTREEXO-WHITEPAPER.md)**

### References

- Utreexo paper: Thaddeus Dryja, "Utreexo: A dynamic hash-based accumulator optimized for the Bitcoin UTXO set" (MIT DCI, 2019)
- BIP341 (Taproot): Pieter Wuille, Jonas Nick, Anthony Towns
- BIP86 (Taproot key derivation): Andrew Chow
- BIP39 (Mnemonic seed phrases): Marek Palatinus, Pavol Rusnak, Aaron Voisine, Sean Bowe
- Bulletproofs: Bunz, Bootle, Boneh, Poelstra, Wuille, Maxwell (Stanford/Blockstream, 2017)

---

## Troubleshooting

### Wallet says "Connecting..." / daemon won't start

1. Check if another dinerod instance is already running:
   ```bash
   # macOS
   ps aux | grep dinerod
   # Linux
   pgrep -a dinerod
   ```
2. Check if port 20998 is in use:
   ```bash
   lsof -i :20998
   ```
3. Kill any existing instance and restart the wallet

### Sync seems slow

The chain is young. Initial sync should take only minutes. If it's taking longer:
- Check your internet connection
- The daemon automatically connects to seed nodes. Check the Debug Console (in dinero-qt) for peer connection messages
- You can manually add nodes in `~/.dinero/dinero.conf`:
  ```ini
  addnode=172.93.160.131:20999
  addnode=173.249.195.59:20999
  ```

### macOS security warning ("unidentified developer")

Right-click the app and select "Open" instead of double-clicking. Alternatively:
```bash
xattr -cr dinero-qt.app
open dinero-qt.app
```

### Mining shows 0 hashrate

- Make sure the daemon is fully synced before mining
- Check that your mining address is valid (must start with `dnr1`)
- Try reducing thread count if your system is memory-constrained

---

## Building from Source

Dinero is open-source software. For those who want to build from source rather than use pre-built binaries:

### Dependencies

- C++17 compiler (GCC 9+ or Clang 12+)
- CMake 3.21+
- OpenSSL 3.x
- RocksDB
- secp256k1
- Qt6 (for dinero-qt only)

### Build dinerod

```bash
git clone <source-repo>
cd dinero
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
```

### Build dinero-qt

```bash
git clone <qt-repo>
cd dinero-qt
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH=/path/to/Qt/6.x/
make -j$(nproc)
```

---

## DineroDPI (iOS Wallet)

A native iOS wallet is in development. DineroDPI (Dinero Direct Payment Interface) brings the full Dinero experience to iPhone:

- **Full node in your pocket** — NodeCore.xcframework embeds the same C++ consensus engine as dinerod, running directly on your phone
- **Pure Swift** — No JavaScript bridges, no React Native. Native Swift with secp256k1 for cryptographic operations
- **Instant verification** — Tier 1 cryptographic verification + Tier 2 mempool confirmation for fast payment confidence
- **Background sync** — Headers, blocks, and Utreexo proofs sync in the background
- **Taproot-native** — Same BIP86 key derivation, same addresses, same seed phrase as the desktop wallet

---

## Community

Dinero is a community project. No ICO, no premine, no VC funding. Just code, mining, and people who believe money should be free.

---

*Dinero: Real Money for Free People*

*Genesis Block 2025*
