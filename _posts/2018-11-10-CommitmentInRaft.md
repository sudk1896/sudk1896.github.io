---
title: On rules of Commitment in Raft.
layout: post
comments: true
---
I was reading up on Raft, an understandable consensus algorithm. The link to the paper is
[here](https://raft.github.io/raft.pdf). Only continue reading if you've read the paper.


Right here The paper states that

![Page 7](https://sudk1896.github.io/images/Screenshot%20from%202018-11-10%2013-57-44.png),
```
A Log entry is committed once the leader that created the entry has 
replicated it on a majority of the servers.
```

I misunderstood this statement to assume that whenever an entry is present on a majority of 
the servers, it is committed. This assumption doesn't hold.

On Page-8 of the raft paper and in Figure-8  

![alt text](https://sudk1896.github.io/images/Screenshot%20from%202018-11-10%2014-15-33.png "here"),


In Figure-8.c, It is Term 4 and S1 is leader. Even though the entry at Index 2 Term 2 is present on a majority of 
the servers, it is entirely possible that that entry can be wiped out as shown in Figure-8.d. In the next Term i.e. Term 5 (whenever new leader is elected term increases) S5 could overwrite entries at indices >= 2 with its own Term 3 entry (because of AppendEntries RPC #3)

```
If an existing entry conflicts with a new one(same index but different terms), 
delete the existing entry and all that follow it.
```

In order for this to not happen, the **leader in the current term** should replicate an entry that it received in the 
**current term** on a **majority** of the servers. In our case, S1 needs to replicate the entry it received in Term 4 on a majority of the servers. After doing that, if S1 crashes before committing the entries, S5 cannot be elected leader and hence won't overwrite the entries
with indices >= 2. The RequestVote RPC would fail for S5 as RequestVoteRPC #2 would not hold, S5's logs aren't up-to-date
with S1,S2,S2 (i.e. a majority) and hence S5 cannot get votes from a majority of the servers and can't be elected as leader.

There are two subtle learnings that I could discern - 

1. Even if entries from older terms are present on a majority of the servers, that is no guarantee of commitment.
2. In order for an entry to be committed, an entry must be stored on a majority of the servers and the leader that commits it must have replicated an entry from its own current term to a majority of the servers.

