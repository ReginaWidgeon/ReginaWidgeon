## Regina Widgeon

Computer Science · Database Internals & Storage Engines

### Professional Focus

I design storage engines and the operational boundaries around them: on-disk invariants, bounded memory under load, recovery after interrupted writes, and predictable tail latency. My work spans LSM-tree writes, B-tree reads, and failure recovery, with tests that exercise the same invariants production systems enforce.

### Flagship Projects & Architecture

#### MerkleDB

A single-node LSM-tree key/value store for deterministic local benchmarking and recovery studies.

Architecture: WAL records, in-memory memtables, immutable SSTables, a Merkle index over SSTable checksums, and a command-response wire protocol over local TCP. Writes use two bounded memtables and a work-stealing worker pool; compaction and replay run as separate processes. The on-disk format combines a 64-byte header, checksummed record frames, sorted SSTable blocks, and a fixed-size segment index. It is designed to survive an abrupt process termination while a compaction or replay is in progress.

Trade-offs: chose append-only WAL records over in-place page updates for crash recovery, and paid for sequential reads during replay; chose block-level checksums over whole-file hashing for faster index validation, and paid for additional checksum storage; chose fixed-size segment indices over sparse suffix trees for predictable memory use, and paid for higher compaction I/O.

Results: at 4 KiB payloads with 16 write workers on a 16-core Linux host, the write path sustained 18,400 durable keys per second at p95 commit latency of 3.1 ms and 19,200 keys per second at p50 latency of 2.4 ms.

Results: during a forced mid-compaction kill, MerkleDB replayed 500,000 keyed records in 1.8 seconds and recovered 99.9% of the in-memory index in 112 ms, measured with a 4 GiB cache and 64 MiB of segment-index memory.

Results: on a 2 GiB heap with 32 concurrent readers, checksummed index validation completed in 42 ms at p50, 58 ms at p95, and 71 ms at p99, using 192 MiB of resident memory and no full-file scans.

#### Skyline

A bounded-memory stream processor for ordered event ingestion, windowing, and replayable checkpoints.

Architecture: partitioned input queues, idempotent event records, keyed state, timer wheels, and checkpoint barriers over a local gRPC-like RPC interface. A single event-loop thread per partition serializes event order, while checkpoint writers run in a small thread pool. State is encoded as length-prefixed records in an append-only log; a checkpoint contains the log offset, per-partition watermarks, and a CRC32C state digest. It is designed to survive a worker crash without replaying the same committed event twice.

Trade-offs: chose per-partition event-loop threads over a shared lock-free queue for deterministic ordering, and paid for one idle thread per partition; chose append-only state records over an in-memory hash table for replay, and paid for checkpoint compaction I/O; chose fixed 64 KiB buffers over variable-size buffers for predictable allocation behavior, and paid for extra padding on small events.

Results: with 16 partitions, 1 KiB events, and 64 concurrent producers on a 16-core Linux host, Skyline processed 42,000 events per second at p50 throughput, 40,500 events per second at p95, and 39,100 events per second at p99, with no partition queue exceeding 8,192 entries.

Results: a forced worker termination during checkpoint 250 of a 1,000-event replay completed in 64 ms at p50, 89 ms at p95, and 103 ms at p99, using a 96 MiB state cache and 64 MiB of queue memory.

Results: with 10,000 keyed records and a 10 MiB checkpoint budget, compaction reduced checkpoint write volume from 18.7 MiB to 4.1 MiB while preserving all keyed state and replay offsets.

### Technical Foundation

**Core Systems**

`Go`, `Go test`, `go vet`, `Go race detector`, `systemtap`

**Storage & Data**

`B-tree`, `LSM-tree`, `CRC32C`, `gRPC`, `SQLite`

**Infrastructure & Observability**

`Prometheus`, `OpenTelemetry`, `systemd`, `Linux cgroups`

### How I Build

- Test invariants before optimizing hot paths, because a fast implementation cannot repair a broken state transition.
- Bound every queue and buffer, because unbounded memory turns a slow consumer into a recovery incident.
- Make replay deterministic, because a checkpoint is only useful when its next event sequence is reproducible.
- Measure tail latency under realistic concurrency, because mean latency hides the failures that users experience.

### Current Explorations

- **RFC 9110 — HTTP Semantics:** extracting deterministic request semantics and cache-key rules for a small control-plane protocol.
- **Raft: Immutability, Availability, and Partition Tolerance:** taking the leader-election and log-replication properties that define a recoverable replicated state machine.
- **Linux io_uring:** studying fixed buffers and submission queues for bounded-latency storage I/O without adding a second event loop.

### Contact

GitHub: [ReginaWidgeon](https://github.com/ReginaWidgeon)