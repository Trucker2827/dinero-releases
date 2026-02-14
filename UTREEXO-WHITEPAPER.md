# Utreexo in Production: Engineering a Cryptographic Accumulator for Live Consensus

**Armin Hajiyev**
**arminhaj@stanford.edu**

**February 2026**

---

## Abstract

Utreexo, proposed by Thaddeus Dryja in 2019, describes a dynamic hash-based accumulator that replaces the ever-growing UTXO database with a compact set of Merkle tree roots. The theory is elegant. The implementation is brutal. This paper documents the first production deployment of Utreexo as a consensus-critical component — not as a sidecar, not as an optional optimization, but as a mandatory field in every block header, verified by every node, from the genesis block forward. We describe the 128-byte header format that commits to the accumulator state via Proof of Work, the two-pass block validation algorithm, the state transition proof system that enables fully stateless verification, and the ten critical bugs we found and fixed along the way — any one of which would have caused a consensus failure, chain split, or silent data corruption in production. The system has been live on Dinero mainnet since genesis (2025), processing blocks with Utreexo verification at every height.

---

## 1. The Problem: UTXO Set Growth as Centralization Pressure

Every UTXO-based cryptocurrency carries a fundamental scaling problem: the set of all unspent transaction outputs must be stored and queried by every fully-validating node. In Bitcoin, this set has grown to over 80 million entries consuming 7+ GB of fast storage, and it can only be accessed via SSD-backed database lookups.

This creates a centralizing force that undermines the network's security model:

1. **Home nodes become impractical.** Users who cannot dedicate SSD space and bandwidth to maintaining the UTXO set stop running full nodes.

2. **Validation concentrates.** When fewer people validate, the network's security degrades from "trustless" to "trust the operators of a few thousand nodes."

3. **The problem compounds.** Every new UTXO added to the set makes the problem slightly worse, forever.

4. **Light clients are not a solution.** SPV nodes verify Proof of Work but trust miners for transaction validity. Bloom-filter-based light clients leak privacy. Neutrino-style compact block filters improve privacy but still trust the server for UTXO existence proofs.

The root cause is that transaction validation requires answering a simple question — "does this coin exist and is it unspent?" — and the only known way to answer it has been to maintain the complete UTXO database.

Utreexo provides another way.

---

## 2. Utreexo Theory

Dryja's insight is that the UTXO set can be replaced with a *cryptographic accumulator* — a data structure that compactly represents a set and supports efficient proofs of membership, without storing the set itself.

### 2.1 The Forest Structure

The Utreexo accumulator is a forest of perfect binary Merkle trees. The number and heights of trees are determined by the binary representation of the leaf count. For example, with 13 leaves (binary `1101`), the forest contains three trees of heights 3, 2, and 0 (sizes 8, 4, and 1).

Each tree's root is a standard Merkle root: the recursive SHA-256 hash of its children. The complete accumulator state is the ordered set of these roots — typically a handful of 32-byte hashes, regardless of how many UTXOs exist.

### 2.2 Operations

**Add** (new UTXO created): A new leaf is appended. If this creates two trees of equal height, they are merged by hashing their roots together, forming a single tree one level taller. This may cascade.

**Remove** (UTXO spent): The spender provides a Merkle proof — the sibling hashes along the path from the leaf to its root. The verifier checks the proof against the known root, then updates the forest structure to reflect the deletion.

**Commitment**: The accumulator state is the set of roots. Two nodes with the same roots agree on the exact same set of UTXOs.

### 2.3 The Gap Between Theory and Production

Dryja's paper describes the data structure and its asymptotic properties. What it does not describe — because it was not the paper's scope — is how to integrate this into a live consensus system where:

- The accumulator state must be committed in the block header and covered by Proof of Work
- Miners must compute the *after-state* commitment before they know the nonce
- Block validation must update the accumulator atomically with UTXO set mutations
- Reorganizations must be able to undo accumulator updates efficiently
- Stateless nodes (which store only the roots) must be able to validate blocks using proofs carried in the block data
- The position model and root ordering must be identical across all implementations or the network splits
- Proofs must be generated *before* the state update that destroys the data needed to generate them
- Crash recovery must handle partial state (forest updated but tip not stored, or vice versa)

Each of these is a source of bugs. We found bugs in all of them.

---

## 3. Dinero's Implementation

### 3.1 Block Header: Utreexo Root in the PoW Hash

Dinero's block header is a frozen 128-byte structure:

```
Offset  Size   Field
0x00    4      version
0x04    32     prev_block_hash
0x24    32     merkle_root
0x44    32     utreexo_root        <-- accumulator commitment
0x64    8      timestamp
0x6C    4      difficulty_bits
0x70    4      nonce
0x74    12     reserved (must be zero)
```

The block hash is `SHA-256d(header[0..128])`. Because `utreexo_root` is at offset 0x44 inside the hashed region, **the Utreexo commitment is covered by Proof of Work**. A miner cannot find a valid nonce for a header with an incorrect Utreexo root — doing so would require a SHA-256 collision.

This is a fundamental design choice. In Bitcoin, if Utreexo were added retroactively, the commitment would likely go in the coinbase transaction (similar to SegWit's witness commitment). This works but means the commitment is one Merkle branch removed from the PoW hash. In Dinero, building from scratch, we placed it directly in the header.

The `utreexo_root` field contains the **after-state** commitment: the accumulator state after all transactions in the block have been applied (all spent UTXOs removed, all new UTXOs added). This means a node that validates block N can verify that block N+1's `utreexo_root` is consistent with the transactions it contains.

The field is enforced at compile time:

```cpp
static_assert(sizeof(BlockHeader) == 128, "BlockHeader v1 MUST be exactly 128 bytes");
static_assert(offsetof(BlockHeader, utreexo_root) == 0x44, "utreexo_root offset must be 0x44");
```

### 3.2 Leaf Hash: Domain-Separated UTXO Commitment

Each UTXO is hashed into a 32-byte leaf using a domain-separated SHA-256:

```
leaf = SHA-256("DINERO-UTXO-LEAF-v1" || txid || vout_le32 || amount_le64 || scriptPubKey)
```

The 19-byte ASCII domain tag `"DINERO-UTXO-LEAF-v1"` prevents cross-protocol hash collisions and provides a clean versioning mechanism for future upgrades. The `txid` is in binary wire format (not display-order hex). The amount and vout are little-endian.

Internal nodes use standard Merkle hashing: `SHA-256(left || right)`.

The commitment hash over the root set is: `SHA-256(root_0 || root_1 || ... || root_k)` for non-empty roots in height order (smallest height first), or 32 zero bytes for an empty accumulator.

### 3.3 Two Data Structures: Forest and Stump

We implement two accumulator variants:

**UtreexoForest** — The full accumulator. Stores every node in every tree plus a reverse index from leaf hashes to positions. Used by bridge nodes and full nodes. Can generate proofs, verify proofs, and compute commitments. Memory usage is O(n) in the number of UTXOs.

```cpp
std::vector<std::optional<UtreexoHash>> roots_;           // Height-indexed
uint64_t numLeaves_;
std::vector<std::optional<UtreexoHash>> nodes_;           // position -> hash
std::unordered_map<UtreexoHash, uint64_t> leaf_positions_; // hash -> position
std::unordered_set<uint64_t> deleted_positions_;
```

**UtreexoStump** — Roots-only accumulator. Stores only the root hashes and the leaf count. Used by Compact State Nodes (CSN) that verify blocks using proofs provided by bridge nodes. Can verify proofs and apply state updates, but cannot generate proofs. Storage is O(log n) — about 2 KB regardless of UTXO set size.

```cpp
std::array<std::optional<UtreexoHash>, 64> roots_;  // 64 slots (supports up to 2^64 leaves)
uint64_t numLeaves_;
```

Both must produce identical commitments for the same UTXO set. This invariant is enforced by tests and verified at runtime.

### 3.4 Height-Indexed Roots and the `getRoots()` Bug

Roots are stored in height-indexed arrays where `roots_[h]` is the root of the tree at height `h`, or `nullopt` if no tree exists at that height. This representation is critical because after deleting UTXOs, some tree slots may become empty — a tree of height 3 might lose all its leaves, leaving `roots_[3] = nullopt`.

Early code included a convenience function `getRoots()` that filtered out null entries:

```cpp
std::vector<UtreexoHash> getRoots() const {
    std::vector<UtreexoHash> result;
    for (auto& r : roots_)
        if (r.has_value()) result.push_back(*r);
    return result;
}
```

This function **loses height information**. If you reconstruct a stump from these non-null roots, you cannot assign them to the correct height slots. We discovered this when transition proofs failed for blocks that spent UTXOs from trees that were subsequently emptied.

The fix: `getIndexedRoots()` returns `const vector<optional<UtreexoHash>>&` — the raw height-indexed array. All consensus code uses `getIndexedRoots()`. The lossy `getRoots()` survives only for display purposes.

### 3.5 Position Model Alignment: MSB-to-LSB

The forest decomposes `numLeaves` into binary components and scans from the Most Significant Bit to the Least Significant Bit. With 13 leaves (`1101`):

- Bit 3 (MSB): tree of 8 leaves at positions 0-7
- Bit 2: tree of 4 leaves at positions 8-11
- Bit 0 (LSB): tree of 1 leaf at position 12

Both the forest and the stump must scan in the same direction, or batch proofs verified by the stump will disagree with proofs generated by the forest.

Our initial stump implementation scanned LSB-to-MSB (smallest tree first). This produced correct commitments (the commitment hash is order-independent for non-empty roots) but incorrect position-to-root mappings. When the stump tried to verify a batch deletion proof, it assigned proof targets to the wrong trees.

The fix was straightforward once identified: both `UtreexoForest::getTreeHeight()` and `UtreexoStump::getTreeHeight()` now use identical MSB-to-LSB scanning loops.

---

## 4. Block Validation: The Two-Pass Algorithm

When a full node validates a new block, the Utreexo forest must be updated to reflect all spent and created UTXOs. This is implemented as a two-pass algorithm on a *cloned* forest:

### Pass 1: Deletions

For each transaction input (except coinbase):
1. Compute the leaf hash from the spent UTXO's data (txid, vout, amount, scriptPubKey)
2. Find the leaf's position in the cloned forest
3. Generate a Merkle proof
4. Remove the leaf from the clone

### Pass 2: Additions

For each transaction output (excluding outputs spent within the same block):
1. Compute the leaf hash from the new UTXO's data
2. Add the leaf to the cloned forest
3. Record the assigned position in the UTXO position index

After both passes, the clone's commitment is extracted and compared against the block header's `utreexo_root`. If they match, the clone replaces the canonical forest. If not, the clone is discarded and the block is rejected.

### Snapshot-First Architecture

The key architectural decision is that mutations happen on a *clone*, never on the canonical state. This makes error recovery trivial — if anything goes wrong at any point, we throw away the clone. The canonical forest is untouched.

```
canonical_forest ──clone──> snapshot
                            ├── PASS 1: remove spent UTXOs
                            ├── PASS 2: add new UTXOs
                            ├── verify commitment
                            └── if OK: canonical = std::move(snapshot)
                               if FAIL: discard snapshot
```

For disconnecting blocks (during reorganizations), the undo data contains a compact delta (not a full forest snapshot), and the pre-block UTXO set snapshot can be restored directly.

### Deferred UTXO Mutations (INV-2)

A subtle ordering constraint: UTXO database mutations (SpendCoin, AddCoin) must not happen until the Utreexo proof has been fully verified. If we spend a coin from the UTXO database before verifying the Utreexo proof, and the proof turns out to be invalid, we've corrupted the UTXO database.

The solution is deferred execution: spent coins are collected in a `pending_utxo_spends` vector during validation. Only after all proofs pass are the pending spends committed:

```cpp
// During validation:
pending_utxo_spends.push_back({outpoint, undo_entry});

// After all proofs verified (INV-2):
for (auto& [outpoint, undo] : pending_utxo_spends)
    consensus_utxo_set_->SpendCoin(outpoint);
```

---

## 5. State Transition Proofs: Fully Stateless Verification

A state transition proof allows a node holding only the accumulator roots (a stump) to verify an entire block without any UTXO database. The proof carries everything needed to verify the state change from pre-block roots to post-block roots.

### 5.1 Structure

```cpp
struct UtreexoTransitionProof {
    vector<optional<UtreexoHash>> roots_before;          // Height-indexed pre-state
    uint64_t num_leaves_before;

    vector<UtreexoHash> deletion_targets;                 // Leaf hashes being spent
    vector<uint64_t> deletion_positions;                  // Their positions in the forest
    vector<UtreexoHash> deletion_proof_hashes;            // Merkle siblings for batch proof

    vector<optional<UtreexoHash>> roots_after_deletions;  // Intermediate state

    vector<UtreexoHash> addition_hashes;                  // New UTXO leaf hashes

    UtreexoHash commitment_after;                         // Expected post-state commitment
    uint64_t num_leaves_after;
};
```

### 5.2 Generation (Bridge Nodes)

A bridge node (which maintains the full forest) generates the proof at `ConnectTip` time — critically, *before* the forest is mutated by `ConnectBlock`:

1. Capture `roots_before` from the canonical forest using `getIndexedRoots()`
2. Clone the forest
3. **Pass 1**: Remove all deletion targets from the clone; capture `roots_after_deletions`
4. **Pass 2**: Add all new UTXOs to the clone; extract `commitment_after`

The timing constraint ("before ConnectBlock") was the source of a serious bug described in Section 6.

### 5.3 Verification (Stateless Nodes)

A Compact State Node verifies the proof without any forest data:

1. Reconstruct `stump_before` from `roots_before` and `num_leaves_before`
2. Verify the batch deletion proof against `stump_before` (this proves all deletion targets existed)
3. Reconstruct `stump_mid` from `roots_after_deletions`
4. Apply all `addition_hashes` to `stump_mid`
5. Verify `stump_mid.getCommitment() == commitment_after`
6. Verify `stump_mid.getNumLeaves() == num_leaves_after`

If step 2 passes, the deletions are cryptographically bound to `roots_before`. If step 5 passes, the additions produce the claimed commitment. An attacker providing fake `roots_after_deletions` would need to find a SHA-256 collision — the intermediate roots plus additions must hash to the same commitment as the real state.

### 5.4 Wire Format

Transition proofs are carried in the `utxoblk` wire message (version 3):

```
version (1 byte, 0x03) + block data + transition_proof
```

Nodes that don't support v3 fall back to legacy batch proofs (v2), which require the receiver to maintain a forest.

---

## 6. The Bugs: What Went Wrong and Why

### 6.1 The Premine Root Mismatch

**What happened**: At genesis, block 1 (the premine block) was mined with a Utreexo root computed using raw `SHA-256(txid || vout || amount || script)`. But the consensus validation function `HashUTXO()` uses domain-separated hashing: `SHA-256("DINERO-UTXO-LEAF-v1" || txid || vout || amount || script)`. On fresh startup, the forest was seeded with domain-separated hashes, producing a different root than what was in the block header.

**Impact**: Every new node would reject block 1.

**Fix**: All code paths — premine generation, mining, validation — now use `HashUTXO()` exclusively. Block 1 was re-mined with the correct root.

**Lesson**: When you introduce domain separation, you must audit *every* code path that computes the hash, including one-time bootstrap code that "will never run again."

### 6.2 The Extranonce Problem

**What happened**: Miners traditionally place an "extranonce" in the coinbase transaction's scriptSig to expand the nonce search space. But the coinbase's txid is `SHA-256d(coinbase_data)`, and the txid is part of the UTXO leaf hash (which is committed in the Utreexo root, which is in the header). Changing the extranonce changes the txid, which changes the leaf hash, which changes the Utreexo root, which changes the header — creating a circular dependency that makes mining impossible.

**Impact**: Miners could not use extranonce, limiting the search space to 2^32 nonces.

**Fix**: Move the extranonce from scriptSig to witness data. In SegWit-style transactions, the witness is not included in the txid computation. This breaks the circular dependency: the extranonce can change without affecting the txid, leaf hash, or Utreexo root.

A hard consensus rule enforces that the coinbase witness contains exactly one item of exactly 8 bytes (the extranonce). Any deviation invalidates the block.

**Lesson**: Utreexo creates new dependencies between block components that didn't exist before. The coinbase-extranonce-txid-leaf-root chain is non-obvious but breaks mining entirely.

### 6.3 The Proof Staleness Bug

**What happened**: When a peer requested a block, the node would generate the Utreexo proof on demand from the current forest state. But by the time the peer asked, `ConnectBlock` had already been called — the forest had been updated to the *after-state*. The proof was generated against the wrong roots, using positions that no longer existed.

**Impact**: Peers (especially the iOS wallet DineroDPI) rejected every block during initial sync because every proof failed verification.

**Fix**: Generate the proof *before* `ConnectBlock`, while the forest still reflects the pre-block state, and cache it for later serving. The call order in `ConnectTip` is now:

```
1. GenerateProofForBlock(block, forest)   // forest is PRE-block state
2. ConnectBlock(block)                     // forest mutates to POST-block state
3. CacheProof(block_hash, proof)          // serve cached proof to peers
```

**Lesson**: Proof generation and state mutation must never be reordered. This is a classic TOCTOU (time-of-check-time-of-use) bug, made non-obvious by the fact that the "check" (proof generation) and the "use" (state mutation) happen in different subsystems.

### 6.4 The Mining Root Overwrite

**What happened**: The block template builder correctly computed the after-state Utreexo root using `ComputeUtreexoRootPure()` (which clones the forest, applies the block's transactions, and extracts the commitment). But the RPC handler that serialized the template for the miner overwrote the header's `utreexo_root` field with the *current* forest commitment — the before-state.

**Impact**: Every mined block contained the wrong Utreexo root and was rejected by validators.

**Fix**: The RPC handler now serializes the builder-computed root without modification.

**Lesson**: Any code path between template construction and block serialization that touches the header is suspect. The header must be treated as immutable after the builder finalizes it.

### 6.5 The Stump Position Misalignment

**What happened**: The forest scanned the binary representation of `numLeaves` from MSB to LSB (largest tree first). The stump scanned from LSB to MSB (smallest tree first). Both produced correct commitments — the commitment hash function is commutative over the set of non-empty roots. But position calculations disagreed: given a position, the forest and stump mapped it to different trees.

**Impact**: Batch deletion proofs generated by the forest failed verification on the stump, because the stump assigned proof targets to the wrong tree roots.

**Fix**: Align the stump to MSB-to-LSB scanning, matching the forest.

**Lesson**: Commitment equality is necessary but not sufficient. Two implementations of the same accumulator must agree on position semantics, not just on the final hash. Test proof *verification* across implementations, not just commitment equality.

### 6.6 The Proof Hash Ordering Bug

**What happened**: When generating a batch proof for a block, proof hashes from multiple inputs were deduplicated using `std::set` (which sorts by hash value). But the verifier consumed proof hashes sequentially — one per input, in transaction order. The sorted order didn't match the consumption order.

**Impact**: Stateless nodes rejected valid blocks because proof hashes were consumed out of order, producing wrong intermediate hashes.

**Fix**: Store proof hashes in per-target path order (leaf-to-root for each input, in transaction order), then deduplicate without reordering.

**Lesson**: Deduplication must preserve consumption order. When a proof is a sequence of hashes consumed by a sequential algorithm, any reordering is a bug.

### 6.7 The Stale Forest After Restart

**What happened**: On daemon restart, the forest checkpoint loads state from height N (the last persisted checkpoint). The active tip is determined by querying the block index. If the block index query failed for the checkpoint's block hash (e.g., because the index was rebuilt or corrupted), the active tip fell back to genesis (height 0). The system then attempted to replay blocks 1 through N against a forest that was already at state N.

**Impact**: Block 1 validation failed with "root mismatch" because the forest contained state from height N, not the empty state expected at genesis.

**Fix**: After checkpoint loading, detect the mismatch between forest height (derived from leaf count) and active tip height. If they diverge, reset the forest to empty state and replay from genesis. Defense-in-depth: in `ActivateBestChain`, if the forest's leaf count is nonzero but the tip is at genesis, reset before entering the loop.

**Lesson**: Crash recovery must handle *every combination* of partial persistence: forest saved but tip not, tip saved but forest not, both saved but at different heights.

### 6.8 The Silent Root Failure

**What happened**: `ComputeUtreexoRootPure()` could return an error (missing UTXO, null forest pointer). The caller logged a warning and continued, accepting the block without Utreexo verification. This was intended as a "soft" mode during development but was never removed.

**Impact**: Blocks with invalid Utreexo roots could be accepted and propagated, creating chain splits between nodes that computed the root successfully (and rejected the block) and nodes that failed to compute it (and accepted it anyway).

**Fix**: Return false immediately on any root computation failure. No warnings, no fallbacks. The block is rejected.

**Lesson**: In consensus code, there is no such thing as a warning. Every error is either "reject the block" or "crash the node." A middle ground ("log and continue") creates inconsistency between nodes, which is the definition of a consensus bug.

---

## 7. Consensus Invariants

We enforce five runtime invariants, checked in every block validation, even in release builds:

**INV-1: Post-Commit Root Verification.** After the forest clone is move-assigned to the canonical forest, we recompute the commitment and verify it matches the expected value. A mismatch indicates memory corruption or a move-semantic bug.

**INV-2: Deferred SpendCoin.** No UTXO database mutation occurs before Utreexo proof verification passes. Spent outputs are collected in a pending vector and committed only after all proofs succeed.

**INV-3: Double-Spend Tracking.** An `unordered_set<OutPoint>` tracks all outpoints spent in the current block. If an insertion fails (outpoint already present), the block is rejected for double-spending. This provides defense-in-depth alongside the Utreexo proof, which also prevents double-spends by construction.

**INV-4: Post-Restore UTXO Count.** After `DisconnectBlock` restores a UTXO snapshot, the UTXO set size is verified against the snapshot's recorded count. A mismatch indicates a restore bug.

**INV-5: AddCoin-Before-Forest-Commit Ordering.** An assertion verifies that all pending `AddCoin` calls completed before the forest snapshot is committed. This prevents a state where the forest includes a UTXO that the UTXO database does not.

These invariants use a forensic assertion framework that dumps the block hash, height, Utreexo root, UTXO count, and total supply on failure. The assertions are always-on (`CONSENSUS_ASSERT`, not `assert`).

---

## 8. Hardening

### 8.1 Integer Overflow Protection

Utreexo position arithmetic operates on `uint64_t` values that can be close to the maximum. We provide checked arithmetic helpers (`checked_add`, `checked_mul`, `checked_shift_left`) that return errors instead of wrapping on overflow. All position calculations in consensus code use these checked variants.

### 8.2 Proof Size Limits

A malicious peer could send a block with an enormous Utreexo proof to exhaust memory. We enforce:

- `MAX_UTREEXO_PROOF_BYTES = 4 MB` (wire format limit)
- `MAX_PROOF_HASHES = 10,000` (per-block proof hash limit)
- `MAX_PROOF_TARGETS = 100,000` (4x headroom over Bitcoin's ~25K inputs per block)
- `MAX_TREE_HEIGHT = 40` (supports up to 2^40 = 1 trillion UTXOs)

Proof decompression has additional protections: maximum decompressed size of 100 KB with a 100:1 compression ratio limit to prevent decompression bombs.

### 8.3 Proof Compression

For blocks with many inputs, proof data can be significant. We implement two compression layers:

1. **Dictionary encoding**: Deduplicate proof hashes that appear in multiple input proofs (30-40% savings for blocks with 10+ inputs)
2. **zstd compression**: Applied on top of deduplication for proofs larger than 256 bytes (10-20% additional savings)

### 8.4 Authority Model

The Utreexo accumulator is *derived state*: it is a deterministic function of the blockchain history. It does not make fork-choice decisions. The consensus engine decides which chain is best; the accumulator updates *after the fact*. This is enforced architecturally:

- The accumulator cannot reject a block (only the consensus engine can)
- The accumulator cannot choose between competing chain tips
- The accumulator updates are applied after `ConnectBlock` succeeds, not before
- On reorganization, the accumulator is restored to match the new best chain

---

## 9. Performance Characteristics

### 9.1 Storage

| Component | Bitcoin (UTXO DB) | Dinero (Utreexo) |
|-----------|-------------------|-------------------|
| Full node state | ~7 GB (UTXO set) | Forest: O(n) + roots |
| Compact state | Not possible | Stump: ~2 KB (roots only) |
| Per-block undo | UTXO changes | Delta: ~100-500 bytes |
| Proof per block | N/A | O(k * log n) where k = inputs |

### 9.2 Validation

The two-pass algorithm adds O(k * log n) hash computations per block (where k is the number of inputs and n is the UTXO count) for proof generation and verification. For a block with 1,000 inputs and 100 million UTXOs, this is approximately 27,000 SHA-256 operations — negligible compared to signature verification.

### 9.3 Sync

New nodes can begin validating immediately using state transition proofs. Each block is self-contained: it carries the proofs needed to verify the state change. There is no "UTXO download" phase. Initial Block Download requires only the block data and proofs, not a separate UTXO set snapshot.

Parallel block download with sequential validation: up to 16 blocks can be fetched simultaneously, but Utreexo validation must proceed in height order (each block's `roots_before` must match the previous block's `roots_after`).

---

## 10. Wire Protocol

### 10.1 Block Relay

Utreexo-aware blocks are relayed using the `utxoblk` message type with three wire format versions:

- **v1**: Block data + batch proof (legacy)
- **v2**: Block data + batch proof + accumulator_root_after + block_height
- **v3**: Block data + full transition proof (enables stateless verification)

Peers negotiate capability during handshake. Stateless nodes request `MSG_UTREEXO_BLOCK` (inventory type `0x50000002`) in GETDATA messages.

### 10.2 Proof Serving RPCs

Bridge nodes expose RPCs for light clients:

- `getutreexoroots` — Current accumulator roots (for bootstrapping a stump)
- `getutxoproof` — Merkle proof for a specific UTXO (txid + vout)
- `getutreexostats` — Accumulator statistics (leaf count, tree heights, commitment)

---

## 11. Activation

Utreexo is active from the genesis block on all Dinero networks (mainnet, testnet, regtest). There is no activation height, no soft fork, no signaling period. Every block at every height must contain a valid `utreexo_root` commitment.

This is enforced at compile time:

```cpp
constexpr uint32_t UTREEXO_ACTIVATION_HEIGHT_MAINNET = 0;
static_assert(UTREEXO_ACTIVATION_HEIGHT_MAINNET == 0, "Mainnet MUST activate Utreexo at genesis");
```

The advantage of building from scratch: we don't need to maintain backwards compatibility with blocks that predate Utreexo. Every header has the field. Every node verifies it.

---

## 12. Conclusion

Utreexo works. Not in theory, not in simulation, but in production on a live network processing real transactions and real Proof of Work. Every block on Dinero mainnet since genesis has been validated with Utreexo verification. Every node — whether a full bridge node storing the complete forest or a compact state node holding only 2 KB of roots — fully validates every transaction.

The implementation required solving problems that don't exist in the paper: header commitment, mining integration, crash recovery, proof timing, position model alignment, and the subtle interaction between accumulator updates and UTXO database mutations. We found and fixed ten consensus-critical bugs, each of which would have caused a chain split or data corruption in production.

The result is a cryptocurrency where the question "does this coin exist?" can be answered with a cryptographic proof instead of a database lookup. Any computer — a Raspberry Pi, a phone, an old laptop — can fully validate the chain without storing gigabytes of UTXO data. This is what trustless verification looks like when you don't need to trust your storage hardware.

---

## References

1. T. Dryja, "Utreexo: A dynamic hash-based accumulator optimized for the Bitcoin UTXO set," MIT Digital Currency Initiative, 2019.

2. P. Wuille, J. Nick, A. Towns, "BIP 341: Taproot: SegWit version 1 spending rules," Bitcoin Improvement Proposals, 2020.

3. A. Chow, "BIP 86: Key Derivation for Single Key P2TR Outputs," Bitcoin Improvement Proposals, 2021.

4. R. C. Merkle, "A Digital Signature Based on a Conventional Encryption Function," CRYPTO '87, Springer, 1987.

5. J. Benaloh and M. de Mare, "One-Way Accumulators: A Decentralized Alternative to Digital Signatures," EUROCRYPT '93, 1993.

6. B. Bunz, J. Bootle, D. Boneh, A. Poelstra, P. Wuille, G. Maxwell, "Bulletproofs: Short Proofs for Confidential Transactions and More," IEEE S&P, 2018.

---

*Dinero: Real Money for Free People — Genesis Block 2025*
