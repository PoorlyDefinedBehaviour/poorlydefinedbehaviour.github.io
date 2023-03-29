---
title: "Notes taken from the Raft paper"
date: 2022-03-04T17:48:19-03:00
categories: ["today-i-learned", "algorithms", "consensus", "distributed systems", "papers"]
draft: false
---

# **R**eplicated **A**nd **F**ault **T**olerant

Raft is a consensus algorithm for managing a replicated log.

The authors claim Raft to be more understandable than Paxos
because Raft separates the key elements of consensus

- Leader election
- Log replication
- Safety

and enforces a stronger degree of coherency to reduce the number of states
that must be considered.

Raft also includes a new mechanism for changing cluster membership.

## What is a consensus algorithm

Consensus algorithms allow a collection of machines to work as a coherent group
that can survive the failures of some of its members.

## Why Raft was created

Paxos, another consensus algorithm, is the most popular algorithm when talking
about consensus algorithms. Most implementations of consensus are based on Paxos
or influenced by it.

The problem is that Paxos is hard to understand and its architecture does not support pratical systems without modifications.

The point of Raft is to be a consensus algorithm that is as efficient as Paxos
but easier to understand and implement.

## What is unique about Raft

Raft is similar in many ways to existing consensus algorithms (most notably, Oki and Liskov's Viewstamped Replication), but it has several new features:

**Strong leader**: Raft uses a stronger form of leadership than other consensus algorithms. For example, log entries only flow from the leader to other servers.
This simplifies the management of the replicated log.

**Leader election**: Raft uses randomized timers to elect leaders. This makes conflicts easier and faster to resolve.

**Membership changes**: Raft's mechanism for changing the set of servers in the cluster uses a new _joint consensus_ approach where the majorities of two different configurations overlap during transitions. This allows the cluster to continue operating normally during configuration changes.

## Raft in the wild

https://github.com/tikv/raft-rs  
https://github.com/hashicorp/raft  
https://github.com/sofastack/sofa-jraft  
https://github.com/eBay/NuRaft

# Replicated state machines

States machines on a collection of servers compute identical copies of the same
state and can continue operating even if some of the servers are down.
Replicated state machines are used to solve a variety of fault tolerance problems in distributed systems.

## How the consensus algorithms work with replicated state machines

Replicated state machines are typically implemented using a replicated log.

The consensus algorithm manages a replicated log containg state machine
commands from clients. The state machines process identical sequences of commands
fro the logs, so they produce the same outputs.

The consensus algorithm module on the server receives commands from clients
and adds them to its log. It communicates with the consensus algorithm modules on other servers to ensure that every log eventually contains the commands in the same order even if some of the servers fail.

Once commands are properly replicated (every server has the same commands in the same order), each server's state machine processes them and the outputs are returned to the clients. As a result, the servers appear to form a single, highly
reliable state machine.

## Typical properties of consensus algorithms

**Safety**: They ensure that an incorrect result is never returned under all non-[Byzantine conditions](https://en.wikipedia.org/wiki/Byzantine_fault), including networks delays, partitions, and packet lost, duplication and reordering.

**Availability**: They are fully functional as the majority of the servers are functional. For example, a cluster of five servers can tolerate the failure of two servers. Servers that failed may rejoin the cluster after recovering.

**Consistency**: They do not depend on timing to ensure the consistency of the logs.

**Performance**: A command can complete as soon as a majority of the cluster has responded has accepted the command.

## Examples of replicated state machines

https://www.cs.cornell.edu/courses/cs6464/2009sp/lectures/16-chubby.pdf  
https://github.com/apache/zookeeper

# Paxos

Created by Leslie Lamport, Paxos first defines a protocol capable of reaching
agreement on a single decision, such as a single replicated log entry. This is known as single-decree Paxos. Paxos then combines multiple instances of this protocol to facilitate a series of decisions such as a log. This is known as multi-Paxos.

## Paxos criticism

Paxos is difficult to understand. The full explanation is opaque and because of that a lot of effort is necessary to understand it and only a few people succeed in understanding it.

Because of its difficulty, there have been several attempts to simplify Paxos explanation. These explanation focus on the single-decree subset and even then, they are still hard to understand.

The second problem with Paxos is that there is no widely agreed-upon algorithm for multi-Paxos. Lamport's descriptions are mostly about single-decree Paxos.

The third problem is that Paxos uses a peer-to-peer approach at its core. If a series of decisions must be made, it is simpler and faster to first elect a leader, then have the leader coordinate the decisions.

Because of these problems, pratical systems implementations begin with Paxos, discover the difficulties in implementing, and then develop a significantly different architecture.

# The Raft consensus algorithm

Raft implements consensus by first electing a distinguished **leader**, then giving the leader complete responsibility for managing the replicated log.
The leader accepts log entries from clients, replicates them on other servers, and tells servers when it is safe to apply log entries to their state machines.
A leader can fail or become disconnected from the other servers, in which case a new leader is elected.

Raft decomposes the consensus problem into three subproblems:

**Leader election**: A new leader must be chosen when an existing leader fails.

**Log replication**: The leader must accept log entries from clients and replicate them accross the cluster, forcing the other logs to agree with its own.

**Safety**: If any server has applied a particular log entry to its state machine, then no other server may apply a different command for the same log index.

## Raft basics

A raft cluster contains several servers, five is a typical number, which allows the system to tolerate two failures. At any given time each server is in one of three states: **leader**, **follower** or **candidate**.

**leader**: In normal operation there is exactly one leader and all of the other servers are followers. The leader handles all client requests.

**follower**: Followers simply respond to requests from leaders and candidates, if a client contacts a follower, the request is redirected to the leader.

**candidate**: A candidate is a follower that wants to become a leader.

Raft divides time into _terms_ of arbitraty length numbered with consecutive integers. Each term begins with an _election_, in which one or more candidates attempt to become leader. If a candidate wins the election, then it serves as leader for the rest of the term. If the election results in a split vote, the term ends with no leader and a new term with a new election begins shortly.

Each server stores a _current term_ number, which increases monotonically over time. Current terms are exchanged whenever servers communicate and if one server's current term is smaller than the other's, then it updates its current term to the larger value. If a candidate or leader discovers that is term is out of date, it immediately reverts to follower state. If a server receives a request with a stale term number, it rejects the request.

## Raft servers communication with remote procedure calls

Raft servers coomunicate using remote procedure calls. The basic consensus algorithm requires two types of RPCs:

**RequestVote**: **RequestVote** RPCs are initiated by candidates during elections.

**AppendEntries**: **AppendEntries** RPCs are initiated by leaders to replicate log entries and to provide a form of heartbeat.

Servers retry RPCs if they do not receive a response in a timely manner.

## Leader election

Raft uses a heartbeat mechanism to trigger leader election. When servers start up, they begin as followers and remain in the follower state as long as it receives valid RPCs from a leader or candidate. Leaders send periodic heartbeats (**AppendEntries** RPCs that carry no log entries) to all followers in order to maintain their authority. If a follower receives no communication over a period of time called the _election timeout_, then it assumes there is no viable leader and begins an election to choose a new leader.

To begin an election, a follower increments its current term and transitions to candidate state. It then votes for itself and issues **RequestVote** RPCs in parallel to each server in the cluster. A candidate continues in the candidate state until one of three things happens:

**Wins election**: The candidate wins the election and becomes the new leader.

A candidate wins an election if it receives votes from a majority of the servers in the full cluster for the same term. Each server votes for at most one candidate in a given term, on a first-come-first-served basis.
Once a candidate wins an election, it becomes the leader and sends heartbeat messages to all of the other servers to establish its authority and prevent new elections.

**Other server wins election**: Another candidate wins the election and becomes the new leader.

During an election, a candidate may receive an **AppendEntries** RPC from another server claiming to be the leader. If the leader's term is at least as large as the candidate's current term, then the candidate recognizes the leader as legitimate and returns to follower state.
If the request term is smaller than the candidate's current term, the request is rejected as described in [Raft basics](#Raft-basics).

**Election timeout**: A period of time goes by with no winner.

If many followers become candidates at the same time, votes could be split so that no candidate obtains a majority. When this happens, each candidate will time out and start a new election by increasing its term and iniating another round of **RequestVote** RPCs.

Elections timeouts are chosen randomly from the range like 150..300ms to ensure that split votes are rare and that hey are resolved quickly.

Each candidate gets a randomized election timeout at the start of an election, and it waits for the timeout to elapse before starting a new election.

## Election restriction

Raft uses the voting process to prevent a candidate from winning an election unless its log contains all commited entries from previous terms. When a **RequestVote** RPC is made, the candidate includes the index and term of the last entry in its log, the server that receives the request (aka the voter) denies the request if its own log is more up-to-date than of the candidate. If the logs have last entries with different terms, then the log with the later term is more up-to-date. If the logs end with the same term, then whichever log is longer is more up-to-date.

## Log replication

Once a leader has been elected, it begins servicing client requests that each contain a command to be executed by the replicated state machines. When a request is received, the leader appends the command to its log as a new entry, then issues **AppendEntries** RPCs in parallel to each of the servers to replicate the entry. Only after the entry has been replicated, the leader applies the entry to its state machine and returns the result of that execution to the client. The leader **retries** **AppendEntries** RPCs until all followers eventually store all log entries.

Logs are composed of sequentially numbered entries. Each entry contains the term in which it was created and a command for the state machine. The leader decides when it is safe to apply a log entry to the state machines, entries that have been applied to the state machines are called _committed_. Raft guarantees that committed entries are durable and will eventually be executed by all of the available state machines. A log entry is commited once the leader that created the entry has replicated it on a majority of the servers. The leader keeps track of the highest index it knows to be committed, and it includes that index in future **AppendEntries** RPCs so that other server eventually find out. Once a follower learns that a log entry is committed, it applies every entry up to the entry to its local state machine.

<pre>
command  term
   │      │
   │      │   0    1    2    3    4    5    6    7      log index
   │      │ ┌────┬────┬────┬────┬────┬────┬────┬────┐
   │      ├─► 1  │ 1  │ 1  │ 2  │ 3  │ 3  │ 3  │ 3  │ │
   └──────┴─►x=3 │y=1 │y=9 │x=2 │x=0 │y=7 │x=5 │x=4 │ │  leader
            └────┴────┴────┴────┴────┴────┴────┴────┘

            ┌────┬────┬────┬────┬────┐
            │ 1  │ 1  │ 1  │ 2  │ 3  │                │
            │x=3 │y=1 │y=9 │x=2 │x=0 │                │
            └────┴────┴────┴────┴────┘                │
                                                      │
            ┌────┬────┬────┬────┬────┬────┬────┬────┐ │
            │ 1  │ 1  │ 1  │ 2  │ 3  │ 3  │ 3  │ 3  │ │
            │x=3 │y=1 │y=9 │x=2 │x=0 │y=7 │x=5 │x=4 │ │
            └────┴────┴────┴────┴────┴────┴────┴────┘ │
                                                      │  followers
            ┌────┬────┐                               │
            │ 1  │ 1  │                               │
            │x=3 │y=1 │                               │
            └────┴────┘                               │
                                                      │
            ┌────┬────┬────┬────┬────┬────┬────┐      │
            │ 1  │ 1  │ 1  │ 2  │ 3  │ 3  │ 3  │      │
            │x=3 │y=1 │y=9 │x=2 │x=0 │y=7 │x=5 │      │
            └────┴────┴────┴────┴────┴────┴────┘

            │                                  │
            └──────────────────────────────────┘
                      committed entries
</pre>

Raft mantains the following properties:

- If two entries in different logs have the same index and term, then they store the same command.

- If two entries in different logs have the same index and term, then the logs are identical in all preceding entries.

## How log inconsistencies are handled

**TLDR**: Leader overwrites follower logs if they are out of sync.

In Raft, the leader handles inconsistencies by forcing the follower's logs to duplicate its own. This means that conflicting entries in follower logs will be overwritten with entries from the leader's log. To bring a follower's log into consistency with its own, the leader must find the latest log entry where the two logs agree, delete any entries in the follower's log after that point, and send the follower all of the leader's entries after that point. The leader maintains a _nextIndex_ for each follower, which is the index of the next log the leader will send to that follower. When a leader first comes to power, it initializes all nextIndex values to the index just after the last one in its log.

# Safety

**TLDR**: Only servers that contain all of the entries commited in previous terms may become leader at any given term.

There's a problem with the Raft description so far:  
For example, a follower might be unavailable while the leader commits several log entries, then it could be elected leader and overwrite these entries with new ones. As a result, different state machines might execute different command sequences. This is fixed by adding a restriction on which servers may be elected leader. The restriction ensures that the leader for any given term contains all of the entries commited in previous terms.

## Restriction on committing logs

Raft never commits log entries from previous terms by counting replicas. Only log entries from the leader's current term are commited by counting replicas.

# Follower and candidate crashes

If a follower or candidate crashes, then future **RequestVote** and **AppendEntries** RPCs sent to it will fail. Raft handles these failures by retrying indefinitely, if the crashed server restarts, then the RPC will complete successfully. If a server crashes after completing an RPC but before responding, then it will receive the same RPC again after it restarts. Raft RPCS are idempotent, so this causes no harm. If a follower receives an **AppendEntries** request that includes log entries already present in its log, it ignores those entries in the new request.

# Timing and availability

Raft will be able to elect and maintain a steady leader as long as the system satifies the following _timing requirement_:

_broadcast_time <= election_timeout <= MTBF_

**broad_cast_time** is the average time it takes a server to send RPCs in parallel to every server in the cluster and receive their responses.

**election_timeout** is the election timeout described in [Leader election](#Leader-election).

**MTBF** is the average time between failures for a single server.

The broadcast time should be an order of magnitude less than the election timeout so that leaders can reliably send the heartbet messages required to keep followers from starting elections.

The election timeout should be a few orders of magnitude less than MTBF so that the system makes steady progress.

# Cluster membership changes

Configuration changes are incorporated into the Raft consensus algorithm.
In Raft the cluster first switches to a transitional configuration called _joint consensus_, once the joint consensus has been committed, the system then transitions to the new configuration.  
The join consensus combines both the old an new configurations:

- Log entries are replicated to all servers in both configurations.
- Any server from either configuration may serve as leader.
- Agreement for elections and entry commitment requires separate majorities from both the old and new configurations.

The joint consensus allows individual servers to transition between configurations at different times without compromising safety.

## How cluster configurations are stored and communicated

Cluster configurations are stored and communicated using special entries in the replicated log. When the leader receives a request to change the configuration from config_old (the current configuration) to config_new (the new configuration), it stores a tuple (config_old, config_new) as a log entry and replicates that entry. Once a given server adds the new configuration entry to its log, it uses that configuration for all future decisions even if the entry has not been committed yet.

# Log compaction

**TLDR**: We don't have infinite memory, discard logs that aren't needed anymore.

Raft's log grows during normal operation to incorporate more client requests, but in a practical system, it cannot grow without bound. As the log grows longer, it occupies more spaces and takes more time to replay.

Snapshotting is the simplest approach to compaction. In snapshotting the entire current system state is written to a _snapshot_ on stable storage, then the tire log up to that point is discarded. Snapshotting is used in Chubby and ZooKeeper and in Raft as well.

## The basic idea of snapshotting in Raft:

**TLDR**: Add the current machine state to the log and delete all logs used to get to this state. The snapshot should also be written to stable storage.

Each server takes snapshots independently, covering just the commited entries in its log. Most of the work consists of the state machine writing its current state to the snapshot. Raft also includes a small amount of metadata in the snapshot: the _last included index_ which is the index of the last entry in the log that the snapshot replaces (the last entry the state machine had applied), and the _last included term_ which is the term of the entry. The snapshot also includes the latest configuration. Once a server completes a snapshot, it may delete all log entries up through the last included index, as well as any prior snapshot.

<pre>
  0    1    2    3    4    5    6     log index
┌────┬────┬────┬────┬────┬────┬────┐
│ 1  │ 1  │ 1  │ 2  │ 3  │ 3  │ 3  │  │ before snapshot
│x=3 │y=1 │y=9 │x=2 │x=0 │y=7 │x=5 │  │
└────┴────┴────┴────┴────┴────┴────┘


         snapshot
┌────────────────────────┬────┬────┐
│last included index: 4  │ 3  │ 3  │  │
│last included term: 3   │y=7 │x=5 │  │
│state machine state:    ├────┴────┘  │ after snapshot
│ x = 0                  │            │
│ y = 9                  │            │
└────────────────────────┘

│                        │
└────────────────────────┘
    committed entries
</pre>

## When a new follower joins the cluster

The way to bring a new foller up-to-date is for the leader to send it a snapshot over the network. The leader uses a new RPC called **InstallSnapShot** to send snapshots to followers that are too far behind because they are new or because they are too slow.

When a follower receives a snapshot with this RPC:

If the snapshot contains new information on in the follower's log, the follower discards and replaces its log with the snapshot.

If the snapshot contains only a prefix of its log, then log entries covered by the snapshot are deleted bu entries following the snapshot are still valid and must be retained.

# Client interaction

Clients of Raft send all of their requests to the leader.

## How clients find the leader

When a client starts up, it connects to a randomly-chosen server. If the client's choice is not the leader, that server will reject the client's request and supply information about the most recent leader it has heard from.

## Command deduplication

For example, if the leader crashes after committing the log entry but before responding to the client, the client will retry the command with a new leader, causing to be executed a second time.

The solution is for clients to assign unique serial numbers to every command. Then, the state machine tracks the latest serial number processed for each client, along with the associated response. If it receives a command whose serial number ha already been executed, it responses immediatelly without re-executing the request.

## Read-only operations should not return stale data

When a leader responds to a request, its log could be outdated with a new leader had been elected in the meantime.

To avoid this situation:

A leader must have the latest information on which entries are committed. Because of that, each leader commits a blank _no-op_ entry into the log at the start of its term.

A leader must check whether it has been deposed before processing a read-only request. Raft handles this by having the leader exchange heartbeat messages with a majority of the cluster before responding to read-only request.

# References

In Search of an Understandable Consensus Algorithm
(Extended Version) - https://raft.github.io/raft.pdf  
Designing for Understandability: The Raft Consensus Algorithm - https://www.youtube.com/watch?v=vYp4LYbnnW8
"Raft - The Understandable Distributed Protocol" by Ben Johnson (2013) - https://www.youtube.com/watch?v=ro2fU8_mr2w  
https://github.com/hashicorp/raft  
MIT 6.824: Distributed Systems (Spring 2020) Lecture 6: Fault Tolerance: Raft - https://www.youtube.com/watch?v=64Zp3tzNbpE&list=PLrw6a1wE39_tb2fErI4-WkMbsvGQk9_UB&index=6
