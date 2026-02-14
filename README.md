# Dinero Releases

Official binary releases for Dinero (DNR) - Real Money for Free People.

## Downloads

**[Latest Release: v4.3.0](https://github.com/Trucker2827/dinero-releases/releases/tag/v4.3.0)**

| Platform | Architecture | Download |
|----------|-------------|----------|
| **macOS** CLI | Apple Silicon (arm64) | [dinero-v4.3.0-macos-arm64.tar.gz](https://github.com/Trucker2827/dinero-releases/releases/download/v4.3.0/dinero-v4.3.0-macos-arm64.tar.gz) |
| **macOS** Qt Wallet | Apple Silicon (arm64) | [dinero-qt-v4.3.0-macos-arm64.zip](https://github.com/Trucker2827/dinero-releases/releases/download/v4.3.0/dinero-qt-v4.3.0-macos-arm64.zip) |
| **Linux** CLI | x86_64 | [dinero-v4.3.0-linux-x86_64.tar.gz](https://github.com/Trucker2827/dinero-releases/releases/download/v4.3.0/dinero-v4.3.0-linux-x86_64.tar.gz) |
| **Linux** Qt Wallet | x86_64 | [dinero-qt-v4.3.0-linux-x86_64.tar.gz](https://github.com/Trucker2827/dinero-releases/releases/download/v4.3.0/dinero-qt-v4.3.0-linux-x86_64.tar.gz) |

## What's Included

- `dinerod` - Full node daemon
- `dinero-cli` - Command-line RPC client
- `dinero-miner` - CPU miner (SHA256)
- `dinero-qt` - Desktop wallet (macOS .app bundle / Linux binary)

## Verify Downloads

```bash
shasum -a 256 -c SHA256SUMS.txt
```

## Quick Start

### macOS
```bash
tar xzf dinero-v4.3.0-macos-arm64.tar.gz
cd dinero-v4.3.0-macos-arm64
./dinerod
```

### Linux
```bash
tar xzf dinero-v4.3.0-linux-x86_64.tar.gz
cd dinero-v4.3.0-linux-x86_64
./dinerod
```

### Qt Wallet (macOS)
```bash
unzip dinero-qt-v4.3.0-macos-arm64.zip
open dinero-qt.app
```

### Qt Wallet (Linux)
```bash
sudo apt install qt6-base-dev
tar xzf dinero-qt-v4.3.0-linux-x86_64.tar.gz
./dinero-qt-v4.3.0-linux-x86_64/dinero-qt
```

## Features

- **BIP86 Taproot-only** - Modern, privacy-preserving P2TR addresses
- **BIP39 mnemonic** - 12-word seed phrase backup
- **Utreexo accumulator** - Lightweight full node verification
- **SHA256 Proof-of-Work** - Real mining, real security
- **PSBT support** - Hardware wallet compatible

---

Dinero: Real Money for Free People - Genesis Block 2025
