# Dinero Releases

Official binary releases for Dinero (DNR) - Real Money for Free People.

## v0.1.0-stable

Canonical wallet baseline. BIP86 Taproot derivation frozen at coin type 1447.

### Downloads

| Platform | Status |
|----------|--------|
| **macOS** (Apple Silicon arm64) | Available |
| **Linux** (x86_64) | Available |

### Binaries

- `dinerod` - Full node daemon
- `dinero-cli` - Command-line RPC client
- `dinero-miner` - CPU miner
- `dinero-qt.app` - GUI wallet with embedded miner (macOS only)

### Verification

```bash
cd mac && shasum -a 256 -c SHA256SUMS.txt
cd linux && shasum -a 256 -c SHA256SUMS.txt
```

## What's in this release

- BIP86 Taproot-only wallet (m/86'/1447'/0'/0/*)
- BIP39 12-word mnemonic seed backup
- SHA256d Proof-of-Work consensus
- PSBT support (hardware wallet compatible)
- Canonical PBKDF2-HMAC-SHA512 key derivation (broken HMAC fallback removed)
- BIP86 derivation regression test pinned

## Quick Start

1. Download binaries for your platform
2. Run `dinerod` to start the node
3. Run `dinero-qt.app` for GUI wallet (macOS)
4. Create wallet with 12-word seed phrase

## Links

- Website: https://dinero-coin.com (coming soon)

---

Dinero: Real Money for Free People - Genesis Block 2025
