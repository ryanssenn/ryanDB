# RaftDB

This is a distributed key-value store written in Go. It creates a cluster of nodes that coordinate to maintain a replicated and fault-tolerant database.

The project implements the [Raft](https://raft.github.io/raft.pdf) consensus algorithm which ensures:

- **Linearizability**: operations are applied in a single consistent order across all machines
- **Persistence**: the system recovers all committed data from crashes

# Raft mechanics

The cluster elects a single authoritative leader to manage consistency. All write requests must be processed by this node. If a client sends a request to a follower node, the follower forwards the command to the leader via gRPC. The leader then serializes the command into a log entry and persists it to its local disk.

The leader broadcasts the new log entry to all follower nodes via AppendEntries RPCs amd waits for acknowledgment from a majority of the cluster (e.g., 3 out of 5 nodes) before considering the entry "committed". Only after this consensus is reached does the leader apply the change to its state machine and confirm success to the client.

The system detects failures using a heartbeat mechanism. The leader periodically sends empty messages to prove it is active. If a follower stops receiving these heartbeats, it assumes the leader has failed. To prevent split votes where multiple nodes try to become Leader simultaneously, election timeouts are randomized (e.g., 300ms–450ms). The node that times out first increments its term and requests votes from peers.

Data durability is handled via an append-only log structure. Each node maintains a .rlog file for command history and a .meta file for the current term and voting status. When a node restarts after a crash, it reads the log file from the beginning, replaying every operation to reconstruct the database state exactly as it was before the failure.

<img width="60%" height="60%" alt="image" src="https://github.com/user-attachments/assets/6c7bf543-4297-4383-9367-21f5dbeb4911" />

## Features:
- Linearizable reads and writes (handled by the leader; followers forward client requests)
- Leader election and heartbeats
- Log replication with commit tracking
- Recovery from crashes with persistent logs and metadata
- HTTP API for external clients, gRPC for internal cluster communication

## Running a node:

    ./ryanDB --id=<node_id> --port=<port> --peers=<id=addr,...> [--reset=true|false]

### HTTP API:
- GET /get?key=<key>: Fetch value for key
- GET /put?key=<key>&value=<value>: Store key-value pair
- GET /status: Get node status (id, term, state, leader ID)

# Testing

The project has integration tests that spawn a local N-node cluster to validate distributed consensus mechanics under failure conditions.

```
go test -v ./test
```

#### Scenarios:

- Leader election and re-election after leader crash (TestElection)
- Basic log replication for a single write (TestLogReplication)
- Higher log replication under concurrent writes from random nodes (Test100LogReplication)
- Log durability and state recovery across node restarts (TestLogPersistence)
- Catch-up of a node that was offline and missed replicated logs (TestMissedLogsRecovery)
- Sustained follower churn (random stop/start) under write load (TestFollowerChurnUnderLoad)
- Network partition and healing with cluster-wide state convergence (TestNetworkPartition)

## Future goals:
- Log compaction
- Stress test
- Performance benchmark

# Resources

I developed the understanding needed to implement the Raft paper through McGill’s graduate course [COMP 512 (Distributed Systems)](https://www.mcgill.ca/study/2024-2025/courses/comp-512) taught by professor Bettina Kemme 


