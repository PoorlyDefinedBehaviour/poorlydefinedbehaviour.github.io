---
title: "Paper notes: Sagas"
date: 2024-10-08T22:00:00-03:00
categories: ["papers", "notes"]
draft: true
---

[Sagas](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)

Sagas are long lived transactions that were broken into a sequence of smaller transactions that can be interleaved with other transactions in the database. The paper defines long lived transactions as transactions that can take hours or days to run because they process lots of data or perform complex computations.  

Since transactions usually lock the objects they access, long lived transactions can degrade the performance of the system because other transactions will be blocked for a long time.  

If one of the transactions in a saga fails, one or more compensating actions must be performed to undo the changes made by the transactions that executed successfully before the one that failed.  

Other transactions may view partial results of other sagas.  

> If the database is immutable and the transaction is read-only, long lived transactions could just read from an immutable view of the database, or a previous version while the database allows mutations to occur as usual because mutations create a new version of the database. For example in [Datomic](https://docs.datomic.com/client-tutorial/history.html#as-of-query) the database can be queried at a specific point in time, meaning the client has an immutable view of the database at a specific point in time.

The paper talks about sagas in the context of a RDBMS but it is common to see sagas being used in the context of distributed services with their own databases.