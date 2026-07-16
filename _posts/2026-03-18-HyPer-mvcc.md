---
title: On MVCC in HyPer
layout: post
comments: false  
---

MVCC (Multi-version concurrency control) in Databases help scale the read and write throughput
by not letting readers block the writers and vice-versa. For now, we will talk about how
snapshot isolation is implemented using MVCC as it has been explained in [Fast Serializable Multi-Version Concurrency Control
for Main-Memory Database Systems](https://15721.courses.cs.cmu.edu/spring2016/papers/p677-neumann.pdf).

Also, this post will only go over how sequential scan would work in the above MVCC scheme.
We will leave Insert, Updates and Deletes for a later post. The goal is to build a solid understanding
via pseudo code on how how timestamps are assigned to transactions and how they are used to read data.
The idea in Snapshot Isolation (SI) is that every transaction gets a consistent snapshot of data as and when
it was created. They then make changes to the data in their own private workspace i.e., the changes aren't
visible to any other transactions. During commit time, transactions check for any possible conflicts
in the data that was written wrt running or committed transactions and 
resolve the conflicts using some scheme such as first committer wins, first updater wins etc.
This post will not go over the Commit phase.

The HyPer paper proposes that every row in the table stores a _version vector_. The version vector
is conceptually a pointer to a Linked list (LL). Each node in this LL contains an _undo buffer_.
This undo buffer just stores the change that was made for the row and not the entire modified row.
As an example, assume we have the following tuple `(1, 2, 3)` and a transaction modified it to
`(11, 2, 3)` i.e., only the first attribute was modified. Instead of storing the modified row,
the undo buffer only stores `(11)`. We also store metadata about which properties were and weren't
modified so that we can reconstruct the tuple later on (say during a read or sequential scan).

Every transaction gets the following properties,


$\texttt{uint64\_t txn\_id} \in [2^{62}, \infty)$ (unique per txn)
$\texttt{uint64\_t read\_ts} \in [0, 2^{62})$ - when did this transaction start ?
$\texttt{uint64\_t commit\_ts} \in [0, 2^{62}]$ - when did this transaction commit ?
$\texttt{vector<UndoLog>} \text{undoBuffer}$ - List of undo logs for this txn

$\texttt{commit\_ts} > \texttt{read\_ts}$


Every tuple has the following properties,

$\texttt{uint64\_t ts}$ — stores the commit timestamp or the transaction id of the running transaction that modified it
$\texttt{bool is\_deleted}$

Every _undo log_ or _undo buffer_ has the following properties
$\texttt{bool is\_deleted}$ — whether this undo log corresponds to a deletion
$\texttt{uint64\_t ts}$ — timestamp corresponding to this undo log
$\texttt{Tuple tuple}$ — stores the partial tuple corresponding to the change, not the entire tuple (additional metadata required to reconstruct the tuple is omitted for brevity)
$\text{Link}$ // -> This is a pointer to the undo log before $\text{ts}$ if one exists

Assume we have transaction $\text{T}$ that starts at time $\text{t}$, we want to read the tuple 
$\text{X}$. In SI, every transaction gets its own snapshot of data. $\text{T}$ should see the world
as it exists at time $\text{t}$.

When $\text{T}$ begins, it gets assigned a $\texttt{txn\_id}$ that is unique and starts from ${2^{62}}$ and upwards.
The $\text{read\_ts}$ is non-negative and monotonically increasing counter that gets assigned at the start and is ${ < 2^{62}}$.

When $\text{T}$ modifies X, it does two things. It modifies the tuple in-place and sets the $\text{ts} = \text{txn\_id}$ and it creates an undo log and stores it in its undo buffer. Lets plug in some numbers to make it
more concrete.
Initially, $\text{X = 200}$ and $\text{T}$ modifies it to $\text{X = 300}$ at time $\text{t = 9}$.
Assume that $\text{txn\_id}$ = ${2^{62} +12}$. We will ignore the ${2^{62}}$ and use the remaining
number as the transaction identifier.

