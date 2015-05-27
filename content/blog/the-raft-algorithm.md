+++
date = "2015-05-27T22:24:16+02:00"
draft = false
title = "The Raft Algorithm"
type = "post"
series = ["Raft 101"]
slug="somethingelse"
tags = ["go", "golang", "programming", "interfaces", "raft"]
[author]
	name = "Nils Lagerkvist"
	uri = "https://plus.google.com/+NilsLagerkvist"

+++

The Raft consensus algorithm provides a understandable, easy to use, and generic way to distribute a state machine in a cluster. The Raft is a successor, or alternative to an algorithm called Paxos.

A node in the cluster can have three states: follower, candidate, or leader. Consensus in the cluster is maintained by the leader, and is responsible for log replication. To become the leader a node will change it's status from follower to candidate.

### Leader Election
The leader in the cluster send heartbeat messages to the followers. If a follower does not receive a heartbeat within the election timeout it will transition state from follower to candidate. The candidate start a new election term and increases the *term* counter, vote for itself and send a *RequestVote* message to the cluster members. A member node will vote on the first candidate that sends a *RequestVote* message, and it will only vote once during a *term*. At this point there are three scenarios in the cluster:

1. The candidate receive a message from a leader with equal or higher *term* - and transition from candidate to flower.
2. The candidate receive a majority of the votes - and transition from candidate to leader.
3. The candidate's election times out - and transition form candidate to follower.

### Log Replication
All changes goes through the leader, leader get proposal, creates a log entry (uncommited), then replicate the entry to the followers, when a majority has written the log entry and commits the entry (the leader state has now changed), the leader notifies the followers that the entry is committed. At this point the cluster is in consensus.

### Log replication
Once there is a leader the cluster can move forward. All changes goes through the leader, however followers can propose a change to the leader. When the leader is updated it creates a log entry, which is uncommited; the leader replicates the log entry to the followers in an *AppendEntries* message. When a majority of followers has written the log entry, the leader will commit the pending entry; the leader state has now changed and notifies the followers that the entry is committed. At this point the cluster is in consensus and has moved forward.

### Cluster Split
Cluster consensus requires at least two nodes; and thus three nodes are needed to create a resilient cluster. In case of a cluster split there will be one part that can achieve consensus, and move forward. However, there are two scenarios that are interesting to look into.

#### Leader in Majority Part
The leader is in the part of the cluster that can reach consensus - That part will continue to work and move forward. In the other part a new leader can not be elected as consensus can not be reached. When the cluster split is resolved the log entries will replicate to the follower that was on the "wrong" side of the split.

#### Leader in Minority Part
The leader is in the part of the cluster that can not reach consensus - Proposed changes to the original leader will be entered in the log and replicated to the followers; however the change can not be committed and that part of the cluster can not move forward. At the same time, on the other side of the cluster a leader election will start, due to the lack of leader, and the term will increment and a leader will be elected. The increment in election term here is important. Now when there a leader, and enough nodes for consensus, this part of the cluster can start to move forward.

When the cluster split is resolved there will be two leaders. However, the leader with the lower *term* will now have uncommitted entries in the log. When it reseive a message from the leader with a higher *term* it will roll back the log entries and start to commit the log entries from the leader with higher *term*.

Next part will be a practical introduction to the Raft algorithm in etcd.
