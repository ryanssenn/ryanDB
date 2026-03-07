# RaftDB

A Distributed, Linearly Consistent Key-Value Store. This system leverages the [Raft](https://raft.github.io/raft.pdf) Consensus Algorithm to maintain strict data consistency across a distributed cluster, ensuring service availability even during node failures or network partitions.

Problems solved:

- Safety over Liveness: Property where a candidate must have a log as up-to-date as the majority to win an election, preventing "stale" leaders from overwriting committed data.

- gRPC Transport Layer: Leveraged gRPC and Protobuf for internal cluster RPCs to minimize serialization overhead and ensure type-safe communication between nodes.

- Concurrency & Locking: Managed complex state transitions (Follower → Candidate → Leader) using Go's sync.Mutex and channel primitives to prevent race conditions during high-frequency heartbeat intervals.

- State Machine Replication (SMR): Architected the separation between the Raft log layer and the underlying state machine, allowing for modular database backends.


# Raft mechanics

The cluster elects a single authoritative leader to manage consistency. All write requests must be processed by this node. If a client sends a request to a follower node, the follower forwards the command to the leader via gRPC. The leader then serializes the command into a log entry and persists it to its local disk.

The leader broadcasts the new log entry to all follower nodes via AppendEntries RPCs amd waits for acknowledgment from a majority of the cluster (e.g., 3 out of 5 nodes) before considering the entry "committed". Only after this consensus is reached does the leader apply the change to its state machine and confirm success to the client.

The system detects failures using a heartbeat mechanism. The leader periodically sends empty messages to prove it is active. If a follower stops receiving these heartbeats, it assumes the leader has failed. To prevent split votes where multiple nodes try to become Leader simultaneously, election timeouts are randomized (e.g., 300ms–450ms). The node that times out first increments its term and requests votes from peers.

Data durability is handled via an append-only log structure. Each node maintains a .rlog file for command history and a .meta file for the current term and voting status. When a node restarts after a crash, it reads the log file from the beginning, replaying every operation to reconstruct the database state exactly as it was before the failure.

<img width="60%" height="60%" alt="image" src="https://github.com/user-attachments/assets/6c7bf543-4297-4383-9367-21f5dbeb4911" />

# Core Features

- Strong Consistency (Linearizability): Guarantees that once a write is acknowledged, all subsequent reads will reflect that write or a later one. Client requests to followers are automatically proxied to the current cluster leader via internal gRPC channels.

- High Availability via Raft Consensus: Implements a robust leader election mechanism with randomized timeouts to ensure the cluster remains operational and elects a new primary node within milliseconds of a failure.

- Durable State Machine Replication: Uses an append-only Write-Ahead Log (WAL) to ensure that every committed transaction is persisted to disk. The system handles full state reconstruction upon node restart using .rlog and .meta recovery files.

- Hybrid Communication Architecture: * Internal (gRPC/Protobuf): Low-latency, type-safe RPCs used for cluster-wide coordination, log replication, and heartbeats.

- External (RESTful HTTP): A simple, language-agnostic API for client operations (GET, PUT) and cluster health monitoring.

- Fault-Tolerant Log Catch-up: Automatically synchronizes lagging or newly joined nodes by identifying log inconsistencies and backfilling missing entries from the leader's log.

## Running a node:

    ./ryanDB --id=<node_id> --port=<port> --peers=<id=addr,...> [--reset=true|false]

### HTTP API:
- GET /get?key=<key>: Fetch value for key
- GET /put?key=<key>&value=<value>: Store key-value pair
- GET /status: Get node status (id, term, state, leader ID)

# Testing

To ensure the robustness of the consensus logic, I built a testing framework that simulates real-world distributed failures:

```
go test -v ./test
```

#### Adversarial Testing & Cluster Resilience:

- Leader Election & Term Consistency (TestElection): Validates the liveness of the cluster by ensuring a new leader is elected within a single randomized election timeout following a primary node failure.

- Linearizable Log Replication (TestLogReplication): Confirms that a single write is correctly replicated and committed to a majority of nodes before being acknowledged.

- Concurrency & High-Throughput Stress (Test100LogReplication): Evaluates the system under a heavy load of concurrent writes from multiple clients, ensuring all nodes reach the same final state machine index without log divergence.

- Durability & Crash Recovery (TestLogPersistence): Verifies the persistence layer by crashing nodes and ensuring they reconstruct their entire state machine from the .rlog and .meta files upon restart.

- Dynamic Catch-up Logic (TestMissedLogsRecovery): Tests the NextIndex and MatchIndex logic by forcing a node offline and ensuring it correctly synchronizes missed entries from the leader once re-connected.

- Availability under High Churn (TestFollowerChurnUnderLoad): Simulates a "flapping" network or unstable hardware by randomly stopping and starting follower nodes under a continuous write load to verify cluster stability.

- Network Partition Resilience (TestNetworkPartition): A critical safety test that simulates a split-brain scenario. It ensures that only the majority partition can commit entries and that the minority partition correctly rolls back uncommitted entries once the network heals.

## Future goals:
- Log Compaction (Snapshotting): Implementation of the Snapshotting mechanism to prevent the WAL (Write Ahead Log) from growing indefinitely and to speed up node recovery.

- Batching & Pipelining: Optimizing AppendEntries to batch multiple log entries into a single RPC to improve throughput.

# Resources

This project was developed as a deep-dive into distributed consensus following the curriculum of McGill University's COMP 512 (Distributed Systems) by Professor Bettina Kemme


