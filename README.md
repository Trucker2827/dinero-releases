# Dinero Releases

Official binary releases for Dinero (DNR) - Real Money for Free People.

## v0.2.0-stable

Wallet security hardening release. All seed/mnemonic persistence is now encrypted-at-rest.

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

### What's new in v0.2.0

- **Encrypted-at-rest wallet storage** - seed and mnemonic are always AES-256-GCM encrypted on disk, even for unencrypted wallets (sealed with empty passphrase)
- **KDF parameter persistence** - encryption metadata table now records algorithm, iteration count, cipher, and salt with every encrypted blob
- **Version-dispatched seed decryption** - loadMasterSeed reads encryption_version from DB instead of brute-forcing iteration candidates
- **Legacy plaintext load blocked** - HDWalletManager refuses to load old mnemonic_plaintext entries (migration required)
- **FFI security hardening** - global mnemonic variable removed, ScopeExit RAII cleanup for sensitive data, mnemonic readback APIs disabled
- **Migration-safe** - legacy encrypted formats (Argon2id, old PBKDF2) still load correctly

### What was in v0.1.0

- BIP86 Taproot-only wallet (m/86'/1447'/0'/0/*)
- BIP39 12-word mnemonic seed backup
- SHA256d Proof-of-Work consensus
- PSBT support (hardware wallet compatible)
- Canonical PBKDF2-HMAC-SHA512 key derivation (broken HMAC fallback removed)
- BIP86 derivation regression test pinned

## Quick Start

1. Download binaries for your platform
2. **macOS: remove quarantine flag before first run:**
   ```bash
   xattr -cr dinerod dinero-cli dinero-miner dinero-qt.app
   ```
3. Run `dinerod` to start the node
4. Run `dinero-qt.app` for GUI wallet (macOS)
5. Create wallet with 12-word seed phrase

## Links

- Website: https://dinero-coin.com (coming soon)

---

Dinero: Real Money for Free People - Genesis Block 2025
