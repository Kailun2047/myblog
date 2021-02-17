---
title: "Very Brief Q&As on Raft"
date: 2021-01-29T11:09:50-08:00
draft: false
tags: ['Distributed-Systems']
---

# Introduction
A year ago (how time flies!) I got to learn Raft when trying to finish [a distributed system course](http://nil.csail.mit.edu/6.824/2018/schedule.html) made by MIT. It's overall a consensus algorithem targeting strong consistency, and it's designed for understandability and the ease for implementation. Recently there's a need to review it again and I'm making this post so that I don't forget as fast in the future.

# Resources
1. [Raft paper](https://raft.github.io/raft.pdf)
2. [A nice visualization on Raft concepts](http://thesecretlivesofdata.com/raft/)

Pictures used for illustration in this post are gratefully borrowed from the resources above.

# Raft Summary
Similar to Paxos, Raft is log-based and maintains a distinguished leader. But Raft has proposed a novel randomized-timer based leader election, and its leader is stronger in the sense that log entries only flow from leader to followers. These two features of Raft are also keys to its understandability. It also involves `term` for each leadership "cycle" as logical clock.

Generally, each server in a Raft cluster will persist a server state (KVs) and a series of logs, and can be acting as one of the three roles: leader, candidate and follower. A (distinguished) leader is responsible for replicating its own log entries onto other followers, and decide up to which index the log entries are successfully replicated on the majority of the cluster, and tells all servers to apply these log entries to their states. When no leader exist in the cluster, a servers can turn into candidate using a time-out mechanism, and request votes from other servers. Therefore, a Raft server will have 2 RPC interfaces: `AppendEntries` (for leader's action) and `RequestVotes` (for candidate's action).

The key properties that Raft gaurantees are:
1. *Election safety*: at most one leader can be elected in a given term
2. *Leader append-only*: a leader never overwrites or deletes entries in its log; it only appends new entries
3. *Log matching*: if two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index
4. *Leader completeness*: if a log entry is committed in a given term, then that entry will be present in the logs of the leaders for all higher-numbered terms
5. *State machine safety*: if a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index

# Raft Components
The essential components of Raft are listed in the figure below.

![alt text][raft-summary]

[raft-summary]: /images/raft/raft-summary.jpg "Raft summary"

The rest of this post are primarily answering some of the implementation-wise questions that are either interesting or somehow a bit vague in the original paper. Terms used are all referenced from the figure in this section. Hopefully they can help with others too in addition to answering questions for myself :-)

# Implementation Q&A
## Leader Election
### 1. How Raft avoids infinite split votes
Randomized election timeout value for each server. This could result in only one server times out in most cases, and wins the election before other servers time out.

### 2. How should a proper electon timeout value be chosen
Rule of thumb: broadcastTime (*system dependant*) << electionTimeout << MTBF (mean time between failures, *system dependant*)

`broadcastTime` should be a few order of magnitude less than `electionTimeout` so that leader can finish sending heartbeats before new election; `electionTimeout` should be a few order of magnitude less than `MTBF` so that servers can stay long enough to win election.

## Log replication
### 1. What happens when a follower find the term of its entry at `prevLogIndex` not matching `prevLogTerm`?
This case is produced in the case where a leader crashes before all its log entries are replicated. When the inconsistency happens, it's possible that:
 - a follower has entries that the current leader doesn't have
 - a follower misses entries that the current leader has
 - both

When a new leader comes to power, it will enforce all followers' log entries to be consistent with its own by:
 - initializing `nextIndex` for each follower to be the last log entry of the leader
 - if at `prevLogIndex` a follower's log entry is different from the leader's, then the follower rejects `AppendEntries` RPC
 - on rejection of `AppendEntries` RPC, the leader decrements `nextIndex` for the rejecting follower, and then send `AppendEntries` again until eventually a matching log entry between this follower and the leader is found

### 2. Possible optimization for the back-and-forth rejection (i.e. how to make leader back up faster on rejection)
```
  S1: 4 5 5      4 4 4      4
  S2: 4 6 6  or  4 6 6  or  4 6 6
  S3: 4 6 6      4 6 6      4 6 6

  (S3 is leader for term 6, S1 comes back to life)
```
When there's a conflict at `prevLogIndex`, follower includes the conflicting term (`conflictingTerm`) and index of its first log entry of that term in `AppendEntries` response (`conflictingIndex`)
 - if the leader doesn't have log entries of conflicting term (case on the left), we know all logs of this term on the follower should be truncated, so bypass this term by setting `nextIndex` to `conflictingIndex`
 - if the leader has log entries of conflicting term (case in the middle), we know the follower's logs of this term after the leader's last index of this term should be truncated, so move `nextIndex` to leader's last entry of `conflictingTerm`

### 3. How do we tell a Raft server to apply log entries to its state
This work is done by a single goroutine. A condition variable (`sync.Cond`) is used to signal the goroutine that's responsible for applying log entries:
 - for leader, `cond.Signal()` is called in another goroutine that periodically detects if there are replicated entries with index larger than `committedIndex` (i.e. majority of `matchIndex` is larger than `committedIndex`)
 - for follower, `cond.Signal()` is called in `AppendEntries` when it finds `LeaderCommit` is larger then its own `committedIndex`
 When there's no signal, the goroutine used to apply entries simply does `Wait()` (release the condition variable lock and get suspended).

## Safety restrictions

### 1. What's the restriction on election? What specific case does it try to deal with?
In `RequestVotes` arguments, a candidate includes the last index of its log entries and the term of this entry. When another server decides whether to vote for this candidate, it must ensure either term of candidate's last entry is greater than the voter's last entry, or they are of the same term but the candidates log entries are longer (i.e. the candidate needs to be as least as up-to-date as others to win the election). This restriction is made to guarantee that selected leader has all committed log entries.

### 2. What's the restriction on committing a log entry? What specific case does it try to deal with?
The leader can only commit log entries that belong to the current term. When a slow follower falls into minority when a log entry is committed & applied, but is then selected as new leader, it will overwrite the committed log entries and force other servers to be compliant with itself. This way different servers will execute different command sequences.

Example:

![alt text][commit-current-term]

[commit-current-term]: /images/raft/commit-current-term.jpg "Commit current term"

(a) S1 is leader
(b) S1 crashes and S5 is elected leader
(c) S5 crashes, S1 restarted and become leader again and replicated log entry at index 2 to majority; but the log entry cannot be committed because if S1 crashes S5 re-elected as leader and replicated log 3 (in (d)), log entry 2 will be overwritten