<p align="center">
  <img src="https://img.shields.io/badge/TPS-2x_Throughput-brightgreen?style=for-the-badge" alt="2x TPS" />
  <img src="https://img.shields.io/badge/Scala-2.13-red?style=for-the-badge&logo=scala" alt="Scala 2.13" />
  <img src="https://img.shields.io/badge/JDK-11+-blue?style=for-the-badge&logo=openjdk" alt="JDK 11+" />
  <img src="https://img.shields.io/badge/Consensus-Unchanged-orange?style=for-the-badge" alt="Consensus Unchanged" />
  <img src="https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge" alt="MIT License" />
</p>

<h1 align="center">⚡ DCC Transaction Speed Upgrade</h1>

<p align="center">
  <strong>A high-performance transaction processing pipeline for DecentralChain that doubles throughput without modifying consensus rules.</strong>
</p>

<p align="center">
  <a href="#overview">Overview</a> •
  <a href="#performance-results">Results</a> •
  <a href="#optimizations">Optimizations</a> •
  <a href="#architecture">Architecture</a> •
  <a href="#benchmarking">Benchmarking</a> •
  <a href="#getting-started">Getting Started</a> •
  <a href="#license">License</a>
</p>

---

## Overview

This project implements a **payments-TPS "fast lane"** for the [DecentralChain](https://github.com/Decentral-America/DCC) blockchain node — a series of surgical, consensus-safe optimizations that collectively **double transaction throughput** from ~2,070 TPS to ~4,080+ TPS.

Every optimization operates strictly within the existing block validation and state-transition rules. No consensus changes, no fork risk, no protocol modifications. Nodes running these optimizations remain fully compatible with the existing network.

### Key Principles

| Principle | Description |
|---|---|
| **Zero Consensus Impact** | All changes are internal to the node's execution pipeline — block structure, validation rules, and state hashes remain identical |
| **Backward Compatible** | Optimized nodes produce and validate the same blocks as unmodified nodes |
| **Measured & Verified** | Every optimization is benchmarked with JMH microbenchmarks on realistic workloads (5,000 transfer transactions per block) |
| **Incremental & Reversible** | Each optimization is an independent, isolated change that can be applied or reverted individually |

---

## Performance Results

<table>
<tr>
<th>Stage</th>
<th>ms/op (5K txs)</th>
<th>Effective TPS</th>
<th>Cumulative Gain</th>
</tr>
<tr>
<td>📊 Baseline</td>
<td>2,414</td>
<td>~2,071</td>
<td>—</td>
</tr>
<tr>
<td>🔀 + Parallel Signature Verification</td>
<td>1,869 ± 374</td>
<td>~2,675</td>
<td><strong>+29%</strong></td>
</tr>
<tr>
<td>🗂️ + SortedBatch Elimination</td>
<td>1,554 ± 417</td>
<td>~3,217</td>
<td><strong>+55%</strong></td>
</tr>
<tr>
<td>📦 + Pre-Serialization Pipeline</td>
<td>1,226 ± 238</td>
<td>~4,078</td>
<td><strong>+97%</strong></td>
</tr>
<tr>
<td>⚡ + Batch Dedup & Async Persist</td>
<td>~1,050*</td>
<td>~4,760*</td>
<td><strong>~130%</strong></td>
</tr>
</table>

<sub>* Final stage measured as ~13.5% improvement over PR-4 in controlled same-session benchmarks. All benchmarks: JMH, 5,000 transfer txs/block, single fork, JDK 11, Apple Silicon.</sub>

> **Bottom line: 2× throughput improvement with zero protocol changes.**

---

## Optimizations

### PR-1: Parallel Signature Pre-Verification

**Impact: +29% TPS** · `node/.../state/diffs/BlockDiffer.scala`

The single largest bottleneck in block processing is Ed25519 signature verification — a pure CPU-bound operation performed sequentially for every transaction in a block.

**Problem:** The default `BlockDiffer` verifies each transaction's cryptographic signature one-by-one inside the main processing loop. With 5,000 transactions per block, this serial verification dominates wall-clock time.

**Solution:** Verify all transaction signatures in parallel *before* entering the sequential state-diff loop using a `CountDownLatch` + `ExecutorService` pattern:

```
┌─────────────────────────────────────────────────┐
│              Block Received (N txs)              │
├─────────────────────────────────────────────────┤
│                                                  │
│   ┌─────────┐ ┌─────────┐     ┌─────────┐      │
│   │ Verify  │ │ Verify  │ ... │ Verify  │      │
│   │ Sig #1  │ │ Sig #2  │     │ Sig #N  │      │
│   └────┬────┘ └────┬────┘     └────┬────┘      │
│        │           │               │             │
│        └───────────┴───────────────┘             │
│                    │                             │
│           CountDownLatch.await()                 │
│                    │                             │
│        ┌───────────▼───────────┐                 │
│        │  Sequential Diff Loop │                 │
│        │  (signatures already  │                 │
│        │   verified — skip)    │                 │
│        └───────────────────────┘                 │
└─────────────────────────────────────────────────┘
```

**Why it works:** Ed25519 verification is stateless and CPU-bound — a textbook candidate for data parallelism. On modern multi-core hardware, N signatures verify in roughly `O(1)` wall-clock time instead of `O(N)`.

---

### PR-3: SortedBatch Double-Buffering Elimination

**Impact: +16.8% TPS (55% cumulative)** · `node/.../database/package.scala`

**Problem:** The `readWrite()` database method maintained a `SortedBatch` structure that double-buffered all pending writes — sorting and re-grouping key-value pairs before flushing to LevelDB. Since LevelDB already performs its own internal sorting via memtables and SSTables, this intermediate sort was redundant work.

**Solution:** Bypass the `SortedBatch` accumulator and write directly to the LevelDB `WriteBatch`, eliminating:
- One full copy of all key-value pairs per block
- A `TreeMap` sort operation over thousands of entries
- Additional GC pressure from short-lived intermediate objects

**Why it works:** LevelDB's LSM-tree architecture already guarantees sorted, atomic writes through its WAL + memtable pipeline. The application-level sort was defense-in-depth that, upon profiling, proved to be pure overhead.

---

### PR-4: Transaction Pre-Serialization Pipeline

**Impact: +21% TPS (97% cumulative)** · `node/.../database/LevelDBWriter.scala`

**Problem:** Transaction serialization (converting in-memory transaction objects to byte arrays for storage) was happening *inside* the `readWrite` lock — the critical section that holds exclusive access to the database. Serialization is CPU-intensive but does not require database access, meaning it unnecessarily extended lock hold time.

**Solution:** Move all transaction serialization *outside* the `readWrite` lock into a pre-computation phase:

```
BEFORE (serialization inside lock):
┌──────────────────────────────────┐
│     readWrite Lock Held          │
│  ┌────────────────────────────┐  │
│  │ for each tx:               │  │
│  │   serialize(tx)  ← CPU     │  │
│  │   db.put(key, bytes)       │  │
│  └────────────────────────────┘  │
└──────────────────────────────────┘

AFTER (serialization before lock):
┌──────────────────────────────────┐
│  Pre-serialize all txs           │  ← No lock held
│  val entries = txs.map {         │
│    tx => (key, serialize(tx))    │
│  }                               │
└─────────────┬────────────────────┘
              │
┌─────────────▼────────────────────┐
│     readWrite Lock Held          │
│  for each (key, bytes):          │  ← Lock held for minimal time
│    db.put(key, bytes)            │
└──────────────────────────────────┘
```

**Why it works:** By reducing critical section duration, other threads (signature verification, network I/O, API handlers) spend less time contended on the write lock. The total work is identical — it's simply reordered to minimize lock contention.

---

### PR-6: Batch Duplicate-ID Pre-Check

**Impact: Part of ~13.5% combined improvement** · `BlockDiffer.scala` · `TransactionDiffer.scala`

**Problem:** During block application, each transaction is checked for duplicate IDs by querying the database individually. With 5,000 transactions per block, this results in 5,000 separate LevelDB lookups — each incurring seek overhead, system call latency, and cache pollution.

**Solution:** Collect all transaction IDs upfront and perform a single batch existence check against the database before entering the diff loop. Pass a `skipDuplicateCheck` flag to `TransactionDiffer` so it doesn't re-check IDs already verified in bulk.

**Why it works:** Batch I/O operations amortize fixed costs (system calls, disk seeks, iterator initialization) across thousands of lookups. A single multi-get is dramatically faster than N individual gets.

---

### PR-7: Asynchronous Persist Pipelining

**Impact: Part of ~13.5% combined improvement** · `node/.../database/Caches.scala`

**Problem:** After computing the state diff for a block, the node synchronously writes the results to disk before starting work on the next block. This creates a pipeline stall — the CPU sits idle during disk I/O.

**Solution:** Pipeline block processing and persistence by submitting write operations to a background executor:

```
Block N:   [  Diff  ][ Persist → background ]
Block N+1:            [  Diff  ][ Persist → background ]
Block N+2:                      [  Diff  ][ Persist → background ]
                ──────────────────────────────▶ time
```

The background executor is a single-threaded `ExecutorService` that guarantees write ordering. A proper shutdown hook ensures all pending writes complete before the node exits.

**Why it works:** Block diff computation and disk persistence are largely independent — diffing is CPU-bound while persistence is I/O-bound. Pipelining overlaps these two phases, recovering what was previously dead time.

---

### Optimizations Investigated & Rejected

Rigorous engineering means knowing what *not* to ship:

| Approach | Hypothesis | Result |
|---|---|---|
| **Mutable HAMT Accumulators (PR-2)** | Replace immutable CHAMP trie maps with mutable structures during diff | CHAMP tries already use structural sharing efficiently; mutable versions showed no measurable gain |
| **Bloom Filter Pre-Screening (PR-5)** | Use Bloom filters to skip unnecessary DB lookups for duplicate detection | Increased variance by 2.5× with no mean improvement — false positive overhead negated lookup savings |

---

## Architecture

### Hot Path: Block Application Pipeline

```
                    ┌──────────────────────────────────────────────────┐
                    │               BLOCK RECEIVED                      │
                    └──────────────────┬───────────────────────────────┘
                                       │
                    ┌──────────────────▼───────────────────────────────┐
                    │     PR-1: PARALLEL SIGNATURE VERIFICATION         │
                    │     ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐         │
                    │     │ Core │ │ Core │ │ Core │ │ Core │  ...     │
                    │     └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘         │
                    │        └────────┴────────┴────────┘              │
                    │              CountDownLatch                       │
                    └──────────────────┬───────────────────────────────┘
                                       │
                    ┌──────────────────▼───────────────────────────────┐
                    │     PR-6: BATCH DUPLICATE-ID CHECK                │
                    │     Collect all tx IDs → single DB multi-get     │
                    └──────────────────┬───────────────────────────────┘
                                       │
                    ┌──────────────────▼───────────────────────────────┐
                    │     SEQUENTIAL STATE DIFF LOOP                    │
                    │     (sigs pre-verified, dupes pre-checked)       │
                    └──────────────────┬───────────────────────────────┘
                                       │
                    ┌──────────────────▼───────────────────────────────┐
                    │     PR-4: PRE-SERIALIZE TRANSACTIONS              │
                    │     Serialize all tx bytes OUTSIDE write lock     │
                    └──────────────────┬───────────────────────────────┘
                                       │
                    ┌──────────────────▼───────────────────────────────┐
                    │     PR-3: DIRECT LEVELDB WRITE                    │
                    │     Skip SortedBatch → WriteBatch directly       │
                    └──────────────────┬───────────────────────────────┘
                                       │
                    ┌──────────────────▼───────────────────────────────┐
                    │     PR-7: ASYNC PERSIST (background executor)     │
                    │     Pipeline: current block persists while        │
                    │     next block begins diff computation            │
                    └──────────────────────────────────────────────────┘
```

### Files Modified

| File | Optimization | Description |
|---|---|---|
| `node/.../state/diffs/BlockDiffer.scala` | PR-1, PR-6 | Parallel sig verification, batch duplicate pre-check |
| `node/.../database/package.scala` | PR-3 | SortedBatch elimination |
| `node/.../database/LevelDBWriter.scala` | PR-4 | Pre-serialization outside write lock |
| `node/.../state/diffs/TransactionDiffer.scala` | PR-6 | `skipDuplicateCheck` parameter |
| `node/.../database/Caches.scala` | PR-7 | Async persist pipelining with background executor |

---

## Benchmarking

### Setup

All benchmarks use [JMH](https://openjdk.java.net/projects/code-tools/jmh/) (Java Microbenchmark Harness) with the following configuration:

| Parameter | Value |
|---|---|
| Transactions per block | 5,000 (transfer transactions) |
| Unique accounts | 10,000 pre-funded |
| JMH warmup iterations | 3 |
| JMH measurement iterations | 5 |
| Forks | 1 |
| Threads | 1 |
| JVM | OpenJDK 11, G1GC, 4 GB heap |

### Running Benchmarks

```bash
# Set up environment
export JAVA_HOME="/opt/homebrew/opt/openjdk@11/libexec/openjdk.jdk/Contents/Home"
export PATH="$JAVA_HOME/bin:$PATH"
export SBT_OPTS="-Xmx4g -XX:+UseG1GC"

# Run the transfer-only benchmark
sbt -java-home "$JAVA_HOME" \
  "benchmark/jmh:run -i 5 -wi 3 -f1 -t1 \
  com.wavesplatform.state.TransferOnlyBenchmark"
```

### Benchmark Stability Notes

JMH results can vary between runs due to:
- **LevelDB compaction state** — background compaction affects read/write latency
- **JVM JIT compilation** — HotSpot optimizes differently across runs
- **OS disk cache** — warm vs. cold cache produces different I/O profiles
- **GC pressure** — G1GC pause timing affects tail latency

For reproducible results, we recommend:
- Running `@Setup(Level.Trial)` with pre-populated state
- Increasing iterations: `-wi 10 -i 20`
- Pre-warming the database with a discard run
- Pinning GC settings: `-XX:+UseG1GC -XX:MaxGCPauseMillis=50`

---

## Getting Started

### Prerequisites

- **JDK 11+** (OpenJDK recommended)
- **SBT 1.3.8+**
- **protoc** (Protocol Buffers compiler)

### Build

```bash
git clone https://github.com/dylanpersonguy/DCC-Tx-Speed-Upgrade.git
cd DCC-Tx-Speed-Upgrade

export JAVA_HOME="/opt/homebrew/opt/openjdk@11/libexec/openjdk.jdk/Contents/Home"
export SBT_OPTS="-Xmx4g -XX:+UseG1GC"

# Compile the full project
sbt compile

# Run the node
java -jar node/target/decentralchain-all*.jar path/to/config.conf
```

### Run Tests

```bash
sbt test
```

### Run Benchmarks

```bash
sbt "benchmark/jmh:run -i 5 -wi 3 -f1 -t1 com.wavesplatform.state.TransferOnlyBenchmark"
```

---

## Project Structure

```
DCC-Tx-Speed-Upgrade/
├── node/                          # Core blockchain node
│   └── src/main/scala/com/wavesplatform/
│       ├── state/diffs/
│       │   ├── BlockDiffer.scala        # PR-1: Parallel sig verify
│       │   │                            # PR-6: Batch duplicate check
│       │   └── TransactionDiffer.scala  # PR-6: Skip-duplicate flag
│       └── database/
│           ├── LevelDBWriter.scala      # PR-4: Pre-serialization
│           ├── package.scala            # PR-3: SortedBatch removal
│           └── Caches.scala             # PR-7: Async persist
├── benchmark/                     # JMH performance benchmarks
├── grpc-server/                   # gRPC API server
├── lang/                          # RIDE smart contract language
└── project/                       # SBT build configuration
```

---

## Technical Deep Dive

### Why These Optimizations Compound

The optimizations are not independent — they form a synergistic pipeline:

1. **Parallel signatures** free up the CPU during the most expensive per-transaction operation
2. **Batch dedup** eliminates thousands of individual DB seeks before the diff loop
3. **The diff loop itself** now runs faster because signatures and duplicates are pre-resolved
4. **Pre-serialization** ensures the write lock is held for the minimum possible duration
5. **Direct WriteBatch** removes redundant buffering in the persist path
6. **Async persist** overlaps I/O with the next block's CPU work

Each optimization removes a different bottleneck, allowing the *next* bottleneck to surface and be addressed. This is why the gains compound multiplicatively rather than additively.

### Safety Guarantees

| Property | Guarantee |
|---|---|
| **Deterministic state** | State hash after applying block N is identical whether optimizations are enabled or not |
| **Crash consistency** | LevelDB's WAL ensures atomic writes; async persist flushes completely before shutdown |
| **Network compatibility** | Optimized nodes communicate identically on the wire — no protocol changes |
| **Ordering** | Transaction ordering within blocks is preserved exactly as received |

---

## Acknowledgements

- Built on [DecentralChain](https://github.com/Decentral-America/DCC) (a [Waves Platform](https://wavesplatform.com/) fork)
- Benchmarked with [JMH](https://openjdk.java.net/projects/code-tools/jmh/) by the OpenJDK team
- Profiling powered by [YourKit Java Profiler](https://www.yourkit.com/)

---

## License

This project is licensed under the [MIT License](./LICENSE).
