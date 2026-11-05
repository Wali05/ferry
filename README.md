# Ferry

A small 3-node replicated message queue in Go, tested by deterministic simulation.

Raft replicates a checksummed write-ahead log. A simulator crashes nodes, partitions the network, drops or duplicates messages, and corrupts the disk. Failures replay from a seed.

Status: early build. Plan and milestones live in `PLAN.md`.
