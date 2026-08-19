# etcd & ZooKeeper: Internal Architecture, Primitives, and Distributed Systems Patterns

A deep reference on the two most widely deployed coordination/storage systems for distributed systems: **etcd** (CNCF, written in Go, powers Kubernetes) and **Apache ZooKeeper** (Apache, written in Java, powers Kafka, HBase, Hadoop HA).

Both solve the same fundamental problem — giving a fleet of machines a shared, strongly-consistent, highly-available view of a small amount of critical state — but with very different internal designs and trade-offs.

---

## Part 1: etcd

### 1.1 Overview

etcd is a distributed, reliable key-value store that provides a strongly consistent view of state across a cluster. It is the backbone of Kubernetes (stores all cluster state) and is widely used for service discovery, leader election, distributed locks, and configuration management.

- Written in Go; designed around **gRPC** (v3 API) with an HTTP/gRPC-gateway for REST-style clients.
- Cluster of `N` members runs the **Raft consensus algorithm** — every member holds a full copy of the data.
- Serves from memory backed by a persistent storage engine (bbolt B+tree); reads/writes go through the Raft log.

### 1.2 Internal Architecture

#### Consensus: Raft

etcd embeds the Raft protocol (via the standalone `go.etcd.io/raft/v3` library, also used by CockroachDB and others):

- **Leader election**: members exchange heartbeats (default 100 ms interval); followers elect a new leader via randomized election timeouts (default 1000 ms) when the leader fails. Only one leader exists at a time; a leader steps down automatically if it loses quorum.
- **Log replication**: every mutation becomes a Raft log entry, replicated to followers and committed once a **quorum** (⌊N/2⌋ + 1) acknowledges it.
- **Quorum math**: a cluster of N members tolerates (N−1)/2 permanent failures:
  - 3 members → tolerate 1 failure
  - 5 members → tolerate 2 failures
- **Membership changes** use joint consensus (`ConfChangeV2`) — a configuration change takes effect only when its entry is *applied*; only one config change may be pending at a time. **Learners** (non-voting members) can be added to catch up safely before promotion.
- **ReadIndex optimization**: the leader serves linearizable reads without appending to the log (ReadIndex + applied-index check), and followers can serve them by forwarding a ReadIndex request. Serializable reads skip this entirely (any member, possibly stale).

#### Storage Layers

The data directory (`--data-dir`) contains three artifacts:

| Artifact | Path | Purpose |
|---|---|---|
| Backend store | `member/snap/db` | bbolt B+tree of all applied key-value data + `consistent_index` |
| Write-ahead log | `member/wal/*.wal` | Raft log of every proposal; **all entries fsynced before commit** |
| Snapshots | `member/snap/*.snap`, `*.snap.db` | Periodic bbolt snapshots (every 100,000 applied entries) used to catch up lagging replicas; WAL segments are purged, keeping the last 5 |

- **MVCC model**: etcd v3 is multiversion. Every mutating operation increments a global **revision**; values are stored keyed by `(key, revision)`. Clients can read any historical revision, and watches replay events from a starting revision.
- **Compaction** discards revisions older than a given `compact revision`; **defrag** reclaims bbolt space left fragmented by compaction. Without compaction the database grows without bound and the cluster triggers `alarm NOSPACE` (default 2 GiB quota) halting all writes.

#### Request Path & API

Six gRPC services (defined in `api/etcdserverpb/rpc.proto`):

- **KV**: `Range`, `Put`, `DeleteRange`, `Txn`, `Compact` (plus streaming `RangeStream`)
- **Watch**: bidirectional stream; one RPC multiplexes multiple key ranges, supports historical replay from a revision, progress bookmarks, and server-side event filtering
- **Lease**: `LeaseGrant`, `LeaseRevoke`, `LeaseKeepAlive`, `TimeToLive`
- **Cluster**: membership management (add/remove member, promote learner, transfer leader)
- **Maintenance**: `Snapshot`, `Defrag`, `Alarm`, `Status`, `HashKV`
- **Auth**: users, roles, RBAC permission grants on key ranges

Every RPC is also exposed as REST-JSON via the gRPC gateway (`POST /v3/kv/put`, etc.). Clients (`etcdctl`, `clientv3`) talk gRPC with load-balancing over member endpoints.

### 1.3 Core Primitives

| Primitive | Description |
|---|---|
| **Keys** | Flat binary keyspace (no directories — `/a/b` is a single key). Range queries with `range_end` implement prefix scans. Values up to 1.5 MiB (default `--max-request-bytes`). |
| **Revisions** | Global monotonically increasing revision per mutation; the basis of MVCC reads and watch history. |
| **Leases** | TTL objects. Keys attach to a lease; when the lease expires (or is revoked), all attached keys are deleted atomically. Keepalives stream over gRPC. This replaced v2's per-key TTL. |
| **Watches** | Push-based event streams (`Put`, `Delete` events with revisions). Replayable from any past revision until compaction. Watchers are cheap: ~350 bytes per watched key, ~18 kB per watch stream. |
| **Transactions** | `Txn` = conditional multi-key transaction: `compare (version / create-revision / modify-revision / value) → then-branch / else-branch`. All writes in one txn share a single revision. Up to 128 ops per txn. This is the CAS/CAD replacement from the v2 API. |
| **Election service** | `Campaign` / `Proclaim` / `Resign` — a lease-based distributed mutex for leader election. |
| **Concurrency package** | `NewMutex` (distributed lock), `NewSemaphore`, `NewSTM` (software transactional memory built on optimistic Txn retries) in `clientv3/concurrency`. |

### 1.4 Solving Distributed Systems Problems with etcd

**Service discovery / registration.** Register a service by writing a key bound to a lease; the lease keepalive proves liveness. When the service dies (or is partitioned away), the lease expires and the key disappears — automatic deregistration with no manual cleanup. Clients watch the prefix to learn membership changes in real time. (This is exactly how etcd itself bootstraps, and how many microservice registries work.)

**Leader election.** Two idiomatic patterns:
1. **Election service**: `Campaign` acquires leadership on a key; the holder keeps the session lease alive; `Resign` or lease expiry hands it over. Followers watch the key to detect leadership loss.
2. **Manual pattern**: use a `Txn` to atomically write a `promote` key only if it doesn't exist (or version matches), backed by a lease — a compare-and-swap race that guarantees a single winner.

**Distributed locks & coordination.** `clientv3/concurrency.NewMutex` creates a lock key with a lease; the lock auto-releases on holder death (lease expiry). `NewSTM` provides optimistic transactions that retry on conflict — a CAS loop over Txn compares — useful for read-modify-write workflows (e.g., atomic counters, inventory).

**Configuration management.** Store config under a key/prefix; consumers watch and receive change events; version-based Txn compares prevent lost updates.

**Kubernetes (the flagship deployment).** kube-apiserver uses etcd as its *only* datastore, via the v3 API:
- All resources live under `/registry` (e.g., `/registry/pods/default/my-pod`).
- The apiserver does **not** hammer etcd directly: each resource type has a **Cacher** (watch cache) — a reflector opens a WATCH on etcd, events flow into a ring buffer, and client LIST/WATCH are served from the cache, dramatically reducing etcd load.
- Watch-based informers give controllers their event-driven model.
- Large clusters are advised to run a separate etcd for Event objects.

### 1.5 Typical Setup

**Clustering.** 3–5 members is standard (5 recommended for production headroom). Bootstrap options:
- **Static**: `--initial-cluster` with peer URLs, unique `--initial-cluster-token`
- **etcd discovery service**: register via `https://discovery.etcd.io/new?size=3`
- **DNS discovery**: `--discovery-srv` with SRV records

Ports: **client 2379, peer 2380**.

**Timeouts & tuning.** Heartbeat 100 ms, election timeout 1000 ms (must be ≥10× RTT between members; bounded at 50 s for geo-distributed clusters). All members must use identical values.

**Hardware.** 2–4 cores / 8 GB RAM typical (8–16 cores, 16–64 GB for heavy watch loads or millions of keys). **Fast SSD is the single most critical factor** — etcd is fsync-latency sensitive; slow disks cause missed heartbeats and spurious leader loss.

**Security.** TLS for client and peer channels (per-member certs signed by a cluster CA, or `--auto-tls`); RBAC with users/roles and read/write permissions on key ranges.

**Backup & restore.** `etcdctl snapshot save` (or copy `member/snap/db`); `etcdutl snapshot restore` creates a *new logical cluster* (member/cluster IDs are overwritten). For Kubernetes, restore with `--bump-revision` (~1e9) and `--mark-compacted` so informer watch caches are invalidated — restoring to an old revision without this silently breaks controllers.

**Operational hygiene.** Auto-compaction (periodic or revision-based) + periodic defrag; rolling upgrades one member at a time (same-minor downgrade supported). Deployment options: standalone binary, embedded in Go apps (`embed` package), Kubernetes StatefulSet.

### 1.6 Typical Scale

**Official throughput numbers** (3-member cluster, 8 vCPU / 16 GB / SSD, 8-byte keys, 256-byte values):

| Scenario | Write QPS | Median latency |
|---|---|---|
| 1 connection, 1 client (leader) | 583 | 1.6 ms |
| 100 conns, 1000 clients (leader) | 44,341 | 22 ms |
| 100 conns, 1000 clients (all 3 members) | 50,104 | 20 ms |

- Reads: serializable reads are the low-latency path (local, no consensus round-trip); linearizable reads cost a ReadIndex round-trip.
- **Watch capacity**: ~17 kB per connection, ~18 kB per watch stream, ~350 bytes per watched key; design goals of O(10k) clients, O(100k) streams, O(10M) watchings within a few GB RAM.

**Limits & bottlenecks.**

| Limit | Value |
|---|---|
| Storage quota (default) | 2 GiB (`--quota-backend-bytes`); 8 GiB suggested ceiling |
| Max request size | 1.5 MiB (`--max-request-bytes`) |
| Max txn ops | 128 |
| Max WAL entry size | ~10 MB (hardcoded WAL decoder limit — practical cap per transaction) |
| Snapshot interval | every 100,000 applied entries |

**Bottlenecks:** (1) fsync latency on the leader — every proposal hits disk before commit; (2) network RTT to quorum — each write is one Raft round-trip; (3) watch fan-out — many watchers multiply memory/event dispatch; (4) compaction lag — long-lived revision history (Kubernetes keeps history for watches) balloons disk; (5) quorum loss = no writes, possible loss of recent writes.

---

## Part 2: Apache ZooKeeper

### 2.1 Overview

Apache ZooKeeper is a distributed coordination service providing a hierarchical, in-memory data tree (znodes) with linearizable writes, per-client ordered reads, sessions, watches, and ACLs. It is the metadata backbone of Kafka, HBase, Hadoop (HDFS/YARN HA), Solr, and countless internal systems.

- Written in Java; clients in Java, C, Python, Go, etc.
- Ensemble of servers runs the **ZAB protocol** (ZooKeeper Atomic Broadcast) — a primary-backup consensus protocol distinct from Raft.
- The entire data tree lives **in memory**; durability comes from a transaction log + periodic fuzzy snapshots.

### 2.2 Internal Architecture

#### Consensus: ZAB (ZooKeeper Atomic Broadcast)

ZAB is a crash-recovery atomic broadcast protocol (Junqueira, Reed & Serafini, DSN 2011) that provides **primary-order (PO) atomic broadcast**: a single primary orders all state changes, and every replica delivers a consistent prefix of that sequence.

Four phases:
1. **Election (Phase 0)**: servers elect a prospective leader (via Fast Leader Election, below).
2. **Discovery (Phase 1)**: followers send their last-promised epoch (`CEPOCH`); the leader proposes a new epoch (`NEWEPOCH`); followers ack with their history. The leader picks the follower with the highest `(epoch, zxid)` as the initial history.
3. **Synchronization (Phase 2)**: leader proposes `NEWLEADER`; followers ack after accepting the initial history; once a quorum acks, the leader broadcasts `COMMIT-LD` and is *established* — only then may it propose new transactions.
4. **Broadcast (Phase 3)**: leader proposes (`PROPOSE` with a zxid), followers log + fsync + `ACK`, leader sends `COMMIT` once a quorum acks. Delivery follows zxid order. Any timeout returns a server to election.

Key concepts:
- **zxid**: 64-bit = high 32 bits **epoch** (increments per leader) + low 32 bits **counter**. Global total order of all writes.
- **Leader activation**: a leader isn't "established" until NEWLEADER commits — this is what makes split-brain writes impossible (a quorum is required to activate).
- ZAB differs from Raft: it separates election from epoch-establishing synchronization, pipelines multiple outstanding proposals, and treats the proposal stream holistically rather than per-proposal.

**Leader election: FastLeaderElection.** Peers in `LOOKING` state exchange `NOTIFICATION` messages (sid, epoch, zxid, state) over a dedicated election port (e.g., 3888). A peer votes for the server with the highest `(epoch, zxid)`, tie-broken by server id (sid). A vote for self wins when a **quorum** of notifications agree. FLE deliberately elects the server with the most recent history, minimizing recovery work.

#### Roles & Data Path

| Role | Votes | Serves reads/writes | Notes |
|---|---|---|---|
| **Leader** | Yes | Yes | Only proposer; also acts as a follower |
| **Follower** | Yes | Yes | Acks proposals, fsyncs before acking |
| **Observer** | No | Reads only (and forwards writes) | Non-voting learner (3.4+); scales reads without growing the voting quorum |

**Request pipeline** (server-side processors): `PrepRequestProcessor` (validate, check session/ACL, assign zxid, transform request→transaction) → `ProposalRequestProcessor` (leader: propose; follower: forward) → `SyncRequestProcessor` (leader: write txn log + fsync, trigger snapshots) → `CommitProcessor` (order commits) → `FinalRequestProcessor` (apply to DataTree, fire watches, respond). **Reads short-circuit on the local server** — they never touch ZAB (hence cheap and horizontally scalable, but potentially stale; `sync` before a read approximates freshness).

**Storage.** In-memory `DataTree` of `DataNode`s (each: data byte[], Stat, children set, ACL list; plus ephemeral session→path maps). Durability:
- **Transaction log**: sequential WAL appends in 64 MB preallocated blocks; **every server fsyncs before ACKing** a proposal.
- **Snapshots**: periodic "fuzzy" snapshots (taken concurrently with writes; triggered at `snapCount` ≈ 100,000 txns with random jitter so servers don't snapshot simultaneously, or `snapSizeLimitInKb` ≈ 4 GB).
- `commitLogCount` (default 500) keeps recent committed txns in memory for fast diff-syncs to lagging followers.

**Sessions.** Each client holds a 64-bit session id + password; the leader's `SessionTrackerImpl` tracks global sessions. Timeout is negotiated within [2×tickTime, 20×tickTime]; clients ping ~every timeout/3. **Session expiry deletes all of the client's ephemeral nodes and fires watches** — the liveness primitive. Since 3.5.0, *local sessions* (tracked only by the connected server) avoid quorum cost for session creation. On leader failover, session timers restart on the new leader.

**Watches.** Server-side `WatchManager` keeps path→watchers and watcher→paths maps (optimized bit-set implementation since 3.6.0). Client-side, watches are partitioned into data/exist/child watches and dispatched serially by the EventThread. Watch events are **one-shot**, ordered, and delivered before the client sees the triggering change.

### 2.3 Core Primitives

| Primitive | Description |
|---|---|
| **Znodes** | Hierarchical tree (path-based, like a filesystem). Four node types: **persistent**, **ephemeral** (auto-deleted when creating session ends; cannot have children), **sequential** (parent-scoped monotonic zero-padded counter, e.g. `0000000001`), and combinations (ephemeral-sequential). 3.5.3+ adds **container** nodes (auto-deleted when last child is removed) and **TTL** nodes (opt-in). Data limit 1 MB per znode (`jute.maxbuffer`). |
| **Sessions** | Liveness anchor: ephemeral nodes + session-scoped guarantees; reconnect within timeout preserves ephemerals; `SessionMovedException` guards against session hijacking. |
| **Watches** | One-shot change notifications (`exists`/`getData` → data watches; `getChildren` → child watches). Events: NodeCreated/Deleted/DataChanged/ChildrenChanged. 3.6.0+ adds persistent and recursive watches (`addWatch`). |
| **ACLs** | Per-znode permission lists (CREATE, READ, WRITE, DELETE, ADMIN) with schemes: `world:anyone`, `auth`, `digest` (user:sha1-pass), `ip:addr/bits`, `x509`. Pluggable authentication providers; SASL/Kerberos supported. |
| **multi()** | Atomic batch of create/delete/setData/**check** (version-verify) ops — the foundation of complex recipes (3.4+). |
| **Optimistic concurrency** | `stat.version` (data), `cversion` (children), `aversion` (ACL) increment on every change; `setData(path, data, version)` / `delete(path, version)` act as compare-and-set — mismatched version fails the write. |
| **zxid ordering** | Global total write order exposed to clients. |

### 2.4 Solving Distributed Systems Problems with ZooKeeper

**Leader election (the canonical recipe).** Each contender creates `ELECTION/guid-n_` with `SEQUENCE|EPHEMERAL`; the smallest sequence number wins. The naive "everyone watches the smallest node" approach causes the **herd effect** (a notification storm when the leader dies); the fix is that **each contender watches only its immediate predecessor**, so exactly one client wakes per deletion. GUIDs in the node name handle the ambiguous-failure case (create succeeded but response lost).

**Group membership.** Members create ephemeral nodes under a group node; liveness is the session — on crash or partition, the node vanishes automatically. This is exactly how Kafka brokers (`/brokers/ids/<id>`) and HBase region servers (`/hbase/rs`) register themselves.

**Distributed locks.** Create `SEQUENCE|EPHEMERAL` nodes; the lowest sequence number holds the lock; each waiter watches only the next-lowest node (one watcher per node — no herd effect, no polling). Unlock = delete the node. Read/write (shared/exclusive) and revocable variants exist. Apache Curator packages these as `InterProcessMutex` (fair, reentrant), `InterProcessReadWriteLock`, `InterProcessSemaphoreMutex`, plus `LeaderLatch`/`LeaderSelector` for election.

**Service discovery.** Services register ephemeral znodes with host:port payloads; consumers watch the parent and rebuild client lists on change. Pinterest ran a fleet-wide pattern where a daemon syncs a serverset znode to a local file so apps never touch ZK directly (a ZK outage went unnoticed overnight).

**Configuration management.** Store config in a znode; consumers watch + version-check with `setData(path, data, version)` to prevent lost updates. Facebook's Configerator distributed config to millions of servers via ZK until watch storms forced them to build a separate distribution layer — a cautionary tale about ZK as a *data* channel rather than a *notification* channel.

**Barriers & queues.** Recipes for single/double barriers (enter/leave with a `ready` child), distributed queues (`SEQUENCE|EPHEMERAL`, consumers take lowest numbers), priority queues, and 2PC.

**Ecosystem usage — the real-world proof.**
- **Kafka**: ZK is the metadata store of truth — broker registration (ephemeral), controller election (race to create ephemeral `/controller` with `/controller_epoch` fencing), partition leader/ISR state (`/brokers/topics/<topic>/partitions/<p>/state`). ZK bottlenecks (controller failover reloads all metadata, ISR write amplification) motivated KIP-500/KRaft — Kafka 4.0 removed ZK entirely.
- **HBase**: master election (`/hbase/master` ephemeral; backups block on watches until the active master's node disappears), region server liveness, region assignment, splitwal, replication state.
- **Hadoop HA**: `ZKFailoverController` (`ActiveStandbyElector`) for HDFS NameNode and YARN ResourceManager active/standby election.

### 2.5 Typical Setup

**Ensemble.** Odd number of **voting** servers: 3 for most production (tolerates 1 failure), 5 for maintenance headroom (tolerates 2), 7 rarely (writes degrade as voters grow). Quorum = ⌊N/2⌋ + 1; every pair of quorums intersects. Each server has a `myid` file (1–255) in `dataDir`; `server.X=host:2888:3888` configures quorum + election ports (`:observer` suffix marks observers; 3.6.0+ supports multiple addresses per server).

**Key `zoo.cfg` settings.** `tickTime=2000` (base time unit, ms), `initLimit=10` ticks (follower connection+sync window), `syncLimit=5` ticks (follower sync/processing stall), `clientPort=2181`, `dataLogDir` (separate device for txn log — **the single biggest performance lever**), `snapCount=100000`, `autopurge.snapRetainCount=3` / `autopurge.purgeInterval` (purge is **disabled by default**), `maxClientCnxns=60`, `globalOutstandingLimit=1000`, `leaderServes=yes`.

**Client config.** Connect string `host1:2181,host2:2181,host3:2181[/chroot]`; chroot scopes a client to a subtree (multi-tenancy); clients pick a random server and fail over on disconnect, reconnecting with session id/password.

**Observers.** Use for: reads over WAN, scaling read/watch capacity without enlarging the voting quorum, tolerating unreliable links. With `observerMasterPort`, followers can host observers — fleets in the hundreds.

**Network partition.** The minority side cannot vote/commit → serves no writes (optionally stale reads with `readonlymode.enabled`); its sessions expire and ephemerals are deleted by the majority; a leader partitioned away is re-elected. Split-brain writes are impossible because leader activation requires a quorum.

**Security & ops.** Kerberos/SASL (JAAS) and TLS for client-server and quorum links; 4-letter-word commands (`conf`, `ruok`, `srvr`, `stat`, `mntr`, `wchs`, `dump`, `isro`, ...) gated behind `4lw.commands.whitelist` (default: only `srvr`); HTTP AdminServer (3.5+, port 8080). Backup = snapshot + txn-log copies. Rolling upgrades supported.

**Kafka deployments.** ZK runs as a **dedicated, separate cluster** (3 or 5 nodes), not colocated with Kafka — "often just as many ZooKeeper nodes as Kafka nodes."

### 2.6 Typical Scale

**Official benchmarks** (ZooKeeper paper, USENIX ATC '10, 1 KB payloads, 2010-era hardware):

| Ensemble size | Reads/sec | Writes/sec |
|---|---|---|
| 3 servers | 87,000 | 21,000 |
| 5 servers | 165,000 | 18,000 |
| 7 servers | 257,000 | 14,000 |
| 13 servers | 460,000 | 8,000 |

- "Tens to hundreds of thousands of ops/sec" at 2:1–100:1 read:write ratios; ~3× Chubby's throughput; ~1.2 ms average latency (3 servers).
- **Modern numbers**: ~950k reads/s and ~17.5k–28k writes/s on 5 modern nodes (NIO + CommitProcessor rework; 3.8/3.9 show ~19–25% better write throughput than 3.5).

**Scaling rules & bottlenecks.**

| Constraint | Reality |
|---|---|
| **Writes** | Capped by single-writer consensus + fsync: every txn is fsynced by leader *and* followers before ack. Mitigations: dedicated log device, `flushDelay`/`maxBatchSize` batching (~50 fsyncs/s from hundreds), batch commits (3.6+), sharding across ensembles. |
| **Reads** | Scale linearly by distributing clients across followers/observers — this is what made ZK 3× Chubby. But >7–10 *voting* members degrades writes; add observers instead. |
| **Memory** | The entire tree must fit in RAM. Practical scale: GBs of heap, millions of znodes (Kafka's 200k partitions ≈ 1–2M znodes), ≤1 MB data per znode, hundreds of bytes overhead per znode. No hard znode-count limit — heap/snapshot/sync time bound it. |
| **Watch storms** | Hot znodes + herd-effect watchers cause read avalanches; Facebook measured ~2,500 subscribers per distributor saturating a 10 Gb/s NIC. Mitigate with predecessor-watching recipes, `WatchManagerOptimized` (3.6), connection/request throttling. |
| **Slow followers** | Quorum waits for the slowest acker — lagging followers stall commits. Mitigations: `syncEnabled`, `commitLogCount`, diff-syncs, snapshot/sync lock-contention removal (3.9.4). |

**Notable recent improvements (3.6→3.9):** audit logs, server-side connection/request/large-request throttling, optimized watch manager, digest-based data-consistency checks, multi-address quorum links, persistent recursive watches, Prometheus metrics, quota enforcement, snapshot compression, admin-server snapshot/restore API, watch events carrying triggering zxid.

---

## Summary: etcd vs ZooKeeper at a Glance

| | **etcd** | **ZooKeeper** |
|---|---|---|
| **Consensus** | Raft (leader-based log replication) | ZAB (primary-order atomic broadcast) |
| **Data model** | Flat binary KV with MVCC revisions | Hierarchical znode tree (persistent/ephemeral/sequential) |
| **Storage** | bbolt B+tree on disk, MVCC history | Entire tree in memory; txn log + fuzzy snapshots |
| **API** | gRPC (v3) + REST gateway | Custom client protocol (Java/C/Python/Go) |
| **Liveness primitive** | Leases + keepalives | Sessions + ephemeral nodes |
| **Change notification** | Replayable watch streams (survive disconnects) | One-shot watches (must re-register; client must re-watch) |
| **Atomicity** | Multi-key conditional transactions (Txn, 128 ops) | multi() batches with version checks |
| **Concurrency control** | Revisions + Txn compares | zxid + stat.version CAS |
| **Write path** | One Raft round-trip, leader fsync | Leader + follower fsync before ack |
| **Read scaling** | Serializable reads on any member; linearizable via ReadIndex | Reads on any server/observer (stale but ordered); `sync` for freshness |
| **Typical cluster** | 3–5 members | 3–5 voting servers (+ observers) |
| **Typical write scale** | ~50k writes/s (3 members, SSD) | ~17–28k writes/s (5 nodes); more with batching |
| **Typical read scale** | Memory-bound; millions of watches supported | ~950k reads/s; scales with followers/observers |
| **Flagship user** | Kubernetes (sole datastore) | Kafka (until KRaft), HBase, Hadoop HA |
| **Failure tolerance** | Quorum ⌊N/2⌋+1; auto leader election | Quorum ⌊N/2⌋+1; FLE + leader activation |
| **Best suited for** | Key-value state with watch semantics, gRPC-native ecosystems, Go services | Coordination recipes (election/locks/membership), Java ecosystems, hierarchical data |

**Choosing between them:** both provide the same core guarantees (linearizable writes, quorum-based HA, liveness-based membership, watch-style notifications). Prefer **etcd** for new systems — especially in Go/Kubernetes ecosystems — for its modern gRPC API, MVCC transactions, replayable watches, and simpler operations. Prefer **ZooKeeper** when you need hierarchical organization, mature battle-tested recipes (via Curator), or must interoperate with the Java big-data ecosystem (Kafka pre-KRaft, HBase, Hadoop HA). The fundamental design lesson from both: coordination state must stay *small and in memory*, consensus bottlenecks at the single-writer + fsync layer, and liveness-based primitives (leases/ephemeral nodes) are the key to automatic failure detection.

---

*Compiled from official documentation and papers: etcd.io docs (v3.8), the etcd and raft GitHub repos, Kubernetes docs; the ZAB paper (DSN 2011), the ZooKeeper paper (USENIX ATC '10), zookeeper.apache.org internals/programmer/administrator guides, Kafka/HBase/Hadoop docs, and engineering posts from Pinterest, Facebook, and Confluent.*