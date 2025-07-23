---
title: "Generating tests from a TLA+ specification"
date: 2025-07-20T00:00:00-00:00
categories: ["distributed systems", "formal methods", "tla+", "testing"]
draft: false
---

## Formal methods

Formal methods are techniques – usually based on mathematics – used for the specification, analysis and verification of software and hardware.

## Why would anyone be interested in formal methods?

The main reasons to consider formal methods are verifying that a design is correct, verifying that a change is correct, analysing a system to learn about it and developing an intuition about the system.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXeItStmwei-06gHeLcBczgo2RjIandofBbnfTbpzcvatoJg0EwARIEXpCLo5hGK0bDcYvoggtmOmhsA-HXdCW858FWr7Ry6gUxnP4o0NN8AkXNeDpy7gpSLSC5z5mXyktIDxl1HSg?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
Formal methods is the umbrella term that contains several techniques inside of it
</div>

We'll be considering only model checking because I believe it is the easiest to get started with and get a return on the investment quickly.

## Formal methods for design

A design doc is meant to be a recipe for a piece of software, it usually contains a problem overview, one or more possible solutions for the problem and a description of how each solution would be implemented.

The description of the solutions in the design doc always contains several assumptions about how its dependencies and how the implementation will work. These assumptions may be correct, partially correct or incorrect.

A design doc author could enumerate all assumptions and requirements the solution aims to solve manually in a matrix – and some people actually do – but that's clearly too much effort for most projects.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXctXqlBFwrpTOCBcRBVluZwmFv_DVcTnPaQa9VCsR6TBoCpelWrgr76flhLhaabJyTalMCBiRO4RxborjeTsyueLiunK8CfuBnUu7h5tHYSHdy8atC_oaquIPeTGjhiyPfrCw-VVQ?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

Formal methods can be included in the phase where the design doc is created. Instead of only describing the solution using words, build a specification of the solution and how it interacts with other parts of the system using a [specification language](https://en.wikipedia.org/wiki/Specification_language). The specification is then checked against the invariants you define by a model checker, the model checker outputs a sequence of steps that lead to an invariant being violated known as a counter example. The counter example is used for debugging and reproducing the issue.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfgD4kUdKnf3cRMGHAwsRUfEhwGSXn5SnIwtynkELQ7YHDPxcvZvc5Pvhoj9M8H7yzLDIwX6S8sJADtBTeamNkOy6tUmD9F166S8Tct1BLAL9pD9viFs1dAU14T3nFiyugTzERfqg?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
A snippet of a sequence of states leading to a bug being found
</div>

## TLA+

[TLA+](https://github.com/tlaplus/tlaplus) is a formal specification language that can be used in the design process of a new system, to find flaws in existing systems, to build a better understanding of existing systems and to prove correctness of algorithms.

TLA+ is a simple language that consists mostly of variables and actions that assign values to variables.

Check the [cheatsheet](https://ocamlpro.com/assets/pdf/tla-cheat-sheet-v1.pdf).

Here's an example specification of a clock that starts between 1 and 2, goes up to 12 then goes back to 1. There's one invariant checking that at all times the clock hours are between 1 and 12 and a temporal property saying that the clock must reach midnight (12) eventually.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXcaPNHdwqn9eeSxyfRFJLXSugxRWEdhO9sP43brXI4V5Nfdxbki2ZcE4EA_lRqmPoJPBKSqK_q6recJlI26h1Ie5RO5n-314l0zyK0kYQmrigALK3sg9F8FReC4WSj-GgH7SLUR?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
A TLA+ spec modelling a clock that starts between 1 and 2, counts up to 12 then goes back to 1
</div>

**\[]** Is read as always

**<>** is read as eventually

## Formal methods for analysing a system

When a system already exists, it can be valuable to write a specification based on the existing system to gain a deeper understanding of how the system works. Everything must be explicit in a TLA+ specification, it's common to start writing a specification and realize that you don't actually understand how something works when you actually need to define its behavior. Sometimes new bugs may even be found in the already existing system after writing a specification.

## Formal methods for testing a system

A model checked TLA+ specification will visit every possible behavior in the system, model checking is exhaustive up to a max number of inputs that the specification author defines. After writing a specification, it's possible to use the specification in combination with the model checker to generate test cases for the real implementation.

MongoDB has a blog [post](https://www.mongodb.com/company/blog/engineering/conformance-checking-at-mongodb-testing-our-code-matches-our-tla-specs) where they talk about generating test cases for a real system using one of their TLA+ specs. They got 100% test coverage by doing that compared to 21% from manually written tests and 91% from [AFL](https://github.com/google/AFL).

The paper [Model-guided Fuzzing of Distributed Systems](https://arxiv.org/pdf/2410.02307) talks about using a TLA+ specification state space to guide a fuzzer to test the real implementation of the specifications. The authors found bugs in etcd-raft and RedisRaft by using this technique.

## Experimenting with formal methods

Since most of the team was not familiar with TLA+, we decided to focus the efforts on a small subset of new a new project and to put some effort into knowledge sharing to make it easier for people in the team to get comfortable with the tool.

The chosen project provides a secret store type of interface where services can store secrets such as passwords and retrieve them when needed. Operations such as creating and deleting secrets were async at first and the service responsible for managing secrets – known as the secrets service – was using the [outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html) to maintain consistency.

We started exploring TLA+ while working on the outbox so we decided to model the outbox interactions to make sure our design was sound. We did not model only the basic operations such as adding an operation to the queue but also the invariants that must always be true for the system to behave correctly.

One of the invariants is that there may be only one operation in progress at a time per secret. This invariant was modelled in our specification as follows:

```
SecretMetadataHasPendingStatusWhenTheresAnOperationInTheQueue ==
      \* For all secrets
      \A secret \in Secrets:
        LET SecretIsInPendingQueue == 
            \E i \in DOMAIN queue.pending:
                queue.pending[i] = secret
        IN
        \* the secret is either in the outbox queue
        \/ /\ SecretIsInPendingQueue
           \* and the secret metadata status is Pending
           /\ SecretMetadataHasPendingStatus(secret)
        \* or the secret is not in the queue
        \/ /\ ~SecretIsInPendingQueue
           \* and the secret metadata status is not Pending
           /\ ~SecretMetadataHasPendingStatus(secret)
```

<div style="font-style:italic;text-align:center;font-size:90%">
An invariant that says: For every existing secret, either there's an operation for it in the outbox queue and the secret status is Pending or there's no operation for the secret in the queue and the secret status is not Pending
</div>

An invariant that says that something **bad** never happens is known as a _safety_ property. TLA+ also allows us to specify properties that say that something **good _eventually happens_**, known as _liveness_ properties.

Our specification defines that when an operation for a secret is added to the outbox queue, the operation will eventually succeed and the secret status will be set to **Succeeded.**

```
EventuallyEveryMetadataStatusIsReady ==
    \* For all secrets
    \A secret \in Secrets:
        \* Transform the queue.pending tuple into a set
        LET PendingQueueSet == {queue.pending[i]: i \in DOMAIN queue.pending} IN
            \* If the secret is in the pending queue
            /\ secret \in PendingQueueSet
            \* Leads to it eventually
            ~>
                \* Being in the processed queue
                /\ secret \in queue.processed
                \* And removed from the pending queue
                /\ secret \notin PendingQueueSet
                \* And the secret being in the metadata table with status "Succeeded"
                /\ \E metadata \in db.secret_metadata:
                    /\ metadata.name = secret
                    /\ metadata.status = "Succeeded"
```

<div style="font-style:italic;text-align:center;font-size:90%">
A temporal property that says: when an operation for a secret is in the outbox queue, eventually the operation will be processed (retried if needed) and the secret will be moved to the Succeeded status
</div>

We didn't get to use the specification to generate tests yet but we were already able to reap one of the benefits of formal modelling: a deeper understanding of the system being specified. While writing the specification, we were forced to answer several questions since we needed to make the behavior explicit.

## Catching logic bugs with formal methods

Some months ago, a team hit a bug where two regions sharing the same SQL database but with caches that were unique per instance would lead to valid tokens being cached as invalid. A cache invalidation bug.

After the bug was found, a basic specification that models the behavior of the systems with the shared database was written in a few minutes. The spec is high-level, doesn't contain implementation details at all and found the same bug immediately.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXeJ3m9tmhogFXjCyGTOfo7BRwXJcMd_XDXMZ7giGIb3AaQRNmhHiRh76Wuo-zKTZ5dkHpaRlPSQWSGeNJqnCUp814RuOzhH6n7X5EK_Lez0-7172ONzf0oTuVl4XWBF37ZX-GwCMw?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
A counter example produced by the model checker. The counter example is a list of steps that lead to the invariant being violated
</div>

We are also exploring [deterministic simulation testing](https://poorlydefinedbehavior.github.io/posts/deterministic_simulation_testing/) as a [lightweight formal method](https://dl.acm.org/doi/pdf/10.1145/3477132.3483540) but we haven't got far enough with it yet.

## Hands-on prerequisites: Install TLA+ tools

- Download the TLA+ toolbox from the official [website](https://lamport.azurewebsites.net/tla/toolbox.html)

- Open VSCode and install the TLA+ extension

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfAFLtt-YdcocuzT1en_PV0z2QwGd-cOxgVS0LO8bze7TVEJnFehI0AbK6_d3hx71E9OdaBk_RJXah8d-BD6jaJuaB6ZrxT1rPixONj_9gC4HRka8U9j49MJB8CTWvS7_c58YrJ0Q?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

- Create a file called **_spec.tla_**. This is where you'll write your spec.

- Write the following boilerplate in your **_spec.tla_** file:

```go
---- MODULE spec ----
EXTENDS TLC, Integers, FiniteSets, Naturals

  \* Write your spec here

====
```

## Hands-on: Clock

Given the following boilerplate, write a specification for a clock that counts from 0 to 12 and then resets back to 0.

```go
---- MODULE spec ----
EXTENDS TLC, Integers, FiniteSets, Naturals

  \* Write your spec here

====
```

## Hands-on: Waves

Given a group of people forming a circle

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXeCF2dOHv3MAIK0wxT2SMf5LsDFGFpHI4Y5FxRF5V6sjCVdpPf-FR_ZSevkMi5vBf53MAUbc4hxc2Khz18P560BoUCfK4Jjl5UhpAXsrzAmy2-kTqOCWhUqbsUVK-Bk_Aq3_r_A9Q?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

After choosing one person in the circle at random, the person must high five the person to its left, then duck and then high five the person to its right. A person that's ducking cannot get up or high five someone until another high five has happened. High fives happen in counterclockwise direction starting from the person that was chosen in the beginning. The wave ends when the person that started the process is reached again.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXd58ia-whZxyjz6qGHYeR-oVlVY9QvGwU3Ebez2VtDpeScdihlZaiVj24Mjl4DYO8apP6xssPCH8fMhmVfYp3AjVBsJKXpG_TL0H4cr-1Mam0R0gMvPC6-tWY1rA3W5tSOm_geYdg?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
One of the people in the circle is chosen at random
</div>

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXcN5uSwI5buxEjxchg72VE6el9okhJTexvRPQDzHVMH0pgE3VQ3xMvWdlm3jz-B8XFwBflTajn5VWI9wwT9TRwgo_SeXTB6xSzicTHsztBe3u8-PYt8QVTaxZdzNPSxLc63w6ZHfg?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
The person high fives the person to their left
</div>

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXds3R2PPHvp7e2P8RCoN5lwniDt9BR9wNKsYrcGMa2imQpwxOBff70yPVOmRBsuqvQd4vqHCeUucEDMvYQD55zoghbWbv81CR6aVVfDdypUefrzDG-h2GBsjP45c8eOpUuoP0Cl?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
The person ducks after high fiving the person to their left
</div>

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXda3dXEFL2S8QnDZAtsI5u6-xDz2adYFa1suKYmOF2AVLt22KmpRl9yymAXWjjZbR-IoaC-6nSmUE7fxMZTWEyf4rxV5DwgR5MhPT--vS0FcOzN3phwz4B9GT1OmnBYj87eOcJXhg?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
The next person in the circle in counterclockwise direction repeats the process
</div>

## Hands-on: Die hard

The die hard problem is a problem from the movie [Die hard with a vengeance](https://www.imdb.com/title/tt0112864/). There is a 5 liters water jug, a 3 liters water jug, a very precise scale (that doesn't show the weight of items on top of it), and a faucet with an infinite stream of water. There's a bomb ready to explode in a few seconds and the only way to stop it is to put exactly 4 liters of water on the scale.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXcnEshTTwEwf26K2Ozzaebe9kaFOrFL_zZ7F2f57yPfJkxsX9voX_R8gM3JuJM7Xu81S9K2a85CM2OnDBN2tP-TUR1NHReqwHul77ykDcsuQFqOYSgiOaiXX1pPpVdTDtzMM4DpyA?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
Source: WikiHow
</div>

The jugs start empty. The possible actions are:

- Fill one of the jugs with water – remember that the jugs are unmarked so the only way to know how much water is in one of them is filling them until they're completely full.

- Remove water from one of the jugs

- Move water from one jug to the other, remember that each jug has a limited capacity.

Design a specification to find the shortest sequence of steps to solve the die hard problem. The solution is found when one of the jugs has exactly 4 liters of water in it.

## Hands-on: Two phase commit

[Two phase commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) is a distributed commit protocol where all participants in a transaction should eventually commit or rollback.

Two phase commit is a protocol that can be found in many real worlds systems, some of them being:

MySQL: Transactions in MySQL clusters

Postgres: Meant to be used by external transaction managers

The protocol works in the following manner: one node is a designated coordinator, which is the master site, and the rest of the nodes in the network are designated the participants. The protocol assumes that:

1. there is stable storage at each node with a write-ahead log,

2. no node crashes forever,

3. the data in the write-ahead log is never lost or corrupted in a crash, and

4. any two nodes can communicate with each other.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXemlWnZY1sUnpWBldf3LoiAwAYsWFyKpVG4Hg-oXDHJ8B9OI0WpIlydqxm7BybmwYroBrneIEpE7agGQkBU7E5AerOH87FjoPw4raVv7eeDKW-f9i-21Tz_5i-Sw8EuLA7ttskt?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
A complete execution of the two phase commit protocol. Source: Wikipedia
</div>

Choose between specifying the two phase commit protocol or creating your own protocol for distributed transactions. Imagine that there's a single transaction and it must commit or abort on all nodes. Remember that nodes may crash at any time.

## Hands-on: Single decree paxos

[Paxos](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf) is a protocol heavily used in the industry to implement replicated systems when the systems need to decide on a value and the value that's decided must be the same for all the nodes involved.

Paxos is used in many real world systems, some of them being:

Cassandra: [For lightweight distributed transactions](https://www.datastax.com/blog/lightweight-transactions-cassandra-20)

Google Spanner: [For state machine replication](https://static.googleusercontent.com/media/research.google.com/en//archive/spanner-osdi2012.pdf)

Failure model:

- Processes crash at any time

- Messages can be lost

- Messages can be reordered

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXeRo6VT0bK8hbIawlZhs2RfgK8oGVU0TrzSAQiYqaAU0QFMwFIY6j1FMG-TkfT5N2fyIKFnfretooFD4Zfi9eV0e_i7f1YHaWHwwtuyHJYiGHeDqcxgV2vkMXL-fOzrjsxNzvH1?key=CZuYqiYDpsuzokR8yCzjpg">
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
A brief summary of the single decree paxos protocol
</div>

Invent and specify a protocol to make 3 nodes decide on the same value. The value must be proposed by one of the nodes and the other nodes cannot know which value will be proposed ahead of time.

## Does my implementation match the specification?

You just wrote a specification that models your system at an abstraction level that doesn't care about things such as I/O. The specification is a high level version of the implementation that captures the main properties of the system you're trying to build.

```
Next ==
  \/  \E instance \in Instances, namespace \in Namespaces, value \in Values, data_key_id \in DataKeyIds:
               Encrypt(instance, namespace, value, data_key_id)
  \/  \E instance \in Instances, namespace \in Namespaces, data_key_id \in DataKeyIds:
               Decrypt(instance, namespace, data_key_id)
  \/  \E instance \in Instances, namespace \in Namespaces:
               RotateDataKeys(instance, namespace)
  \/  \E instance \in Instances:
               \/  CacheRemoveExpiredEntries(instance)
               \/  RestartInstance(instance)
  \/  AdvanceTime
  \/  Terminating
```

<div style="font-style:italic;text-align:center;font-size:90%">
The next state relation. Defines which actions can be taken at each step in the model checking process
</div>

The model checker uses the specification to check possibly thousands of behaviors, generating a graph of the reachable states. A state is the set of variables with their values.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXciCPmiGSuIAmcOJC-YVUkgaoEeFAVHZsjleA-OZDZM-Uyd5CBqlFFWIILRuJXQd0quX2Mprsl4rlLQV_Huf0yaJhm__7oZMh_HKfNJRY_ACIecb3NTBzJxITYbc3C7OIyqx28J2Q?key=2053luUD1bm7MwEnF9Bh-w"/>
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
A graph generated by model checking the a specification of a module that uses the outbox pattern
</div>

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXclXIeQzzb98-Sv03tljxXTJDf7fGrZdMiMO_WeyDMljkRO_eU2S4MiEpHgzbjv1Sd2frEIfUk5RPBuvIe5Xa0Vvaa6o-R5dOQt6TB6lcuFZhO9qAcD0SlBYIs8W6yvAhhruwOSnw?key=2053luUD1bm7MwEnF9Bh-w"/>
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
The number of times each action was executed during the model checking of a specification
</div>

Every invariant is satisfied by the behaviors allowed by the specification and you believe your spec gives you **_enough_** confidence to move on to the implementation.

```
DataKeyUidUniqueness ==
   \A dk1 \in database:
       {dk2 \in database : dk2.uid = dk1.uid} = {dk1}

DataKeyUniqueness ==
   \A dk1, dk2 \in database:
       (/\ dk1.uid /= dk2.uid
        /\ dk1.namespace = dk2.namespace)
        /\ dk1.label = dk2.label
        /\ dk1.active
        /\ dk2.active
           =>
               Assert(FALSE, <<"duplicated data key", dk1, dk2>>)

CacheConsistency ==
   \A instance \in Instances:
       /\  \A entry \in instances[instance].cache.by_id:
               \E row \in database:
                   /\ entry.namespace = row.namespace
                   /\ entry.data_key.uid = row.uid
       /\  \A entry \in instances[instance].cache.by_label:
               \E row \in database:
                   /\ entry.namespace = row.namespace
                   /\ entry.data_key.uid = row.uid

DataKeysAreCachedByLabelOnlyAfterCautionPeriod ==
   \A instance \in Instances:
       \A entry \in instances[instance].cache.by_label:
           Assert((now - entry.data_key.created) > CautionPeriod,
                 <<"data key cached before caution period",
                  "now", now,
                  "entry.data_key.created", entry.created,
                  "CautionPeriod", CautionPeriod>>)
```

<div style="font-style:italic;text-align:center;font-size:90%">
A few examples of invariants defined in a specification
</div>

The implementation is coming along just fine but you can't help but think how do you know that the implementation implements what's in the specification. If the implementation follows the specification precisely, the implementation will be a refinement of the specification – it will allow every behavior allowed by the specification and possibly more behaviors.

```go
func (s *EncryptionManager) RotateDataKeys(ctx context.Context, namespace string) error {
  s.log.Info("Data keys rotation triggered, acquiring lock...")

  s.mtx.Lock()
  defer s.mtx.Unlock()

  s.log.Info("Data keys rotation started")
  err := s.store.DisableDataKeys(ctx, namespace)
  if err != nil {
    s.log.Error("Data keys rotation failed", "error", err)
    return err
  }

  s.dataKeyCache.flush(namespace)
  s.log.Info("Data keys rotation finished successfully")


  return nil
}
```

<div style="font-style:italic;text-align:center;font-size:90%">
A snippet of the implementation of the system that was modelled in the specification
</div>

Maybe at this point you decide to put the implementation and the specification side by side and pattern match between them.

```
RotateDataKeys(instance, namespace) ==
    /\  DisableDataKeys(namespace)
    /\  DataKeyCacheFlush(instance, namespace)
    /\  UNCHANGED now
```

<div style="font-style:italic;text-align:center;font-size:90%">
A snippet of one of the actions defined in the specification
</div>

That can get you quite far but having the implementation execute the same sequence of actions that the specification allows would increase the confidence that the implementation actually implements the specification.

The output graph generated in the model checking process can be used to build every possible path from the initial state to the final state. Similar to what has been done here, the paths can be used to exercise the implementation in a normal test.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXcvNCOUD-wUJISaRpa8Vz7hxsNr_TKzm8WCaGvjMVgFrQP9PbTwfsRS-YFO8EOwK4gGI7eZesvX5e4NveuCnc2FNHbE98qk3bDtKBO4xtjCjUuP5OCpNufLqpD-X87hE5Er3tqZ?key=2053luUD1bm7MwEnF9Bh-w"/>
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
A diagram generated in the model checking process. The diagram contains every possible state and how to get there
</div>

The generated graph is stored in a _.dot_ file, the test starts by reading and parsing the graph stored in the file.

```go
func TestParseDotFile(t *testing.T) {
  file, err := os.Open("./testdata/spec2.dot")
  require.NoError(t, err)
  defer file.Close()

  buffer, err := io.ReadAll(file)
  require.NoError(t, err)
  graphAst, _ := gographviz.ParseString(string(buffer))
  graph := gographviz.NewGraph()
  if err := gographviz.Analyse(graphAst, graph); err != nil {
    panic(err)
  }
}
```

<div style="font-style:italic;text-align:center;font-size:90%">
A snippet of a test that reads and parses a file containing the graph of reachable states generated by model checking a specification
</div>

Then for each possible path starting from the initial state, we apply each action in the order they appear in the path. Here's a few examples of the possible sequence of actions:

    [Init AdvanceTime AdvanceTime RestartInstance]
    [Init Encrypt AdvanceTime AdvanceTime RestartInstance]
    [Init Encrypt AdvanceTime AdvanceTime CacheRemoveExpiredEntries]
    [Init Encrypt AdvanceTime AdvanceTime RotateDataKeys]
    [Init Encrypt AdvanceTime AdvanceTime Decrypt]
    [Init AdvanceTime AdvanceTime CacheRemoveExpiredEntries]
    [Init AdvanceTime AdvanceTime RotateDataKeys]
    [Init AdvanceTime AdvanceTime Encrypt]

<div style="font-style:italic;text-align:center;font-size:90%">
Examples of sequences of actions that will be used to exercise the implementation
</div>

Each sequence is a test case to which the implementation is exercised against.

```go
for path := range visitor.Iter() {
  check(t, path)
}
```

For each action in the path generated based on the specification, an action that's equivalent to it is applied to the implementation. It boils down to a loop and a switch case.

```go
func check(t *testing.T, path []visitor.EdgeWithLabel) {
  defer func() {
    if t.Failed() {
      pathJSONbytes, _ := json.MarshalIndent(path, "", "  ")
      fmt.Printf("%s\n\n", string(pathJSONbytes))
    }
  }()

  var (
    encryptionManager      *EncryptionManager
    encryptedValuesByKeyId map[string][]map[string][]byte
    fakeTime               *fakeTime
  )

  for _, node := range path {
    switch node.Label {
    case "Init":
     // Initialize the system under test
     ...
    case "Encrypt":
     // Perform the Encrypt operation
     ...
    case "Decrypt":
     // Perform the Decrypt operation
     ...
    case "RotateDataKeys":
     // Perform the RotateDataKeys operation
     ...
    case "AdvanceTime":
     // Perform the AdvanceTime operation
     ...
    case "CacheRemoveExpiredEntries":
     // Perform the CacheRemoveExpiredEntries operation
     ...
    case "RestartInstance":
     // Perform the RestartInstance operation
     ...
    default:
      panic(fmt.Sprintf("unhandled label, did you forget a switch case?: %+v", node))
    }
  }
}
```

Assertions are added at any place you seem fit. It's recommended to assert that the system is in a valid state and that every response received – if any – makes sense at each step.

## Generating tests for snapshot isolation implementation

In databases, snapshot isolation is a transaction isolation level that guarantees that a transaction will have the illusion of working on a snapshot of the database, the snapshot is taken at the time the transaction starts.

Specifications for snapshot isolation can be easily found but I've decided to write my own in TLA+. After writing the specification I decided to implement an example database with snapshot isolation in Go. After I was done with the implementation and had written a few tests I decided to generate the test cases using the state graph generated during the model checking process.

```go
for path := range visitor.Iter() {
   db := NewDatabase()
   // Map from model tx id to impl tx
   modelMap := make(map[int]int)

   for _, node := range path {
     switch node.Label {
     case "TxRead":
       modelTxId := int(mustParseInt(node.Args[0]))
       key := node.Args[1]
       db.Read(modelMap[modelTxId], key)
     case "TxBegin":
       modelTxId := int(mustParseInt(node.Args[0]))
       txId := db.BeginTransaction()
       modelMap[modelTxId] = txId
     case "TxCreate":
       modelTxId := int(mustParseInt(node.Args[0]))
       key := node.Args[1]
       value := node.Args[2]
       db.Write(modelMap[modelTxId], key, value)
     case "TxUpdate":
       modelTxId := int(mustParseInt(node.Args[0]))
       key := node.Args[1]
       value := node.Args[2]
       db.Write(modelMap[modelTxId], key, value)
     case "TxDelete":
       modelTxId := int(mustParseInt(node.Args[0]))
       key := node.Args[1]
       db.Delete(modelMap[modelTxId], key)
     case "TxRollback":
       require.Equal(t, 1, len(node.Args))
       modelTxId := int(mustParseInt(node.Args[0]))
       db.Rollback(modelMap[modelTxId])
     case "TxCommit":
       require.Equal(t, 1, len(node.Args))
       modelTxId := int(mustParseInt(node.Args[0]))
       db.Commit(modelMap[modelTxId])
     case "Terminating":
       // no-op
     default:
       panic(fmt.Sprintf("unhandled action: %+v", node))
     }
   }
 }
```

<div style="font-style:italic;text-align:center;font-size:90%">
A snippet of the code used to test the database with snapshot isolation with the paths generated from the model checking the specification. Some code has been removed to make the snippet shorter
</div>

The generated test cases resulted in 100% code coverage without any special effort.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXcNlUuBudJsjL451VG1S7rIT5-k_Nw2V1IbuiU6XrNem55GwjWQAHtUtSN-tdIe8sJAs2TDQNU6HSAz1kuIWpmmoKVTgnOeep6ud8s24ZYiRMHdFpu5GK_Qhyu-PbpqFULPUHKNnQ?key=2053luUD1bm7MwEnF9Bh-w"/>
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
An image showing a code coverage report with 100% coverage. Green means the code has been exercised during testing
</div>

I also got a specification for snapshot isolation from [github.com/will62794/snapshot-isolation-spec](http://github.com/will62794/snapshot-isolation-spec). It was the first one I found and I decided to generate test cases from it as well. The following is the first result I got after running the tests.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXcSJN66BMnpXmFfc8uxDFz7N30usVNRRQTpf8XISBd11Izl0NVYD9HIvmg8V0CkgIVoG06sRb1sViekaS0Z2Oy_lY1Gu68zhfFitUuy8F5TaObSSYqtBoe8-wJXjHeq4aDLm0Fc?key=2053luUD1bm7MwEnF9Bh-w"/>
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
An image showing a code coverage report with 78% coverage. Green means the code has been exercised during testing
</div>

After that I decided to exercise the snapshot isolation implementation from the blog post [Implementing MVCC and major SQL transaction isolation levels](https://notes.eatonphil.com/2024-05-16-mvcc.html). I got 85.8% coverage using the specification I wrote. Note that Phil's implementation contains more than just the code for the snapshot isolation level so code paths related to other isolation levels are not covered – as expected.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXe3H79Zc8NIfD2oVm80O9NHUhAgO8X-YeA1wsS0TU9u2k4YJxWhvBzEnhhZEiU-Nk6EU9_rJ-HtvVrAUdbGIs5AhUlbPFYzbPK8mPeoCHE9lGNHTIpaF_NqioCZgeh9SmvYHO5U?key=2053luUD1bm7MwEnF9Bh-w"/>
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
An image showing a code coverage report with 85.8% coverage. Green means the code has been exercised during testing
</div>

I also ran the same tests but using will62794's specification this time and got 80.1% coverage in the first try.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXddrKebU47GS14iw7F9IlhiBPxZ2CioerYIh1N5zHsli7YuYS-mmdCCpbeIMsRvc2viH76KvceVrkCCCtKns9dRjpS8bPGyF7TYfbIbEuvFG0PR83ukAe1hj3_R7t9yP7SV-trRuw?key=2053luUD1bm7MwEnF9Bh-w"/>
</div>
<div style="font-style:italic;text-align:center;font-size:90%">
An image showing a code coverage report with 80.1% coverage. Green means the code has been exercised during testing
</div>

## Generating tests for the encryption manager

The encryption manager is a package used for envelope encryption. A specification modeling some of the operations supposed by the encryption manager was used to generate test cases for the implementation. The test cases resulted in 60.3% coverage.

```
Next ==
   \/  \E instance \in Instances, namespace \in Namespaces, value \in Values, data_key_id \in DataKeyIds:
           Encrypt(instance, namespace, value, data_key_id)
   \/  \E instance \in Instances, namespace \in Namespaces, data_key_id \in DataKeyIds:
           Decrypt(instance, namespace, data_key_id)
   \/  \E instance \in Instances, namespace \in Namespaces:
           RotateDataKeys(instance, namespace)
   \/  \E instance \in Instances:
           \/  CacheRemoveExpiredEntries(instance)
           \/  RestartInstance(instance)
   \/  Terminating
```

<div style="font-style:italic;text-align:center;font-size:90%">
The next state relation of the encryption manager specification
</div>

Most of the uncovered lines are error handling paths or unsupported operations. Both cases were not modeled in the specification.

<div align="center">
<img style="width:50%" src="https://lh7-rt.googleusercontent.com/docsz/AD_4nXfk0U7dE9-pHpbhbZ1mA9fSD1RJxvefadquvQX0xFBmmQWb_23NBvyUtsUEGlhXpzeA8TOTVjswqN_REVx86-xoa991-8xYFv5hT6C_KJPNXsbOSZyiPrrdsDvbnNUMyu8X3raIaA?key=2053luUD1bm7MwEnF9Bh-w"/>
</div>

<div style="font-style:italic;text-align:center;font-size:90%">
An image showing a code coverage report with 60.3% coverage. Green means the code has been exercised during testing
</div>
