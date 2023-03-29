---
title: "Logs"
date: 2022-04-30T16:46:09-03:00
categories: ["algorithms", "distributed systems", "data-structures"]
draft: false
---

# What is a log

A log is just a immutable sequence of records wih strong ordering semantics that can be used to provide durability, replication and to model consensus. It is usually a 0 indexed file that new entries are appended to because expensive disk seeks can usually be avoided when appending to a file[^ostep-hard-disk-drives].

<p align="center">
<img src="https://user-images.githubusercontent.com/17282221/168452116-a751154f-ec58-4a65-91f5-a90269529963.png" />
</p>

> Not to be confused with the type of logs most people are used to: application logs that are meant to be read by humans although application logs are a degenerative case of the log we are talking about[^i-love-logs].

What i called a record is a entry in the log. The entry can be anything in any format.

# You have seen a log before

## Databases

Postgres uses a write-ahead log to ensure data is not lost if a crash happens[^postgres-write-ahead-log], to enable replication and change data capture. Tables and indexes are modified only after the change been written to the log in which case if a crash happens, the log can be used to go back to a valid state.

> Datomic takes it to the next level by being a log-centric database[^rich-hickey-descontructing-the-database].

## File systems

Some file systems known as journaling file systems[^journaling-file-system] write changes to a log before actually applying them to the internal file system structures to enable crash recovery and avoid data corruption.

## Distributed systems

Distributed systems such as Kafka which considers a message as accepted by the cluster after the quorum of in-sync replicas(configuration dependent) have written the message to their log[^kafka-the-definitive-guide]

## Consensus

Consensus algorithms such as Raft[^raft-paper] aka replicated state machines[^replicated-state-machines].

[^ostep-hard-disk-drives]: [Operating systems: Three easy pieces - Hard Disk Drives](https://pages.cs.wisc.edu/~remzi/OSTEP/file-disks.pdf)
[^postgres-write-ahead-log]: [Postgres write-ahead log](https://www.postgresql.org/docs/current/wal-intro.html)
[^change-data-capture]: [Change data capture](https://en.wikipedia.org/wiki/Change_data_capture)
[^journaling-file-system]: [Journaling file-system](https://en.wikipedia.org/wiki/Journaling_file_system)
[^kafka-the-definitive-guide]: [Kafka: The Definitive Guide: Real-Time Data and Stream Processing at Scale](https://www.confluent.io/resources/kafka-the-definitive-guide-v2/)
[^replicated-state-machines]: [Replicated state machines](https://en.wikipedia.org/wiki/State_machine_replication)
[^raft-paper]: [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
[^i-love-logs]: [I Love logs](https://www.confluent.io/ebook/i-heart-logs-event-data-stream-processing-and-data-integration/)
[^rich-hickey-descontructing-the-database]: [Rich Hickey: Deconstructing the Database](https://www.youtube.com/watch?v=Cym4TZwTCNU)
