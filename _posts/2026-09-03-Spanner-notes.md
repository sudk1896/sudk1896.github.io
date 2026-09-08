---
title: Thoughts on Google Spanner
layout: post
comments: false
spanner_paper_url: "https://research.google/pubs/\
  spanner-googles-globally-distributed-database-2/"
spanner_reads_url: "https://docs.cloud.google.com/spanner/docs/whitepapers/\
  life-of-reads-and-writes"
clock_sync_url: "https://www.yugabyte.com/blog/\
  evolving-clock-sync-for-distributed-databases/"
---

<!-- markdownlint-disable-file MD033 -->

Spanner is Google's scalable multi-version, distributed and synchronously
replicated database. It provides externally-consistent distributed transactions,
lock-free snapshot reads, and lock-free read-only transactions. The
[Google Spanner paper]({{page.spanner_paper_url}}) is the basis for these
learnings, along
with the other sources cited at the bottom.

## Distributed Transactions with High-Availability

Spanner supports Distributed Transactions i.e., transactions that span
multiple nodes by using Two-Phase Commit (2PC). One major problem with 2PC is
that its a blocking protocol. Assume that we have a Txn co-ordinator (TC) and
two participants P1, P2 (P1 and P2 are different nodes). The co-ordinator has
sent Prepare messages to P1 and P2. Both of them have replied with Prepared
messages. In doing so, they are holding locks and are telling the co-ordinator
that they are ready to Commit. Now if the TC fails, the protocol won't make any
progress since P1/P2 don't know whether to commit or abort. P1 and P2 would be
left holding locks which might block other transactions that want to access the
items locked by P1 and P2. This brings down the availability of the system. So,
how does Spanner solve this ?

Simple, by replicating the TC and the participants state using Paxos. Every time
the TC or a participant sends a message (Prepare, Prepared, Commit/Abort etc),
they wait for the message to be replicated by Paxos. Once the message is
replicated, if the TC or Participant fails, an up-to-date replica is elected as
the new TC or Participant. This solves the availability problems caused by the
blocking nature of 2PC.

## TrueTime API

Spanner uses a novel API called TrueTime. We will focus on the API provided and
not on how its actually implemented. Spanner uses TrueTime to assign timestamps
to the transactions.

```go

struct TTInterval{
    TTstamp earliest
    TTstamp latest
}

TTInterval tt
tt = TT.Now() // returns [earliest, latest]
TTstamp t
has_t_passed := TT.after(t) // returns true if t has definitely passed
// Returns true if t has definitely not arrived.
has_t_not_arrived := TT.before(t)
```

If we invoke `TT.Now()` instead of getting the precise timestamp as of this
instant, we get a window which says, the current time is within this window with
a very high probability. The width of this window, the _epsilon_ in the paper,
is usually 1-7ms although it can increase due to DC failures, clock drift etc.

## Concurrency control in a Distributed Database

Spanner provides _external consistency_ which is equivalent to Strict
Serializability. Here is what external consistency means: if we have two txns T1
and T2 and T1 finishes before T2 starts, then T2 should see all the effects of
T1. It also provides lock-free (non-blocking in the paper) read-only txns and
snapshot reads.

### Non-blocking snapshot reads and non-blocking read-only txns

Spanner is an MVCC database. When it writes an item _X_ , it creates a new
version of X, assigns it a timestamp, and stores this version along with the
older versions. In a snapshot read, the client can supply a timestamp and ask
Spanner for _read(X, ts)_, which translates to _give me X as of ts_. In essence,
Spanner supports Snapshot Isolation (SI), which is an Isolation level supported
by most single-node DBs (Postgres, MySQL etc), but Spanner is distributed!

In a single-node DB, supporting SI is simple: assign each transaction a
monotonically increasing counter, which roughly translates to just having a lock
around a 64-bit integer and atomically incrementing it every time a txn wants to
execute. Any txn that executes gets assigned this monotonically increasing
counter, and if it wants to read _X_, it reads X as of this counter.

For read-only txns, Spanner assigns them a system-assigned time stamp and
executes them as a snapshot read at this time stamp. For both these scenarios,
there are no locks taken. All of this is possible because of the timestamping
scheme used by Spanner, which in turn relies on their novel _TrueTime_ API.
More on this timestamping scheme below.

### Transactional reads and writes

Transactional reads and writes use two-phase locking (2PL) with a wound-wait
scheme to break deadlocks. Below is how Spanner handles r/w txns:

- The client issues reads to the leader replicas and acquires read locks to read
  the latest data. Writes are then buffered by the client. The client drives 2PC
  (instead of a TC).
- The client chooses a co-ordinator group (meaning the TC and the participant
  leaders) and sends each participant leader a Commit message with the identity
  of a co-ordinator (chosen randomly).
- A participant leader (non co-ordinator) acquires write locks and assigns a
  _Prepare_ timestamp. This should be greater than all previous timestamps the
  leader has issued. It logs a _Prepare_ record through Paxos. The participant
  then sends the _Prepare_ timestamp to the co-ordinator leader.
- After hearing from all participant leaders, the co-ordinator acquires write
  locks but skips the _Prepare_ phase. It assigns the commit timestamp
  _commit_ts_ per the following scheme, described in 4.2.1. It then logs the
  commit record through Paxos.

```go
commit_ts = max(
    all_prepare_ts_from_nonparticipants,
    TT.Now().Latest,
    all_prev_ts_coordinator_has_assigned,
)
```

- It waits till _TT.after(commit_ts) = true_ and then returns the _commit_ts_ to
  the client and all participant leaders. This waiting period is what the paper
  calls _Commit wait_.
- Each participant leader logs the txn result through Paxos. They apply at the
  same timestamp and release the locks.

## Deep dive into the Timestamping scheme

The timestamping (TS) scheme is used to provide external consistency. The
description is spread out throughout Section 4 of the paper. Lets dive into how
it works.

### Monotonic timestamps and the disjointness invariant (Section 4.1.1)

In Paxos, leaders can go down and in place new leaders are elected. For the
commit timestamp, Spanner uses the timestamp that Paxos assigns to the
Transaction commit step (part of 2PC). Spanner uses long-lived leases (10s) so
that there are no frequent elections (part of Paxos). Whenever there is a
leadership change in a Paxos group, the timestamps assigned by the leaders are
disjoint from the previous leaders. Over a series of leadership changes, here is
how the timestamps assigned by the various leaders would look like:

`[0, 10], [11, 30], [35, 45]...`

This invariant is maintained for a single Paxos group (think group of P1
replicas, group of TC replicas etc) and not across groups.

### Assigning timestamps to r/w txns (Section 4.1.2 and 4.2.1)

Lets assume that we are the client and we have begun a r/w txn. Assume that we
are reading data items `a, b and c` which are on Spanner nodes `A, B and C`.
Assume that node A is chosen as the TC co-ordinator by us (the client). Below is
the txn we are executing:

```sql
R(a) // read a
W(b) // write b
W(c) // write c
```

Lets define some variables for transaction _T_i_,

e<sub>i</sub> <sup>start</sup> = Start of the txn
e<sub>i</sub> <sup>commit</sup> = Commit of the txn
t<sub>abs</sub>(t) = Absolute time at t
s<sub>i</sub> = commit timestamp assigned to T<sub>i</sub>

To ensure external consistency, we must ensure

t<sub>abs</sub>(e<sub>1</sub><sup>commit</sup>) <
t<sub>abs</sub>(e<sub>2</sub><sup>commit</sup>) =>
(s<sub>1</sub> < s<sub>2</sub>).

- Client sends Commit to _B_ and _C_. Each chooses a _Prepare_ time stamp larger
  than any previous timestamp it has assigned, _B_ts_ and _C_ts_. These are sent
  to the co-ordinator.
- After hearing from _B and C_, _A_ assigns the commit timestamp per the
  following scheme:

```go
tt := TT.Now()
latest_ts := tt.latest
commit_ts := max(max(B_ts, C_ts), max(latest_ts, largest_prev_ts_A))
// largest_prev_ts_A = largest ts assigned by A till now
```

- _commit_ts_ is the Commit timestamp of this txn. The co-ordinator leader then
  waits until `TT.After(commit_ts) = true` and only then replies to the client
  with the commit timestamp.

Lets see why we must do this _Commit wait_ step. Note that between any two
Spanner servers anywhere in the world, their clock drift is constrained by
_Epsilon_. Meaning, if _Epsilon_ is assumed to be 8ms, the difference between
the times on any two Spanner servers is 8ms at max.

- Assume that _tt = [100 108]_ and we assign _commit_ts = 108_. This implies
  that even though the actual physical time is somewhere between 100 and 108,
  for e.g., its 103, the timestamp commits the txn in the future at 108. Now,
  assume that we don't do commit wait and immediately return a successful result
  to the client. At physical time 105, the client starts another txn _T2_. From
  the client's POV, T1 finished and then T2 started, so external consistency
  demands that _commitTs(T1) < startTs(T2)_, but that is violated here.

```txt
physical time:

100   103   104   105              108
 |-----|-----|-----|----------------|
       |     |     |
       |     |     T2 starts
       |     |
       |     client sees T1 commit
       |
       T1 chooses timestamp = 108
```

If we must preserve external consistency, the timeline will look as follows

```txt
physical time

100       103             108   109
 |---------|---------------|-----|
           |               |     |
           |               |     client sees SUCCESS, T2 can start
           |               |
           |               commit timestamp
           |
           T1 chooses commitTS=108

           <--- commit wait --->
```

### Serving snapshot reads (4.1.3)

Any Paxos group leader might be holding read/write locks, so to support high
read-throughput, snapshot reads should be served by replicas. But replicas might
not be up to date and might be lagging the leader by varying amounts (they could
be in a Paxos minority).

The paper defines t<sub>safe</sub> as the maximum timestamp at which a replica
is up-to-date. Any replica can satisy a read if _t <= t<sub>safe</sub>_.
Intuitively, t<sub>safe</sub> says that the replica can serve data upto that
timestamp and that the history is settled up to that timestamp.

Its defined as follows,

t<sub>safe</sub> = min(t<sub>safe</sub> <sup>Paxos</sup>,
t<sub>safe</sub> <sup>TM</sup>)

t<sub>safe</sub> <sup>Paxos</sup> = timestamp of the highest applied Paxos write

Suppose the Paxos log conceptually contains:

```txt
entry        timestamp

write A        40
write B        55
write C        70
write D        90
```

One of the replicas has currently applied only:

```txt
40
55
70
```

but not yet 90.

Therefore:
t<sub>safe</sub> <sup>Paxos</sup> = 70

Why can we trust 70?

Because Spanner requires Paxos write timestamps to increase monotonically, and
they are applied in order. Therefore, after we've applied the write at 70, some
new Paxos write at timestamp 60 cannot suddenly appear later.

But this is not enough. There could be a txn that is _PREPARED_ but not
_COMMITTED_. This is the handled by t<sub>safe</sub> <sup>TM</sup>.

Now suppose that t<sub>safe</sub> <sup>TM</sup> = 100. We might assume we can
serve reads _@90_. But we have a _PREPARED_ txn T2 which has its prepare
timestamp as s<sub>i,g</sub> <sup>prepapre</sup> = 60. We don't know if and when
this txn will commit. It could commit at _65, 75, 85_ etc.

Say that this replica answers read _@90_ with `x = old_value` and later the T1
co-ordinator tells it `T1 committed@80`. T1 should have seen the new value but
instead we returned an old value.

_t_<sub>safe</sub> <sup>TM</sup> = earliest prepare timestamp of all txns which
are prepared but not committed. These txns may be thought of as unresolved. If
there are no prepared but not committed txms, its set as inifinity. Combined
together, a replica can be fully caught up from Paxos's perspective and still
not be safe for snapshot reads because of unresolved 2PC transactions.

```txt
Paxos writes applied through timestamp 120
t_safe^Paxos = 120
T1 prepare @80
T2 prepare @105
T3 prepare @95
t_safe^TM = 79
t_safe = min(79, 120) = 79

0--------------79|80----------95------105----------120
                 ^
                 T1 unresolved

Paxos:
-------------------------------------------------->120

Transaction manager:
--------------->79

Overall:
--------------->79
```

As the txns T1, T2 and T3 are resolved (committed), t<sub>safe</sub> will
increase and therefore advance. So if _t > t<sub>safe</sub>_ at a replica, the
client must wait for t<sub>safe</sub> to advance before serving the read.

### Serving read-only txns (4.1.4)

A r/o txn executes in two phases: assign a timestamp, and then execute as a
snapshot read at that assigned timestamp.

A simplistic assignment of s<sub>read</sub> = TT.now().latest will work. But
t<sub>safe</sub> would not have advanced to s<sub>read</sub>, so the client
would then have to wait (be blocked). To reduce the chances of blocking, we use
the oldest timestamp that preserves external consistency.

Lets assume that the read can be served by one single Paxos group,

```sql
R(a) //a is served by one single Paxos group
```

Using s<sub>read</sub> = TT.now().latest can cause the client to wait (block) if
t<sub>safe</sub> hasn't advanced. Instead we can define
`LastTS() = last committed write in that Paxos group` and use s<sub>read</sub> =
LastTS() instead. If there are no Prepared but uncommitted txns, this assignment
will see the result of the last write and therefore satisfy external
consistency.

Assume that we have reads that are served by multiple Paxos groups,

```sql
R(a)
R(b)
R(c)
```

One option would be to set s<sub>read</sub> = min(LastTS() across all Paxos
groups). This would require communication with the leaders of all the Paxos
groups. Instead, Spanner uses something simpler: s<sub>read</sub> =
TT.now().latest, and has the client wait if t<sub>safe</sub> hasn't advanced
sufficiently. This presents a few opportunities to cut down on waiting time,
which we discuss below.

## Refinements (4.2.4)

There's a weakness to the way t<sub>safe</sub><sup>TM</sup> is defined. Even a
single _Prepared_ txn prevents t<sub>safe</sub><sup>TM</sup> from advancing. So
even if the txn wants to read something that doesn't conflict with the
_Prepared_ txn, it has to wait. This can be fixed by making
t<sub>safe</sub><sup>TM</sup> more granular. We can divide the key space covered
by the Paxos group into disjoint key ranges and have a mapping for
t<sub>safe</sub><sup>TM</sup> per key-range. When a read arrives, it only needs
to be checked for the safe time of the key range with which it conflicts.

`LastTS()` also has a similar weakness. If a txn has just committed, a
non-conflicting r/o txn has to be assigned s<sub>read</sub> = TT.Now().latest,
which might delay the read. Instead, we can have `LastTS()` be more fine-grained
and have a `LastTS()` value for every key-range. When a r/o txn arrives, its ts
can be assigned by taking the max of all `LastTS()` for the key ranges with
which it conflicts.

## References

[MIT 6.824 Spanner](https://pdos.csail.mit.edu/6.824/notes/l-spanner.txt)

[Life of Spanner reads & writes]({{page.spanner_reads_url}})

[Evolving Clock sync for Distributed DBs]({{page.clock_sync_url}})
