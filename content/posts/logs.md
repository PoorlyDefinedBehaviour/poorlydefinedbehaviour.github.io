---
title: "Logs"
date: 2022-04-30T16:46:09-03:00
categories: ["algorithms", "distributed", "data-structures"]
draft: false
---

# What is a log

A log is just a immutable sequence of records. It is usually 0 indexed file where new entries are appended because expensive disk seeks can usually be avoided when appending to a file[^ostep-hard-disk-drives].

<p align="center">
<img src="https://user-images.githubusercontent.com/17282221/168452116-a751154f-ec58-4a65-91f5-a90269529963.png" />
</p>

> Not to be confused with the type of logs most people are used to: application logs that are meant to be read by humans.

What i called a record is a entry in the log. The entry can be anything in any format.

# You have seen a log before

Logs are used everywhere, from databases like Postgres[^postgres-write-ahead-log] where logs enable recovery after crashes, leader-follower replication and change data capture[^change-data-capture], file systems[^journaling-file-system] to avoid data corruption, distributed systems such as Kafka which considers a message as accepted by the cluster after the quorum of in-sync replicas(configuration dependent) have written the message to their log[^kafka-the-definitive-guide] and consensus algorithms such as Raft[^raft-paper] aka replicated state machines[^replicated-state-machines].

[^ostep-hard-disk-drives]: [Operating systems: Three easy pieces - Hard Disk Drives](https://pages.cs.wisc.edu/~remzi/OSTEP/file-disks.pdf)
[^postgres-write-ahead-log]: [Postgres write-ahead log](https://www.postgresql.org/docs/current/wal-intro.html)
[^change-data-capture]: [Change data capture](https://en.wikipedia.org/wiki/Change_data_capture)
[^journaling-file-system]: [Journaling file-system](https://en.wikipedia.org/wiki/Journaling_file_system)
[^kafka-the-definitive-guide]: [Kafka: The Definitive Guide: Real-Time Data and Stream Processing at Scale](https://www.confluent.io/resources/kafka-the-definitive-guide-v2/)
[^replicated-state-machines]: [Replicated state machines](https://en.wikipedia.org/wiki/State_machine_replication)
[^raft-paper]: [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)
