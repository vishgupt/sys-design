# The Raft Consensus Algorithm: A Deep Dive for Working Engineers

Raft is the consensus algorithm that powers etcd (Kubernetes), CockroachDB, TiKV, Consul, and many other systems. It is the most widely deployed consensus protocol in the world, and it was designed from day one to be *understandable* — the authors' stated goal was that "it should be possible for the average reader to understand the algorithm better than it is possible for them to understand Paxos."

This document explains Raft from the ground up: what problem it solves, how it works, why it's safe, and how it behaves in production. It keeps the math to intuition and works through concrete examples.

---

## Table of Contents

1. [The Problem: Replicated State Machines](#1-the-problem-replicated-state-machines)
2. [Raft at a Glance](#2-raft-at-a-glance)
3. [Terms: Raft's Logical Clock](#3-terms-raffts-logical-clock)
4. [Leader Election](#4-leader-election)
5. [Log Replication](#5-log-replication)
6. [Safety: Why Raft Never Goes Wrong](#6-safety-why-raft-never-goes-wrong)
7. [Handling Failures: Partitions, Crashes, Reordering](#7-handling-failures-partitions-crashes-reordering)
8. [Log Compaction: Snapshots](#8-log-compaction-snapshots)
9. [Membership Changes](#9-membership-changes)
10. [Client Interaction and Linearizable Reads](#10-client-interaction-and-linearizable-reads)
11. [Production Extensions](#11-production-extensions)
12. [Performance and Engineering Details](#12-performance-and-engineering-details)
13. [Raft in the Real World](#13-raft-in-the-real-world)
14. [Common Bugs and Misconceptions](#14-common-bugs-and-misconceptions)
15. [Glossary](#15-glossary)
16. [Further Reading](#16-further-reading)

---

## 1. The Problem: Replicated State Machines

### The setup

Imagine you run a service that must never lose data — say, a configuration store for your entire company. You run it on one machine. The machine dies. Data gone. Bad.

So you run it on three machines. Now a new question appears: **how do the three machines agree?**

- They all hold the same data (a *replicated* copy).
- A client writes `key = "red"`. What if the write reaches machine A but not machines B and C? Now they disagree.
- What if two clients write different values at the same time? Which one "wins"?
- What if the machines can't talk to each other (a *network partition*)? Do they still answer clients? If they all answer, you get different answers from different machines — *split brain*.

This "how do machines agree" problem is the **consensus problem**, and it is much harder than it looks. Messages can be lost, delayed, duplicated, or reordered. Machines can crash mid-operation. Clocks can be wrong. Machines can be partitioned from each other. A correct consensus algorithm must still return *correct* results through all of this.

### The replicated state machine model

The standard way to build a fault-tolerant service is the **replicated state machine**:

```
            client requests
                  │
   ┌──────────────┼──────────────┐
   ▼              ▼              ▼
┌───────┐    ┌───────┐       ┌───────┐
│  log  │    │  log  │  ...  │  log  │      ← every server keeps the SAME ordered log
├───────┤    ├───────┤       ├───────┤
│  FSM  │    │  FSM  │       │  FSM  │      ← every server applies the log in the same order
└───────┘    └───────┘       └───────┘
```

If every server applies the **same commands in the same order** to a **deterministic state machine**, every server ends up in the same state — even if some servers were temporarily behind. The only hard part is keeping the logs identical.

> **The whole job of Raft:** keep the logs on all servers as close to identical as possible, and decide *definitively* which entries are safe to apply ("committed") — without any single point of failure.

### What Raft promises (and what it doesn't)

Raft provides **safety**: under any non-Byzantine failure (crashes, partitions, message loss/dup/reorder), it will never return a wrong result. There is always at most one leader, and every committed entry is applied identically on every server.

Raft provides **availability** as long as *any majority* of servers is up and can reach each other. A 3-server cluster survives 1 failure. A 5-server cluster survives 2. That's the fundamental trade — safety is guaranteed unconditionally; availability is majority-conditional.

Raft does **not** rely on timing for safety: slow machines, bad clocks, and extreme message delays can make the cluster slow or unavailable, but never *wrong*. (If this surprises you, hold that thought — it will make sense by Section 6.)

### Why not just use Paxos?

Paxos was the classic consensus algorithm, but even experts found it notoriously hard to understand and implement correctly (Lamport's own first paper was framed as a joke with a fabricated Greek parable, because nobody was reading the real paper). Raft's authors identified a real problem: **understandability is a practical requirement**. Hard-to-understand protocols get implemented wrong. Raft's design goal was to be decomposable into independently understandable pieces and to reduce the state space ("logs can't have holes" etc.). The paper includes a user study: 33 students scored higher on Raft questions (25.7/60) than Paxos (20.8/60).

---

## 2. Raft at a Glance

Raft's key design decision: **elect a strong leader, let the leader do all the work, and make everyone else obey it.**

```
   Leader            Followers
   ┌────┐            ┌────┐  ┌────┐  ┌────┐
   │    │◄───────────►│    │  │    │  │    │
   │    │◄───────────►│    │  │    │  │    │
   └────┘            └────┘  └────┘  └────┘
```

The algorithm breaks into four separable subproblems:

| Subproblem | What it does | Section |
|---|---|---|
| **Leader election** | Pick a leader when the old one fails | §4 |
| **Log replication** | Leader pushes entries to followers, commits on majority | §5 |
| **Safety** | Guarantee the wrong things can never happen | §6 |
| **Membership changes** | Add/remove servers safely | §9 |

(Log compaction is a fifth piece, §8.)

### The three roles

Every server is always in exactly one of three states:

| Role | Job |
|---|---|
| **Leader** | Receives all client requests, appends to its log, replicates to followers, decides what's committed. Sends periodic *heartbeats* (empty AppendEntries) to keep authority. |
| **Follower** | Passive. Never initiates anything. Responds to RPCs from leaders and candidates. A follower that hears nothing becomes a candidate. |
| **Candidate** | A follower that started an election. Votes for itself, asks others for votes. Wins → leader. Loses or times out → back to follower/candidate. |

```
       times out, starts election          wins election
Follower ─────────────────────────────► Candidate ─────────────────► Leader
   ▲                                      │  │
   │                                      │  │ discovers higher term
   │       discovers valid leader         │  └──────────────────────┘
   └──────────────────────────────────────┘          (back to follower)
```

### The core invariants (the whole algorithm on one page)

Everything that follows is in service of these rules:

1. **One leader per term** (election safety). Servers vote for at most one candidate per term; a leader needs a majority of votes.
2. **The leader's log is the source of truth.** Followers overwrite conflicting entries with the leader's versions.
3. **An entry is committed when it's on a majority of servers** — with one crucial exception (the "current term" rule, §6.3) — and committed entries are never lost.
4. **Only an "up-to-date" candidate gets votes** — this single rule is what makes all the safety properties provable (§6.2).
5. **Election timeouts are randomized** — this is what makes elections terminate quickly.

---

## 3. Terms: Raft's "Logical Clock"

Raft divides time into **terms** — consecutive integers that act as a logical clock:

```
term 1        term 2             term 3 (no leader — split vote)
   │              │                   │          term 4
   ▼              ▼                   ▼            ▼
──────────────────┬──────────────────┬──────────┬───────────────►  time
                  │                  │          │
              election            election   election
```

Key rules:

- Every server tracks `currentTerm`, starting at 0, increasing monotonically. **Every RPC carries the sender's term.**
- If a server sees a *higher* term in any RPC (request or response), it immediately updates its `currentTerm` and reverts to **follower** (if it was a leader or candidate, it steps down).
- If a server receives a request with a *stale* (lower) term, it **rejects** it.
- Each term starts with an election. A term can end with **no leader** (e.g., a split vote where nobody reaches a majority).

**Why terms matter:** they let every server know who is "old news." When a server sees a message with a higher term, it knows someone newer is in charge and gets out of the way. This is the mechanism that prevents two leaders from coexisting across time — no two leaders can be elected in the same term (§6.1), and terms only increase.

**Persistent state** (written to disk *before* responding to any RPC): `currentTerm`, `votedFor` (who this server voted for in the current term), and the `log[]`. This is what lets a crashed server recover without voting twice.

---

## 4. Leader Election

### How an election starts

A follower is happy as long as it hears from the leader. The leader sends **heartbeats** (empty AppendEntries) on a fixed interval (e.g., every 100 ms in etcd).

Each follower has an **election timer**. If it expires without hearing anything valid from a leader (or a candidate it voted for), the follower assumes the leader is dead and becomes a **candidate**:

```
1. increment currentTerm          (now it's a NEW term — this is important!)
2. vote for itself                (votedFor = its own id)
3. reset its election timer
4. send RequestVote RPC to all other servers in parallel
```

### The critical detail: randomized timeouts

Election timeouts are chosen **randomly** from a fixed range (the paper recommends 150–300 ms; etcd uses ~1000 ms with 100 ms heartbeats).

Why random? Consider three followers all hearing silence at the same moment. If their timers expired at the same time, all three would start elections simultaneously. Each gets its own vote, and each gets one vote from a third party — nobody reaches a majority of 2. **Split vote, election failed, repeat.** With *randomized* timers, one server's timer fires first, and it usually reaches a majority before the others even start campaigning.

```
T1 timer fires first → T1 becomes candidate, term 2
      T1 asks S2, S3 for votes
      S2, S3 vote for T1 (their timers haven't fired yet — no one else is campaigning)
      T1 wins with 2 votes = majority of 3   ✓ leader
```

Randomization makes repeated split votes extremely unlikely: even if a split vote happens, the next round's random timers will almost certainly stagger differently.

### How a server decides to grant a vote

A receiver of `RequestVote(term, candidateId, lastLogIndex, lastLogTerm)` grants a vote **only if**:

1. The candidate's term is at least the receiver's `currentTerm` — **and**
2. The receiver hasn't already voted in this term (first-come-first-served: one vote per server per term) — **and**
3. **The candidate's log is at least as up-to-date as the receiver's.**

The third rule is the **election restriction**, and it's the heart of Raft's safety. "Up-to-date" is defined precisely:

> Compare the *last* log entries. Whichever log has the **higher term** in its last entry is more up-to-date. If the last terms are equal, the **longer** log is more up-to-date.

This rule says: *don't vote for a candidate that might be missing committed entries.* A server that holds every committed entry (the leader's log) will never vote for a lagging candidate, so a lagging server can never become leader. We'll see why this single rule makes Raft provably safe in §6.2.

### Three possible outcomes

A candidate wins if it gets votes from a **majority of the full cluster** (2 of 3, 3 of 5):

| Outcome | What happens |
|---|---|
| **Wins majority** | Becomes leader. Immediately sends empty AppendEntries heartbeats to all followers (to assert authority fast, before their timers fire). |
| **Another server claims to be leader** | If it receives AppendEntries with `term ≥ its own term`, it accepts the leader and reverts to follower. |
| **Election timeout passes with no majority** | Starts a new election with an incremented term, resets its randomized timer. |

### What happens during a partition?

Say we have servers A, B, C, and A is leader. The network splits: A alone on one side; B and C on the other.

- **B and C** stop hearing heartbeats → one of them becomes candidate → wins votes from the other → **B becomes leader in a new term** on the majority side. Writes flow to B and C, commit with their majority of 2.
- **A** is alone. It keeps receiving client writes, but can't reach a majority — no quorum — so **nothing it does can ever commit**. Its elections fail (it can't get votes).
- Clients that reach A can't make progress — but crucially, **A never gives wrong answers**: no entry becomes committed without a majority.

What if A and B are partitioned, C alone? C can't form a majority of 2 (with 3 servers, quorum = 2, C needs one more) → no progress on either side, no split brain. With 4 servers: A+B vs C+D — *each* side has a quorum of 2! Two leaders can be elected. That's why **Raft clusters should have an odd number of servers** (3, 5, 7...): it guarantees any partition has at most one majority side.

### The old leader comes back

When the partition heals, the old leader A sees B's messages with a **higher term** and immediately reverts to follower. Its stale log entries get overwritten by B's (via log replication, §5). No ambiguity, no argument — terms settle everything.

---

## 5. Log Replication

### Anatomy of the log

Every server has a log of entries. Each entry has three fields:

```
        index:    1       2       3       4       5
        ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
Leader  │ 1 set │ │ 2 set │ │ 3 set │ │ 4 del │ │ 5 get │
        │ color=│ │ x=42  │ │ x=43  │ │ key=y │ │ x=44  │
        │ red   │ │       │ │       │ │       │ │       │
        │term 1 │ │term 1 │ │term 1 │ │term 2 │ │term 3 │
        └───────┘ └───────┘ └───────┘ └───────┘ └───────┘
```

- **Index**: position in the log (starting at 1).
- **Term**: the term in which the entry was *originally created* — never rewritten, even when a later leader re-replicates it.
- **Command**: the state machine operation (e.g., `set x = 42`).

### The write path, step by step

```
Client:  set x = 42
              │
              ▼
┌───────────────────────────  Leader  ───────────────────────────┐
│ 1. Append entry {x=42, term=2, index=4} to its own log        │
│ 2. Persist it to disk (fsync!)                                 │
│ 3. Send AppendEntries to all followers in parallel            │
│    (with prevLogIndex=3, prevLogTerm=1 — consistency check)   │
│ 4. Wait for ACKs from a MAJORITY of the cluster               │
│ 5. Commit: advance commitIndex, apply x=42 to state machine   │
│ 6. Reply success to the client                                │
│ 7. Tell followers "commitIndex is now 4" (piggybacked on the  │
│    next heartbeat / append)                                   │
└───────────────────────────────────────────────────────────────┘
```

The AppendEntries RPC is the workhorse — it does double duty as **heartbeat** (when `entries[]` is empty) and **replication** (when it carries entries). Its fields:

```
AppendEntries(
    term,                      // leader's current term
    leaderId,
    prevLogIndex, prevLogTerm, // index+term of the entry right before the new ones
    entries[],                 // one or more new entries
    leaderCommit               // leader's commitIndex, so followers learn what's safe
)
```

### The consistency check (the heart of log replication)

Every AppendEntries includes `prevLogIndex`/`prevLogTerm` — the entry the follower must already have *right before* the new entries. The follower checks: *"do I have an entry at prevLogIndex with term prevLogTerm?"*

- **Yes** → append the new entries (and delete any conflicting ones that start there), reply `success = true`.
- **No** → reply `success = false`. The leader can't send the follower the new entries, because the follower might be missing a common prefix — appending would create holes.

This check is the induction step that maintains the **Log Matching Property**:

> **Log Matching:** (1) if two logs contain an entry with the same index *and* term, the entries hold the same command; (2) if they contain identical entries at a given index/term, all preceding entries are also identical.

Once a follower confirms it has the same prefix, the append is guaranteed to extend an identical prefix — so logs can never diverge silently.

### Fixing up lagging followers

A new leader (or a leader with a slow follower) doesn't know where follower logs diverge. It guesses: `nextIndex[follower]` starts at the leader's last index + 1. When the follower rejects (mismatch), the leader **decrements nextIndex and retries** — walking back one entry at a time (or one *term* at a time, if the follower reports the conflicting entry's term — a common optimization) until it finds the last common entry. Then:

- The follower **deletes all conflicting entries** after the common prefix.
- The leader sends the rest.

This is the "conflicting entry deletion" rule. **Leaders never delete their own entries** (Leader Append-Only) — only followers are corrected, because the leader is the source of truth.

### When does an entry become committed?

The leader keeps a `commitIndex` (highest committed) and per-follower `matchIndex[]` (highest index the follower has confirmed). It commits entry *N* when:

1. `N` is replicated on a **majority** of the cluster (`matchIndex[f] ≥ N` for a majority of servers), **and**
2. **`log[N].term == currentTerm`** ← the "current term rule" (§6.3 — this is the subtle one).

When the leader advances `commitIndex`, it piggybacks the value on subsequent heartbeats/appends, and followers apply entries up to that point in log order.

### What if the leader crashes mid-replication?

RPCs are designed to be **idempotent**: a follower ignores entries it already has (matching index and term). Servers retry requests indefinitely. If the response was lost (the classic "did it commit or not?" ambiguity), the retry just re-sends the same entries and the follower answers based on its current state. When the leader crashes, a new one is elected and simply walks `nextIndex` down from the end of *its* log to find common ground — no special recovery machinery. This "fix by retry" design is what makes Raft's failure handling so much simpler than Paxos's.

---

## 6. Safety: Why Raft Never Goes Wrong

This is the part where most explainers hand-wave. Let's build it up properly, because it's actually understandable.

### 6.1 Election Safety: at most one leader per term

A leader needs a **majority** of votes. Two leaders in the same term would both need a majority of votes — and two majorities always overlap by at least one server (with 5 servers, two majorities of 3 share at least 1 server; in general, 2⌈N/2⌉+... — the pigeonhole principle guarantees an intersection). Since each server votes for **at most one candidate per term** (enforced by persisting `votedFor` to disk before responding), no single server can vote for two different candidates in the same term. So a second leader in the same term is impossible. Terms only increase, and a higher term demotes a lower-term leader — so across all time, **at most one leader is valid at any moment**. That's Election Safety.

```
Term 5:            votes:  2 of 5 servers are enough
                ┌─ 3 of 5 majorities always overlap ─┐
        Candidate X gets {S1,S2,S3}   Candidate Y gets {S4,S5,S1}?
        → impossible: S1 already voted in term 5     (votedFor = X, persisted)
```

### 6.2 Leader Completeness: every new leader has every committed entry

The claim: **if an entry is committed in term T, every leader of every later term contains it.**

Why is this true? Picture a committed entry `e` (replicated to a majority in term T) and a future leader `U` elected in a later term:

```
   Servers that got entry e:      Servers that voted for U:
   ┌───┐ ┌───┐ ┌───┐             ┌───┐ ┌───┐ ┌───┐
   │ S1│ │ S2│ │ S3│              │ S3│ │ S4│ │ S5│
   └───┘ └───┘ └───┘             └───┘ └───┘ └───┘
     majority for e                 majority for U
                          └─ S3 is in BOTH majorities ─┘
```

By the pigeonhole principle, a majority for `e` and a majority for `U` **must share at least one server** — call it the *voter*. The voter held `e` when it voted (committed entries are never removed, and the voter accepted `e` before voting — otherwise its term would have been ahead of the leader that sent `e`). The voter only granted its vote to `U` because **U's log was at least as up-to-date as the voter's** (the election restriction from §4). An up-to-date log contains everything the voter has — including `e`. So `U` has `e`. ∎

That's the entire proof: **two majorities overlap + up-to-date voting rule = every leader inherits all committed entries.** Every safety property of Raft cascades from this.

### 6.3 The "current term" commit rule (Figure 8)

Here's the subtlety that trips up most implementations: **a leader may NOT commit entries from previous terms just by counting replicas.** It can only count replicas for entries from its *own* term.

The danger, from the paper's Figure 8: a stale entry replicated to a majority in a previous term might later be overwritten by a different entry. Timeline:

```
(a) S1 (term 2) replicates entry X at index 2 to S2 only. No commit (1 of 3).
(b) S1 crashes. S5 gets elected (term 3) with votes from S2, S3, S4.
    S5 writes a DIFFERENT entry Y at index 2 and replicates to S3, S4.
    Now index 2 is Y on a majority — but it's term 3's entry.
(c) S5 crashes. S1 restarts, gets elected (term 4), and replicates X at
    index 2 to a majority. If S1 committed X by replica count alone —
    it would be wrong: S2 has X, S3/S4 have X now... but consider:
(d) S1 crashes again. S5 (term 5) can be re-elected with votes from
    S2, S3, S4 (S5's log is more up-to-date: it has term 3's Y at index 2)
    and would overwrite X with Y — even though X "had a majority."
```

The flaw: an entry from an *older* term replicated by a *newer* leader hasn't necessarily reached the servers that will vote in the *next* election — a later leader (like S5) could still have a more up-to-date log *without* that entry, get elected, and overwrite it.

**The fix:** a leader commits entries from its own term by majority count, and **commits older entries only indirectly** — once an entry from the current term is committed, Log Matching (§5) makes everything before it committed too. That's why a new leader should always have a no-op entry (or commit quickly) — to pin down its predecessors (see §10).

> **Rule of thumb:** *commit only what you wrote; inherit the rest.*

### 6.4 State Machine Safety

The final property: **if a server has applied an entry at index `i`, no other server will ever apply a *different* entry at index `i`.**

Proof sketch: take the *first* term in which any server applies index `i`. That entry `e` is committed in that term (servers only apply committed entries). By Leader Completeness, every later-term leader contains `e`; by Log Matching, followers are always corrected to the leader's log. So no server can ever hold a different entry at `i` — applying in log order, all state machines produce identical outputs. This is what "safe" means in practice: **a replicated state machine never contradicts itself.**

---

## 7. Handling Failures: Partitions, Crashes, Reordering

Raft's safety holds through every combination of these, and this is the section that demonstrates it.

| Failure | What Raft does |
|---|---|
| **Message loss** | Retry indefinitely (idempotent RPCs make retries safe). |
| **Message duplication** | Idempotent RPCs: duplicate AppendEntries/RequestVote are harmless. |
| **Message reordering** | Followers append strictly in log order; out-of-order entries fail the consistency check and are retried. Leaders pipeline but follow up correctly. |
| **Slow server** | Doesn't matter for correctness; commits only need a majority. The leader keeps retrying the slow follower in the background. |
| **Follower crash** | Log persists (fsync before ACK); it rejoins and gets caught up via AppendEntries or a snapshot. |
| **Leader crash** | New election (§4); new leader's log has everything committed (§6.2); stale entries get overwritten. |
| **Network partition** | Only the majority side elects a leader and commits; the minority side stalls but never serves wrong data; terms heal everything on rejoin. |
| **Clock skew / message delay** | Can slow elections but never corrupt state — safety is timing-independent. |
| **Disk stall** | A stalled disk on the leader delays fsync → heartbeats slow → followers start elections → leader deposed. Availability problem, not a safety problem. |

---

## 8. Log Compaction: Snapshots

The log grows forever — that's a problem. Raft compacts the log with **snapshots**:

```
Before snapshot (log keeps every entry):
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
  │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │
  └───┴───┴───┴───┴───┴───┴───┴───┘

After snapshot (state captured, log truncated):
  ┌───────────────┐  ┌───┬───┬───┐
  │ SNAPSHOT      │  │ 6 │ 7 │ 8 │   ← new entries after the snapshot
  │ state at idx 5│  └───┴───┴───┘
  │ lastIncluded  │
  │   index  = 5  │
  │   term   = 2  │
  └───────────────┘
```

A snapshot stores the **entire state machine state** plus two crucial fields:

- `lastIncludedIndex` — the index of the last entry the snapshot replaces
- `lastIncludedTerm` — that entry's term

Why these two fields? Because the snapshot *removed* log entries — and the `prevLogIndex`/`prevLogTerm` consistency check (§5) still needs to work for the entry right after the snapshot. The snapshot's last-included index/term serve as the "virtual log entry" for that check.

**When to snapshot:** when the log exceeds a byte threshold — ideally relative to the *previous* snapshot size (e.g., snapshot when the log is 4× the last snapshot; the paper's numbers: ~20% of disk bandwidth spent snapshotting, ~6× capacity headroom).

**Snapshotting is per-server and independent** — followers snapshot on their own schedule without asking the leader (this violates the "strong leader" principle, but it's safe because the state was already agreed).

### InstallSnapshot: catching up a server that's too far behind

If a follower (or a brand-new server) is so far behind that the leader has already discarded the log entries it needs, the leader sends **InstallSnapshot** — the snapshot in chunks (`offset`, `data[]`, `done`). Chunks also reset the follower's election timer (a sign of life).

**The critical rule for the receiver** (Figure 13, Rule 7 — the one real implementations get wrong):

> If the snapshot's `lastIncludedIndex` is *newer* than the follower's log, **discard the entire log** and reset the state machine from the snapshot. If the snapshot overlaps existing entries, keep the suffix only if it matches exactly (same index+term).

Why discard everything? The follower's old log might contain **uncommitted conflicting entries** — mixing them with snapshot state would corrupt the state machine. When in doubt, throw away and rebuild.

---

## 9. Membership Changes

Adding or removing servers is a consensus problem *within* consensus. Naively switching the configuration from `C_old` to `C_new` is **unsafe**. Example from the paper: a 3-server cluster (S1, S2, S3) growing to 5 (add S4, S5).

```
          C_old (3 servers)                C_new (5 servers)
   majority of C_old = 2 servers    majority of C_new = 3 servers
   S1 + S2 can elect a leader       S3 + S4 + S5 can elect a leader
```

At the moment of transition, S1+S2 (majority of the *old* config) and S3+S4+S5 (majority of the *new* config) can elect **two different leaders at the same time** — split brain. Direct config switch is broken.

### Solution 1: Joint consensus (the paper's approach)

Raft transitions through an intermediate configuration **`C_old,new`** — a special log entry that names *both* old and new servers:

1. The leader appends `C_old,new` to its log; from then on, **any server in either configuration can be leader**, and entries are replicated to **both** configs.
2. An entry is committed under joint consensus only if it has a **majority in C_old AND a majority in C_new** — overlapping quorums again make split brain impossible.
3. Once `C_old,new` is committed, the leader appends `C_new`; when *that* commits (under C_new's rules), servers not in `C_new` shut down.

### Solution 2: One server at a time (the simpler approach — what etcd actually does)

The dissertation describes a simpler scheme used by most production systems (etcd/raft's `ConfChangeV2` supports joint consensus, but **single-server changes are recommended**): **change only one server at a time**. Any major change (3→5) becomes a sequence of single-server changes (3→4→5). With a one-server change, every majority of `C_old` overlaps every majority of `C_new` (removing/adding one server can't create disjoint majorities), so safety follows from the same overlap argument.

**Learners** make this even safer: a new server joins as a **learner** — it replicates the log but *doesn't vote and doesn't count toward quorum*. Once caught up (etcd waits ~10 rounds, then requires the last round to be faster than an election timeout), it's promoted to a voting member. This prevents the common failure where a brand-new, empty server is added directly, becomes part of quorum, and immediately stalls the cluster (it can't keep up with the log).

### The disruptive removed-server problem

A removed server that doesn't know it's removed keeps starting elections with higher and higher terms, repeatedly deposing the real leader. Fixes: **pre-vote** (§11) helps, but the definitive fix is servers ignoring RequestVote from candidates they haven't heard from in a while — "if I've heard from a valid leader within the minimum election timeout, ignore votes." (This conflicts with leadership transfer, which tags its vote requests as authorized.)

---

## 10. Client Interaction and Linearizable Reads

### The basic write path

- Clients contact any server; a **follower rejects writes and returns the leader's address** (it learned the leader from AppendEntries' `leaderId`).
- The leader executes, replicates, commits, and replies.

### Making reads safe is harder than writes

Reads *could* just read the leader's state machine. But consider: the leader gets partitioned away. It still believes it's leader. A client keeps asking it for reads — and it returns **stale data** (the new leader on the majority side has committed newer writes). This is the classic **stale-read bug** and it has shipped in real implementations.

The **ReadIndex algorithm** fixes it with two steps:

1. **Commit a no-op entry at term start.** When a leader is elected, it appends a blank no-op entry to its log and commits it. Why? The new leader can't be sure which of its entries are actually committed (it might have uncommitted entries from previous terms, §6.3). The no-op's commit pins `commitIndex` to reality.

2. **Quorum round-trip before reading.** To serve a linearizable read, the leader:
   - Records its current `commitIndex` as `readIndex`.
   - **Heartbeats a majority of the cluster** — proving it's still the legitimate leader (if a newer leader existed, someone would have rejected the heartbeat with a higher term).
   - Waits until `lastApplied ≥ readIndex` (its state machine has caught up).
   - Serves the read from local state. A single quorum round-trip can be amortized across a *batch* of pending reads.

Followers can serve reads too: they ask the leader for a `readIndex` and wait until they've applied it. (etcd exposes this as `ReadOnlySafe` — default — with the leader executing the ReadIndex protocol.)

### Leader leases: skipping the round-trip (at your own risk)

**Leader lease** optimization: followers won't start an election until their election timeout expires after the last heartbeat. So for a brief window after a majority acks heartbeats, *no one can win an election* — the leader can serve reads without the quorum round-trip (etcd's `ReadOnlyLeaseBased` mode). This is faster but **depends on clock drift staying small** — if follower clocks run fast, they might elect a new leader while the old one still believes its lease is valid → **stale reads**. The paper explicitly recommends against it unless necessary; CockroachDB's closed-timestamp scheme (§13) is the modern alternative that gets bounded-staleness reads without leases.

---

## 11. Production Extensions

All extensions are optional features that keep the core algorithm untouched:

| Extension | What it does | Why it exists |
|---|---|---|
| **Pre-vote** | Before incrementing its term, a candidate asks a majority whether they *would* vote for it. Only with permission does it start a real election. | A partitioned server with an inflated term rejoining would otherwise depose a healthy leader (its high term demotes everyone). Pre-vote prevents term disruption. |
| **CheckQuorum** | The leader steps down if it hasn't reached a quorum within an election timeout. | Prevents a partitioned leader from serving reads forever; speeds client failover. |
| **Leader transfer** | Graceful leadership handoff: the leader stops accepting requests, catches the target follower up, then sends it a `TimeoutNow` message that fires its election timer immediately. | Maintenance without downtime; the target wins the next term legitimately (its majority win preserves Leader Completeness). |
| **Learners** | Non-voting members that replicate but never count toward quorum (§9). | Safe scaling/member addition. |
| **Leader lease** | §10. | Cheaper linearizable reads, with clock-drift risk. |

---

## 12. Performance and Engineering Details

### The fundamental cost: fsync

Here is the single most important performance fact about Raft:

> **An entry is not committed until it's durable on a majority of servers — and "durable" means fsync'd to disk.**

- A write's minimum latency = network RTT + fsync time. Measured: NVMe ~100 µs/fsync, SATA SSD ~1–2 ms, spinning disk >5 ms (untenable). etcd's own docs state the minimum request time is RTT + fdatasync.
- That's why **etcd is famously sensitive to slow disks**: a leader whose disk stalls misses heartbeats and gets deposed. "Disk latency is part of leader liveness."
- Mitigations: **group commit / batching** — the leader collects many client requests and fsyncs them in one batch (how etcd sustains ~10k writes/s on NVMe), and parallel disk writes (the leader can commit when a majority of *followers* are durable, even before its own fsync completes — the commit is still safe).

### Batching and pipelining

- **Batching**: AppendEntries can carry multiple entries; one RPC + one disk write moves many commands. TiKV goes further: a batched event loop gathers *all* ready Raft groups and writes all log appends in a single WriteBatch — one fsync for hundreds of groups.
- **Pipelining**: the leader sends AppendEntries without waiting for the previous ACK, optimistically advancing `nextIndex`. On rejection it rewinds. Pipelining roughly doubles throughput on lossy links but is famously bug-prone (even LogCabin's first pipelining attempt never shipped).

### Tuning elections

The paper's timing requirement: `broadcastTime ≪ electionTimeout ≪ MTBF`. Rule of thumb: heartbeat ≈ election timeout / 10 (etcd: 100 ms / 1000 ms). Too-short timeouts cause constant elections; too-long timeouts delay failover. With 150–300 ms random timeouts, the paper measured: no randomization → elections >10 s; +5 ms randomization → 287 ms median downtime; +50 ms → 513 ms worst case. In production, etcd typically fails over in ~1–2 s (100 ms heartbeat + 1000 ms election timeout); MongoDB (Raft-inspired, 10 s timeouts) fails over in ~12 s.

### Throughput and latency reality

| System | Typical commit latency | Notes |
|---|---|---|
| etcd (3 nodes, NVMe, intra-DC) | ~1.6 ms median writes; ~10k writes/s | 100 µs fsyncs, batched |
| Cross-region Raft | 50–100+ ms | One quorum round-trip per commit |
| HashiCorp raft (Consul/Vault) | ~20 ms/write on laptops | Laptop fsyncs are slow |

Election results: with 5 servers you survive 2 concurrent failures; 3 servers survive 1.

---

## 13. Raft in the Real World

| System | How it uses Raft | Notable adaptation |
|---|---|---|
| **etcd / etcd-raft** (Go) | The reference library; powers Kubernetes, Docker Swarm, Calico. Minimal by design: the library implements only the algorithm — transport and disk are the caller's job (`Ready`/`Advance` pull interface). | Learners, pre-vote, CheckQuorum, ReadIndex (`ReadOnlySafe` default), `MaxInflightMsgs` flow control, single-server config changes (recommended over joint consensus). |
| **CockroachDB** | **One Raft group per range** — a node participates in *thousands* of Raft groups (default 3–5 replicas per ~512 MB range). | **Closed timestamps** for consistent follower reads: the leaseholder promises "no writes below timestamp T" and followers can read at T once applied — bounded-staleness reads without leader leases. Delegated snapshots (v23.1+). |
| **TiKV / TiDB** (Rust, raft-rs) | One Raft group per Region; a batched event loop with a single WriteBatch+fsync per tick across all groups. | **Raft Engine**: custom append-only log store replacing RocksDB (QPS +4%, write I/O −25–40%, recovery of 10 GB in <2 s). Hibernate regions to idle unused groups. |
| **Consul / Vault** (hashicorp/raft) | Servers form the Raft cluster; client agents forward to the leader. | **Autopilot** automates membership (dead-server removal, health checks, stabilization delay before a new server votes). Lease-based reads can be stale in bounded windows — documented. |
| **MongoDB** (pv1) | Raft-*inspired* replica-set protocol — terms, one-vote-per-term, majority elections, oplog freshness checks — but with specialized features: only 7 voting members (of up to 50), arbiters, priority-based elections, and **rollback** instead of log overwrite. | "Raft-like" by the engineers' own description; ~200,000 replica sets in their DBaaS, half electing a leader each week. |
| **LogCabin** | The paper author's reference implementation (~2,000 lines of C++), built for RAMCloud. | The place Raft ideas were proven in practice. |

**Common theme:** every system keeps the *core* algorithm (terms, votes, log, commit-by-majority) faithful to the paper, and adapts everything around it — reads (ReadIndex/leases/closed timestamps), membership (learners/autopilot), and storage (dedicated log engines).

---

## 14. Common Bugs and Misconceptions

These are real mistakes that have shipped in real systems — studying them teaches the algorithm better than anything:

1. **Committing entries from previous terms by replica count.** The Figure 8 violation (§6.3) — the #1 misconception. Fix: commit only current-term entries by count; older entries commit transitively.
2. **Term confusion.** Using a stale term from a response received *before* you advanced your own term; not comparing the response's term against the term you sent in the request. Responses from older terms must be ignored or handled with care.
3. **Granting two votes in one term.** HashiCorp raft bug #695: an async heartbeat race let one node vote for two candidates in the same term — producing two leaders and violating three of the four safety properties. The fix is why `votedFor` is persisted to disk *before* responding.
4. **InstallSnapshot not discarding the entire log** (Figure 13 Rule 7). HashiCorp raft #697: the follower kept a stale log, causing an AppendEntries/InstallSnapshot livelock *forever*. If the snapshot's last-included index is ahead of you, discard everything.
5. **Snapshot overwriting newer entries.** A leader applying a stale snapshot must not clobber log entries newer than the snapshot (etcd-io/raft #157, tikv/raft-rs #577) — the snapshot's lastIncludedIndex vs. your log must be compared carefully.
6. **Broken leader leases.** The paper's lease spec is vague, and several implementations got it wrong (Jelle van den Hooff's famous "Make leader leases actual leases" issue): a deposed leader serving reads within a broken lease = stale reads. TiDB's lease even causes ~10 s of read unavailability after elections.
7. **Conflating nextIndex and matchIndex.** `nextIndex` is an optimistic guess (performance); `matchIndex` is a conservative measurement (safety). Never initialize matchIndex optimistically — commit decisions must be based on real acknowledgments.
8. **Membership-change footguns.** Direct config switches are unsafe (§9); removed servers must be prevented from campaigning; etcd warns against 2-member clusters (removing one member from 2 requires... arithmetic that doesn't work — 2 servers is *worse* than 1 for availability).
9. **"Raft guarantees liveness."** It doesn't, strictly: under pathological network behavior, liveness can be delayed indefinitely. Raft guarantees **safety** unconditionally; liveness holds under reasonable timing assumptions (`broadcastTime ≪ electionTimeout ≪ MTBF`).
10. **Antithesis's finding**: it found bugs in *every* Raft implementation it tested. Raft is simple to read — and surprisingly hard to get right in code.

---

## 15. Glossary

| Term | Definition |
|---|---|
| **Consensus** | Agreement among a group of machines despite failures. |
| **Replicated state machine** | The pattern: identical logs + deterministic execution = identical state. |
| **Term** | Raft's logical clock; consecutive integers; each term starts with an election. |
| **Leader / Follower / Candidate** | The three server roles. |
| **Election timeout** | Randomized timer; expiry makes a follower start an election. |
| **RequestVote** | Election RPC: `(term, candidateId, lastLogIndex, lastLogTerm)`. |
| **AppendEntries** | Replication + heartbeat RPC: carries `prevLogIndex`/`prevLogTerm`, entries, `leaderCommit`. |
| **InstallSnapshot** | Catch-up RPC carrying snapshot chunks. |
| **Quorum / majority** | ⌊N/2⌋ + 1 servers; the minimum needed to commit or elect. |
| **commitIndex** | Highest index known to be committed (safe to apply). |
| **lastApplied** | Highest index applied to the state machine. |
| **nextIndex / matchIndex** | Leader's per-follower guess / confirmed replication position. |
| **Log Matching Property** | Same (index, term) ⇒ same command; identical prefix extends identically. |
| **Election restriction** | Only up-to-date candidates get votes (the safety linchpin). |
| **Leader Completeness** | Every committed entry is in every future leader's log. |
| **ReadIndex** | Linearizable read via commitIndex pin + quorum heartbeat. |
| **Joint consensus** | Transitional config needing majorities in both old and new configs. |
| **Learner** | Non-voting member that catches up before promotion. |
| **Pre-vote / CheckQuorum** | Extensions preventing term disruption and stale leadership. |
| **Leader lease** | Timing-based read optimization (clock-drift sensitive). |

---

## 16. Further Reading

1. **The Raft paper** — *In Search of an Understandable Consensus Algorithm* (Ongaro & Ousterhout, USENIX ATC 2014), extended version: https://raft.github.io/raft.pdf — read Figure 2 (the whole algorithm) slowly; everything else is commentary.
2. **The dissertation** — *Consensus: Bridging Theory and Practice* (Ongaro, Stanford 2014): pre-vote, leases, learner promotion details, LogCabin implementation notes: https://github.com/ongardie/dissertation
3. **raft.github.io** — links to talks, TLA+ spec, and the interactive visualizations.
4. **The Secret Lives of Data** (Ben Johnson) — the gentlest guided visual walkthrough: https://thesecretlivesofdata.com/raft/
5. **RaftScope** — interactive sandbox where you can kill servers and watch elections: https://github.com/ongardie/raftscope
6. **The Student's Guide to Raft** (Jon Gjengset) — the best catalog of implementation bugs, written from MIT 6.824 TA experience: https://thesquareplanet.com/blog/students-guide-to-raft/
7. **The TLA+ spec** — machine-checked formal proof of Leader Completeness: https://github.com/ongardie/raft.tla
8. **TiKV deep dive** — multi-Raft and Raft Engine internals: https://tikv.github.io/deep-dive-tikv/
9. **etcd raft package docs** — the production reference library: https://pkg.go.dev/go.etcd.io/etcd/raft/v3
10. **Antithesis: Finding Bugs in Raft Implementations** — real bugs in HashiCorp raft, Aeron, OpenRaft, MicroRaft: https://antithesis.com/blog/2026/finding-bugs-in-raft-implementations/

---

*Compiled from the Raft paper and dissertation (Ongaro), raft.github.io, the TLA+ specification, etcd/raft and raft-rs documentation, CockroachDB/TiKV/Consul/MongoDB engineering docs and blogs, Jon Gjengset's student guide, and the Antithesis fault-injection writeup. All protocol details follow Figure 2 of the paper.*
