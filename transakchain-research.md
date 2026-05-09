# TransakChain: A Cardano-Anchored L2 Protocol with Self-Indexing Data Availability

**Research Draft v0.1**
**Status:** Conceptual / Pre-specification
**Authors:** TransakChain Research

---

## Abstract

TransakChain is a Layer 2 protocol anchored to Cardano's L1, designed around three core principles: **eliminating mandatory concurrent database infrastructure**, **self-indexing transactions via on-chain datums**, and **deterministic eUTxO replication** without a trusted sequencer. This document formalizes the architecture derived through protocol design research, covering the data availability layer, ledger state management, proof systems, consensus, and sharding strategy. The protocol draws on Cardano's native Blake2b hashing, eUTxO determinism, and Plutus validator guarantees as its cryptographic backbone.

---

## Table of Contents

1. [Motivation & Design Philosophy](#1-motivation--design-philosophy)
2. [Scaling Metrics](#2-scaling-metrics)
3. [Data Availability Layer](#3-data-availability-layer)
4. [Self-Indexing Transaction Datums](#4-self-indexing-transaction-datums)
5. [Ledger State as UTXO Linked Structure](#5-ledger-state-as-utxo-linked-structure)
6. [Core Data Structures](#6-core-data-structures)
7. [Transaction Validity — Replicating Cardano eUTxO](#7-transaction-validity--replicating-cardano-eutxo)
8. [Zero-Knowledge Proofs — Surgical Application](#8-zero-knowledge-proofs--surgical-application)
9. [Consensus Design](#9-consensus-design)
10. [Optional DB as Materialized View](#10-optional-db-as-materialized-view)
11. [Sharding for Parallelism](#11-sharding-for-parallelism)
12. [Research Agenda & Open Problems](#12-research-agenda--open-problems)
13. [Data Structures & Algorithms Reference](#13-data-structures--algorithms-reference)

---

## 1. Motivation & Design Philosophy

### 1.1 The Problem with Conventional L2s

Most L2 protocols inherit a fundamental assumption: a **concurrently running database layer** is required to maintain queryable state. This creates infrastructure that is:

- Expensive to operate continuously
- A centralization pressure point (who runs the DB?)
- A corruption/sync risk (DB can diverge from on-chain truth)
- A trust assumption hidden beneath the protocol

TransakChain challenges this assumption directly.

### 1.2 Core Design Thesis

> A blockchain ledger does not require a persistent concurrent database if transactions are self-describing, state is committed to L1 via cryptographic proofs, and queries reconstruct state from verifiable on-chain records.

The goal is a protocol where:

- **L1 (Cardano) is the canonical source of truth** — always
- **Off-chain infrastructure is optional and reconstructible** — never a single point of failure
- **Queries are trustless** — any result can be verified against an on-chain root
- **No sequencer is required** — eUTxO ordering is enforced by the UTXO chain itself

### 1.3 Relationship to Existing Work

| Prior Art | Similarity | TransakChain Difference |
|---|---|---|
| ZK Rollups | Batch + proof to L1 | No sequencer, eUTxO model |
| Validium | Off-chain DA, on-chain proof | DB is optional not required |
| Plasma | Periodic batch commits | No mass exit problem (MMR proofs) |
| Celestia | DA layer separation | Index embedded in tx datum |
| Mina Protocol | ZK-compressed state | Cardano L1 anchor, not standalone |

---

## 2. Scaling Metrics

Any protocol claiming scalability must be measured against a well-defined set of metrics. TransakChain adopts the following as primary benchmarks.

### 2.1 Throughput & Latency

| Metric | Definition | TransakChain Target |
|---|---|---|
| **TPS (sustained)** | Real-world txs/sec under load (not theoretical max) | Defined per shard |
| **Time-to-Finality** | Time until tx is cryptographically irreversible | ~15 minutes (L1 window) |
| **Block/Batch Time** | Frequency of state commitment to L1 | Configurable window |
| **Proof Generation Time** | Time to produce ZK validity proof | Per circuit design |

### 2.2 Cost Efficiency

| Metric | Definition |
|---|---|
| **Fee per Transaction** | Economic cost under load |
| **Fee Volatility** | Spike behavior during congestion |
| **Compression Ratio** | Datum batch efficiency before L1 posting |
| **L1 Calldata Cost** | Bytes posted per batch to Cardano |

### 2.3 Network & State Health

| Metric | Definition |
|---|---|
| **State Growth Rate** | How fast on-chain state (MMR size) grows |
| **Node Sync Time** | Time to reconstruct full state from L1 |
| **Shard Imbalance** | Load distribution across address-space shards |
| **Nullifier Set Size** | Growth rate of spent UTXO commitments |

### 2.4 The Scalability Trilemma Lens

All design decisions are evaluated against the tradeoff triangle:

```
         Scalability
              /\
             /  \
            /    \
  Security ——————— Decentralization
```

TransakChain prioritizes **Security + Decentralization** at the base layer (L1 Cardano), and **Scalability** at L2 — accepting that L2 finality is probabilistic during the batch window.

---

## 3. Data Availability Layer

### 3.1 The No-Persistent-DB Principle

TransakChain eliminates the requirement for a continuously running database by separating concerns into three distinct temporal phases:

```
Phase 1 — COLLECTION (0 to window close):
  Raw transactions received by temporary node runners
  Full datum stored in-memory / ephemeral storage
  No persistence commitment made

Phase 2 — COMMITMENT (at window close):
  Transactions batched
  blake2b(datum) committed to L1 UTXO (Cardano)
  MMR peaks updated on-chain
  ZK validity proof posted

Phase 3 — DISCARD (after finality confirmation):
  Temporary node runners discard raw tx data
  On-chain commitment is the sole source of truth
  State reconstructible from L1 proofs alone
```

### 3.2 Why Contention Is Not a Problem

A critical design question: does the single UTXO holding MMR peaks become a bottleneck?

**No — for the following reasons:**

1. The UTXO is only spent **once per batch window** (e.g., every 15 minutes), not once per transaction
2. Within the window, transactions accumulate off-chain freely with zero on-chain interaction
3. The UTXO update is a **single atomic batch operation**, not per-tx
4. Sharding (Section 11) parallelizes across multiple shard MMR UTXOs for further throughput

```
Window model:
  [tx_1, tx_2, ... tx_N] → collected off-chain for 15 min
                         → single UTXO spend at window close
                         → peaks UTXO updated with new MMR root
  
  On-chain operations per window: 1
  Off-chain txs per window: unbounded
```

The 15-minute window provides sufficient time for Cardano's probabilistic finality to converge before the next window opens, giving a meaningful finality guarantee on the L1 anchor.

### 3.3 Temporary Node Runner Incentives

Temporary node runners store raw transaction data during the collection phase. Incentive design:

- Runners earn a share of batch fees proportional to uptime during the window
- Cryptographic commitment to availability: runners sign a **data availability attestation** before window close
- If insufficient attestations are collected, the batch window extends
- Post-commitment, runners are cryptographically released from storage obligation

---

## 4. Self-Indexing Transaction Datums

### 4.1 Core Concept

Instead of maintaining a separate index database, TransakChain embeds index metadata **inside each transaction datum**. The index travels with the data — it can never drift out of sync because it is part of the data.

This mirrors the architecture of **Apache Kafka's offset model** and **LSM Tree (Log-Structured Merge Tree)** storage engines — applied to blockchain transaction ordering.

### 4.2 Datum Schema

```typescript
type TxIndexDatum = {
  // === Position Metadata ===
  mmrLeafIndex: number;         // position in the MMR leaf set
  batchId: string;              // which L1 batch this belongs to
  batchSlot: number;            // Cardano slot of the batch commitment
  shardId: number;              // which address-space shard

  // === Search Keys (queryable off-chain) ===
  fromAddressHash: string;      // blake2b(address).slice(0, 8) — 8 bytes
  toAddressHash: string;        // blake2b(address).slice(0, 8)
  assetIds: number[];           // pointers into asset registry (not full IDs)
  amountBucket: number;         // Math.floor(Math.log2(amount)) — 1 byte

  // === Proof Anchor ===
  mmrProof: Blake2bHash[];      // inclusion proof path to MMR peak
  nullifierCommitment: string;  // blake2b(utxoRef + spenderSecret) — for spent check
}
```

### 4.3 Why This Eliminates the Index DB Problem

Traditional architecture:
```
Write → update index separately
Read  → consult index → fetch data
Problem: index can corrupt, drift, or require expensive reindex jobs
```

TransakChain datum approach:
```
Write → index is part of the datum
Read  → scan datum metadata, no separate index lookup
Property: index corruption is impossible — the index IS the data
```

### 4.4 Query Path Using Self-Indexing Datums

```
User query: "What is my balance for address X?"

Step 1: Fetch MMR peaks UTXO from Cardano L1
        → get current MMR root and leaf count

Step 2: Request batch headers from DA nodes or L1 calldata
        → filter by batchSlot range (temporal pruning)

Step 3: Within candidate batches, scan datum metadata
        → filter where fromAddressHash or toAddressHash matches blake2b(X)[0:8]

Step 4: For matched datums, fetch full leaf data
        → verify MMR inclusion proof against known root

Step 5: Check nullifier SMT
        → confirm which matched UTXOs remain unspent

Step 6: Sum unspent UTXO values → verified balance
        → no trust in query node required
```

### 4.5 Bloom Filters for Batch Pre-filtering

Each batch carries a compact **Bloom filter** over the address hashes it contains. A querying client can check the filter before fetching full datum data:

```
Does batch_N contain address X?
  → Check bloom(batch_N, blake2b(X)[0:8])
  → False negative: impossible
  → False positive: ~1% (fetch batch, find no match — acceptable)
  → Avoid fetching: ~99% of irrelevant batches skipped
```

This reduces query network overhead significantly for sparse address queries.

---

## 5. Ledger State as UTXO Linked Structure

### 5.1 Sequencer-Free Ordering

TransakChain achieves transaction ordering without a trusted sequencer by using the **UTXO chain itself as the ordering mechanism**.

The Cardano eUTxO model provides a natural guarantee: a UTXO can only be spent once. A chain of UTXOs where each spends the previous head creates a **tamper-evident, totally ordered linked structure** with no external coordinator.

```
Genesis UTXO
    ↓
Batch UTXO [1]  →  MMR root_1, nullifiers_1, batchSlot_1
    ↓
Batch UTXO [2]  →  MMR root_2, nullifiers_2, batchSlot_2
    ↓
Batch UTXO [N]  →  MMR root_N, nullifiers_N, batchSlot_N
         ↑
     current head
```

The ordering index (N) in each datum is a **soft hint** for navigation — not a trust anchor. The actual ordering guarantee comes from the UTXO spend chain. If the index is wrong, the proof path to the MMR root will fail — making index manipulation self-defeating.

### 5.2 On-Chain UTXO Payload (Minimized)

To minimize L1 fees, the on-chain UTXO stores only what is cryptographically necessary:

```typescript
type BatchHeadUTXO = {
  mmrPeaks: Blake2bHash[];      // compact — handful of 32-byte hashes
  nullifierSMTRoot: Blake2bHash; // 32 bytes
  leafCount: number;            // total leaves ever appended
  batchSlot: number;            // Cardano slot
  zkProofHash: Blake2bHash;     // hash of ZK validity proof (proof stored off-chain)
  validatorHash: Blake2bHash;   // enforces correct datum schema on spend
}
```

Full batch data (individual datums, ZK proofs) are stored in the DA layer and referenced by hash — never stored verbatim on-chain.

### 5.3 Validator Logic

The Cardano Plutus validator on the batch UTXO enforces:

```
On every UTXO spend (batch window close), check:
  ✅ blake2b(submittedDatum) === storedDatumHash
  ✅ MMR append is structurally valid (peaks correctly updated)
  ✅ Nullifier commitments are non-duplicated
  ✅ New UTXO correctly references previous head (linked list integrity)
  ✅ ZK proof hash is present (proof verified off-chain by consensus)
  ✅ Datum schema matches expected TxIndexDatum format
```

---

## 6. Core Data Structures

### 6.1 Merkle Mountain Range (MMR)

The primary append-only authenticated data structure for transaction history.

**Structure:**
```
MMR after 7 leaves:

       /\              /\
      /  \            /  \
     /    \          /    \
    0  1  2  3      4  5   [6]

Peaks: [root_0_3, root_4_5, leaf_6]
All peaks stored in a single on-chain UTXO
```

**Properties:**
- Append-only — no rebalancing, no deletion
- O(log n) inclusion proofs
- Proof size grows logarithmically with history
- Only peaks array needs to be on-chain (not full tree)
- New leaf appended by updating at most log(n) peaks

**Proof generation:**
```typescript
// To prove leaf at index i:
// 1. Identify which subtree contains i
// 2. Collect sibling hashes up to subtree root
// 3. Verify subtree root is in peaks array
// Proof size: O(log n) hashes
```

### 6.2 Sparse Merkle Tree (SMT) — Nullifier Set

Used for the spent UTXO nullifier set. Supports both inclusion AND non-inclusion proofs — critical for proving a UTXO has not yet been spent.

**Key:** `nullifier = blake2b(utxoRef + spenderSecret)` — 256-bit key
**Value:** `1` if spent, empty otherwise

**Properties:**
- Fixed depth (256 levels for 256-bit keys)
- Non-inclusion proof: path of empty siblings to root
- Updates are O(log n) = O(256) = constant for 256-bit keys
- Root stored in batch head UTXO (32 bytes)

### 6.3 Global MMR of Shard Roots

When sharding is active, a global MMR holds shard roots:

```
globalRoot = MMR(shardRoot_A, shardRoot_B, shardRoot_C, shardRoot_D)
```

Stored in a single global head UTXO on Cardano L1. Cross-shard state is provable against this root.

### 6.4 Address-Keyed State (Optional SMT Layer)

For direct balance lookups without scanning, an additional SMT keyed by address maps current UTXO state:

```
addressSMT[blake2b(address)] = currentUTXOSet
```

This enables O(log n) balance queries without scanning batch history — at the cost of maintaining the SMT root as additional on-chain state.

---

## 7. Transaction Validity — Replicating Cardano eUTxO

### 7.1 The Four Validity Conditions

TransakChain replicates Cardano's eUTxO validity model exactly:

```
A transaction is valid if and only if:
  1. All input UTXOs exist (inclusion proof in MMR)
  2. All input UTXOs are unspent (non-inclusion in nullifier SMT)
  3. Value is conserved: Σ inputs = Σ outputs + fees
  4. All signatures are valid (ed25519)
  + 5. Validator scripts pass: redeemer satisfies datum (Plutus/Aiken)
```

### 7.2 Verification Without a Sequencer

Each condition is verifiable without trusting any coordinator:

| Condition | Verification Method | Trust Requirement |
|---|---|---|
| UTXO exists | MMR inclusion proof vs on-chain root | None — cryptographic |
| UTXO unspent | SMT non-inclusion proof vs on-chain root | None — cryptographic |
| Value conservation | Local arithmetic re-execution | None — deterministic |
| Signature valid | ed25519 verify | None — cryptographic |
| Validator passes | ZK proof of correct execution | None — ZK soundness |

### 7.3 Nullifier Design

```typescript
// Spending a UTXO:
const nullifier = blake2b(
  utxoRef +          // identifies which UTXO
  spenderSecretKey   // proves ownership
)

// Post nullifier to SMT
// Validator checks: nullifier NOT in SMT → UTXO unspent ✅
// If nullifier already exists → double spend rejected ❌

// Privacy property:
// The nullifier reveals THAT a UTXO was spent
// but not WHICH UTXO (if utxoRef is kept private)
// This enables optional privacy-preserving transfers
```

### 7.4 Cross-Shard Transaction Protocol

For transactions spanning two shards (e.g., address in shard A sending to address in shard B):

```
Window N:
  Phase 1 — Burn: Shard A destroys input UTXO
                  Posts receipt = blake2b(outputSpec + windowId)
                  Nullifier added to Shard A SMT

Window N+1:
  Phase 2 — Mint: Shard B verifies receipt against Shard A's posted root
                  Creates output UTXO in Shard B MMR
                  Receipt nullified to prevent double-mint

Result: Atomic across windows, no coordinator needed
Latency: 1 additional window (~15 minutes) for cross-shard txs
```

---

## 8. Zero-Knowledge Proofs — Surgical Application

### 8.1 What the Protocol Solves Without ZK

The MMR + SMT + eUTxO design already provides:
- ✅ Data availability
- ✅ Ordering without sequencer
- ✅ UTXO existence and non-double-spend proofs
- ✅ Tamper-evident history
- ✅ Trustless state reconstruction

### 8.2 Where ZK Is Still Necessary

ZK fills three gaps that cryptographic hashing alone cannot close:

**Gap 1 — Validator Script Correctness**

Without ZK, every node must re-execute every validator script. With ZK:

```
Prover (off-chain):
  runs validator(datum, redeemer)
  generates ZK proof of correct execution

Verifier (on-chain or any node):
  checks proof in O(1)
  does NOT re-execute the script

Benefit: Light clients and DA nodes can verify validity
         without running a full Plutus interpreter
```

**Gap 2 — Recursive State Compression**

```
Proof_N = ZK(Proof_{N-1}, txBatch_N, MMR_root_N)

Each batch proof recursively includes the previous batch's proof.
Result: entire TransakChain history compressible to a single proof.
Light clients verify the whole ledger in milliseconds — regardless of history length.
```

This mirrors the Mina Protocol approach but anchored to Cardano rather than operating as a standalone chain.

**Gap 3 — Optional Privacy**

For private transactions, ZK proofs can prove:
- Value is conserved without revealing amounts (Pedersen commitments + range proofs)
- Sender owns the input UTXO without revealing the UTXO reference
- Validator is satisfied without revealing the redeemer

### 8.3 ZK Proof System Candidates

| System | Proof Size | Verify Time | Trusted Setup | Best For |
|---|---|---|---|---|
| **Groth16** | ~200 bytes | < 1ms | Per-circuit | Production, smallest proof |
| **PLONK** | ~400 bytes | ~2ms | Universal | Flexible circuits |
| **STARKs** | ~50-100KB | ~10ms | None | Quantum resistance |
| **Nova** | Logarithmic | Fast | None | Recursive proofs |
| **Halo2** | ~1-2KB | ~5ms | None | Recursive, used by Zcash |

**Recommended:** PLONK for validator correctness proofs (universal setup, Cardano CIP-0381 compatible). Halo2 or Nova for recursive batch compression.

### 8.4 ZK-Friendly Hashing

Blake2b (used throughout this protocol) is **not ZK-friendly** — it is expensive to represent in arithmetic circuits. For ZK proof generation contexts, use **Poseidon Hash** instead:

```
Poseidon:  ~200 constraints per hash  (ZK-efficient)
Blake2b:   ~20,000 constraints per hash (ZK-expensive)

Strategy:
  On-chain commitments:  blake2b  (Cardano native, fast)
  Inside ZK circuits:    Poseidon (circuit-efficient)
  Bridge: hash_onchain = blake2b(poseidon_tree_root)
```

---

## 9. Consensus Design

### 9.1 Consensus Scope

TransakChain consensus operates at the **batch window level** — not per-transaction. Validators agree on:

1. The set of transactions included in the current window's batch
2. The correctness of the resulting MMR root update
3. The validity of all nullifier additions to the SMT

Per-transaction finality is probabilistic (data availability guarantee). Per-batch finality is deterministic once the L1 UTXO is updated.

### 9.2 HotStuff BFT for Batch Agreement

HotStuff is recommended for batch-level consensus due to:
- **Linear message complexity** — O(n) not O(n²) like PBFT
- **Deterministic finality** — once 2/3 validators agree, the batch is final
- **Leader rotation** — no single sequencer; leadership changes each round
- **Pipeline-friendly** — consecutive batch proposals can overlap

```
HotStuff pipeline for TransakChain:

Prepare  → Leader proposes batch (MMR root + nullifiers + ZK proof hash)
Pre-Vote → Validators verify MMR update validity
Vote     → Validators sign if valid
Commit   → 2/3 threshold reached → batch committed to L1
```

### 9.3 BLS Signature Aggregation for Consensus Compression

Individual validator signatures are aggregated into a single BLS signature before L1 posting:

```
Individual: N validators × 96 bytes per sig = 96N bytes on-chain
Aggregated: 1 BLS aggregate sig = 96 bytes on-chain (regardless of N)

A ZK proof of BLS aggregation reduces this further:
  ZK(2/3 stake signed this batch) = constant size proof
  Verifiable by any light client in O(1)
```

### 9.4 Validator Set

- Validators stake ADA on Cardano L1 as collateral
- Slashing conditions: equivocation (signing two conflicting batches)
- Entry/exit: governed by a Cardano smart contract managing the validator registry UTXO
- Minimum viable validator set: 7 nodes (BFT requires f < n/3 faults)

---

## 10. Optional DB as Materialized View

### 10.1 Architectural Role

The database in TransakChain is **not a protocol requirement** — it is a performance optimization. This is a fundamental departure from conventional L2 designs.

```
Source of truth:    MMR root on Cardano L1        (slow, trustless, permanent)
Performance layer:  Replicated off-chain DB        (fast, trust-minimized, optional)
Relationship:       DB is a materialized view      (always reconstructible from L1)
```

### 10.2 DB as Verifiable Cache

Any DB query result can be challenged:

```
User queries DB node: "What is my balance?"
DB returns: 150 ADA

User verifies (optionally):
  1. Fetch MMR root from Cardano L1
  2. Request MMR inclusion proofs for claimed UTXOs from any node
  3. Check nullifier SMT for each UTXO
  4. Sum → should equal 150 ADA

If DB lied → proof fails → user queries a different DB node
DB nodes have no power to forge state — only to serve it
```

### 10.3 DB Reconstruction

If all DB nodes go offline, the full state is reconstructible:

```
1. Fetch all batch head UTXOs from Cardano L1 (the UTXO chain)
2. For each batch, fetch full datum set from archived DA storage
3. Verify each datum against its MMR inclusion proof
4. Replay UTXO set construction (deterministic — eUTxO is pure)
5. Rebuild DB indices (addressHash → UTXO set)

Result: Full state restored from L1 alone
Time: Proportional to chain history length (one-time cost)
```

### 10.4 Recommended DB Stack

For DB nodes opting into the performance layer:

| Component | Technology | Rationale |
|---|---|---|
| Primary storage | RocksDB (LSM Tree) | Write-optimized for indexing new batches |
| Query layer | PostgreSQL | Complex queries, range scans, joins |
| Cache | Redis | Hot address balance cache |
| Index | B+ Tree on addressHash | O(log n) balance lookups |
| Sync protocol | gRPC streaming | DB nodes sync from each other + L1 |

---

## 11. Sharding for Parallelism

### 11.1 Natural Parallelism in eUTxO

Cardano's eUTxO model provides an inherent parallelism property: **transactions spending different UTXOs have zero dependency on each other** and can be processed, validated, and batched simultaneously.

Sharding makes this parallelism explicit at the protocol level.

### 11.2 Address-Space Sharding (Primary)

```
Shard assignment:
  shardId = blake2b(address)[0] >> (8 - log2(numShards))

Example with 4 shards:
  Address → blake2b → first byte
  0x00 - 0x3F → Shard 0
  0x40 - 0x7F → Shard 1
  0x80 - 0xBF → Shard 2
  0xC0 - 0xFF → Shard 3
```

Each shard maintains its own:
- MMR peaks UTXO (on Cardano L1)
- Nullifier SMT root (on Cardano L1)
- Validator node set
- Batch window (potentially staggered for load smoothing)

### 11.3 Global Root Aggregation

```
After all shards close their windows:
  globalMMR = MMR(shardRoot_0, shardRoot_1, shardRoot_2, shardRoot_3)
  globalRoot UTXO updated on Cardano L1

Cross-shard proofs resolve against the globalRoot
Light clients only need to track globalRoot
```

### 11.4 Block-STM for Intra-Shard Parallelism

Within a single shard's batch window, transactions can be executed in parallel using **Block-STM** (Optimistic Concurrency Control):

```
Phase 1 — Parallel execution:
  All txs in the window executed concurrently
  Each tx speculatively reads UTXO state

Phase 2 — Validation:
  Check for read-write conflicts (tx A read a UTXO that tx B spent)
  Conflicting txs are re-executed sequentially

Phase 3 — Commit:
  Non-conflicting txs committed to MMR in parallel
  Conflicting txs appended after re-execution

Result: Near-linear speedup for non-conflicting tx workloads
```

### 11.5 Cross-Shard Transaction Latency

| Scenario | Latency |
|---|---|
| Same-shard transfer | 1 window (~15 min) |
| Cross-shard transfer | 2 windows (~30 min) |
| Cross-shard DeFi (multi-hop) | N windows for N hops |

Cross-shard latency is the primary scalability tradeoff in the sharded design.

---

## 12. Research Agenda & Open Problems

### 12.1 Immediate (Pre-Specification)

| Problem | Description | Difficulty |
|---|---|---|
| **Nullifier batching** | Efficiently batch nullifier updates into SMT without contention under high load | High |
| **ZK circuit for validator** | Express Plutus/Aiken validator subset as ZK-provable constraints | High |
| **Poseidon ↔ Blake2b bridge** | Efficiently bridge between ZK-internal Poseidon hashes and on-chain Blake2b commitments | Medium |
| **DA attestation protocol** | Formal design of temp node availability attestation and slashing | Medium |

### 12.2 Medium Term (Specification Phase)

| Problem | Description |
|---|---|
| **Recursive proof compression** | Implement recursive PLONK/Halo2 proofs for batch chain compression |
| **Shard rebalancing** | Protocol for resharding when address distribution becomes imbalanced |
| **Economic model** | Fee market design within batch windows; incentive alignment for temp DA nodes |
| **Light client spec** | Minimal data required for a trustless light client to verify balance |

### 12.3 Long Term (Research)

| Problem | Description |
|---|---|
| **Verkle migration** | Migrate from Merkle proofs to Verkle proofs for constant-size inclusions |
| **Privacy layer** | Full Zcash-style shielded pool integrated with eUTxO model |
| **Formal verification** | Formally verify validator logic and MMR append correctness |

---

## 13. Data Structures & Algorithms Reference

### 13.1 Authenticated Data Structures

| Structure | Time Complexity | Proof Size | Use In TransakChain |
|---|---|---|---|
| **MMR** | Append O(log n), Prove O(log n) | O(log n) hashes | Transaction history |
| **Sparse Merkle Tree** | O(log n) all ops | O(log n) | Nullifier set, balance index |
| **Verkle Tree** | O(log n) | O(1) constant | Future migration target |
| **Patricia Merkle Trie** | O(log n) | O(log n) | Reference: Ethereum state |
| **Authenticated Skip List** | O(log n) avg | O(log n) | Alternative to MMR |

### 13.2 Hashing & Commitment Schemes

| Primitive | Properties | Use |
|---|---|---|
| **Blake2b-256** | Fast, Cardano-native, not ZK-friendly | All on-chain commitments |
| **Blake3** | Faster than Blake2b, parallel | Off-chain data hashing |
| **Poseidon** | ZK-friendly (~200 constraints) | Inside ZK circuits |
| **Pedersen Commitments** | Hiding + binding, additively homomorphic | Value hiding |
| **KZG Commitments** | Constant-size proofs, trusted setup | Danksharding-style DA |

### 13.3 ZK Proof Systems

| System | Proof Size | Verify | Setup | Recursion | Use Case |
|---|---|---|---|---|---|
| **Groth16** | ~200B | < 1ms | Per-circuit | With accumulator | Validator proofs (production) |
| **PLONK** | ~400B | ~2ms | Universal | Yes | Flexible validator circuits |
| **STARKs** | ~50-100KB | ~10ms | None | Yes | Quantum-resistant option |
| **Nova** | O(log n) | Fast | None | Native | Recursive batch compression |
| **Halo2** | ~1-2KB | ~5ms | None | Native | Recursive (Zcash-proven) |

### 13.4 Consensus Algorithms

| Algorithm | Message Complexity | Finality | Fault Tolerance | Notes |
|---|---|---|---|---|
| **HotStuff** | O(n) | Deterministic | f < n/3 | Recommended for TransakChain |
| **Tendermint** | O(n²) | Deterministic | f < n/3 | Simpler, higher overhead |
| **PBFT** | O(n²) | Deterministic | f < n/3 | Only viable for small sets |
| **Bullshark** | O(n) | Deterministic | f < n/3 | DAG-based, high throughput |

### 13.5 Parallelism Algorithms

| Algorithm | Use | Key Property |
|---|---|---|
| **Block-STM** | Intra-shard parallel execution | Optimistic, rollback on conflict |
| **Topological Sort (Kahn's)** | Ordering dependent txs in batch | O(V+E), deterministic |
| **Consistent Hashing** | Shard routing | O(log n) lookup, stable under membership changes |
| **Rendezvous Hashing** | Alternative shard routing | Simpler than consistent hashing |

### 13.6 Suggested Reading

**Foundational Papers:**
- *Merkle Mountain Ranges* — Peter Todd (2012)
- *HotStuff: BFT Consensus with Linearity and Responsiveness* — Yin et al. (2019)
- *PLONK: Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge* — Gabizon et al. (2019)
- *Block-STM: Scaling Blockchain Execution by Turning Ordering Curse to a Performance Blessing* — Gelashvili et al. (2022)
- *Nova: Recursive Zero-Knowledge Arguments from Folding Schemes* — Kothapalli et al. (2022)

**Applied References:**
- Cardano CIP-0381: Plutus Support for Pairings (ZK primitives)
- L2Beat.com — Production rollup metrics and architecture comparisons
- Ethereum Research (ethresear.ch) — Danksharding, KZG commitments
- Celestia whitepaper — Data availability sampling

---

## Appendix A: Architecture Summary Diagram

```
┌──────────────────────────────────────────────────────────────────┐
│                        TRANSAKCHAIN L2                           │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │
│  │  Shard A   │  │  Shard B   │  │  Shard C   │  │ Shard D  │  │
│  │  MMR+SMT   │  │  MMR+SMT   │  │  MMR+SMT   │  │ MMR+SMT  │  │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └────┬─────┘  │
│        └───────────────┴──────────────┴────────────────┘        │
│                              ↓                                   │
│                    Global MMR Root UTXO                          │
│              (Block-STM parallel execution per shard)            │
│                              ↓                                   │
│              ┌───────────────────────────────┐                   │
│              │  HotStuff BFT Consensus        │                   │
│              │  BLS Aggregate Signatures      │                   │
│              │  ZK Proof of Batch Validity    │                   │
│              └───────────────┬───────────────┘                   │
│                              ↓                                   │
│              ┌───────────────────────────────┐                   │
│              │  Cardano L1 UTXO Anchor        │                   │
│              │  blake2b commitments           │                   │
│              │  Plutus validator enforcement  │                   │
│              └───────────────────────────────┘                   │
│                                                                  │
│  Query paths:                                                    │
│    Fast:      Optional DB (materialized view, verifiable)        │
│    Trustless: MMR scan + SMT proof (no trust, slower)            │
│    Both produce identical, cryptographically verifiable results  │
└──────────────────────────────────────────────────────────────────┘
```

---

*TransakChain Research — Draft v0.1 — Subject to revision as specification matures*
