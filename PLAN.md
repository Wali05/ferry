# Ferry: a replicated message queue in Go, tested by deterministic simulation

Working name: **Ferry** (it carries messages across, like Courier, but over a cluster).

## Goal

Wali's Courier is a single-node Go broker with a WAL, gRPC streaming and retries/DLQ.
Ferry is the next step: a small 3-node replicated queue where Raft replicates a
checksummed write-ahead log, and every claim about safety is backed by a
deterministic simulation test that crashes nodes, partitions the network,
drops/duplicates/delays messages and corrupts the disk, then replays any
failure from its seed.

Why this project:
- It's the hard part of the systems Wali already contributes to (QuestDB WAL and
  dedup, TiDB, DataFusion, quix-streams consumers, his DBOS queue-stall fix) and
  continues the failure-testing theme of RecoveryLab.
- Deterministic simulation testing (FoundationDB, TigerBeetle, Antithesis,
  gosim, Polar Signals) is a hot topic on HN in 2026, and S3/Kafka-alternative
  queues are all over r/apachekafka and r/rust. Hiring managers for infra roles
  recognise both.
- Every piece is explainable in an interview: fsync and torn writes, leader
  election, commit index, at-least-once delivery, linearizability checking.

Non-goals for v0.1: Kafka wire protocol, membership changes, multi-partition
sharding, object storage. Honest README says so.

## v0.1 scope (demoable in about two weeks)

- `internal/wal`: segmented, CRC32C-checked log with fsync, crash recovery that
  truncates a torn tail, and an injectable file system so tests can lose
  un-fsynced writes.
- `internal/raft`: Raft core as a pure state machine (no goroutines, no real
  time). Elections, log replication, commit, persistence of term/vote/log.
  Talks to the world through Transport, Clock, Storage interfaces.
- `internal/queue`: the replicated state machine. Topics, enqueue, lease with
  visibility timeout, ack, redelivery, dead-letter after N attempts.
- `cmd/ferry`: run a node; `ferry demo` starts a local 3-node cluster.
  Small HTTP/JSON API (gRPC later if time allows).
- `internal/sim`: deterministic simulator. Fake clock, in-memory network with
  loss/dup/delay/partition, fake disk with crash and lost-write faults. One
  seed fully determines a run. Invariant checks every step:
  one leader per term, committed prefixes agree, no acknowledged enqueue is
  lost, no message acked twice, per-topic FIFO among non-redelivered messages.
- History checker (Porcupine-style) for the client-visible operations.
- Bug zoo: flags that inject real bugs (ack before fsync, voting without
  log check, commit from an old term) so the demo shows the simulator catching
  each one, like RecoveryLab's demo.
- Benchmark: throughput and p50/p99 latency for 1 vs 3 nodes, with and without
  batched fsync, on the box. Report the numbers as measured, nothing more.
- `docs/how-it-works.md` written so Wali can explain every part himself.

## Milestones

1. **M1 Storage**: WAL with segments, CRC, fsync, recovery, fault-injecting FS, tests.
2. **M2 Raft core**: elections + replication + commit, unit tests with a manual network.
3. **M3 Queue on Raft**: queue state machine, HTTP API, `ferry demo` 3-node local cluster.
4. **M4 Simulator**: fake clock/net/disk, chaos runs, invariants, seed replay, `make sim SEEDS=500`.
5. **M5 Checker + bug zoo**: history checker, injected bugs caught with printed seeds.
6. **M6 Benchmark + README + demo GIF**, Claude/Codex review via Engineering Lead, X post draft.

Stretch: snapshots and log compaction, gRPC streaming consumer, a tiny web
timeline that replays a failing seed.

## Rules

- Local commits only until Wali says yes to a public repo (proposed name `Wali05/ferry`).
- Plain commit messages, no AI trailers.
- Keep test runs light on the shared box (cap sim runs, no long benchmarks while others work).

## Status

- 2026-11-05: Plan written, local repo initialised. Waiting for Wali's yes on the
  project and the public repo. Next run: M1.
