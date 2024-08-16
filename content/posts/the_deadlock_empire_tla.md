---
title: "Model checking the deadlock empire"
date: 2024-08-15T00:00:00-00:00
categories: ["formal methods", "distributed systems"]
draft: false
---

[Non atomic instructions](https://deadlockempire.github.io/#T2-Expansion)

There's two threads executing the following code:

```c#
a = a + 1;
if (a == 1) {
  critical_section();
}
```

Since the `a` increment is not atomic, conceptually, it is like setting a temporary variable to the value of `a`-- `tmp = a` and then setting `a` to the temporary variable value incremented by 1 -- `a = tmp + 1`.

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Integers, FiniteSets

(*--algorithm spec

variables
    a = 0,
    threads = 1..2,
    Enterd_critical_section = {};

define
    OnlyOneThreadEntersCriticalSection == Cardinality(Enterd_critical_section) <= 1
end define;

process Thread \in threads
variables
    tmp = 0;
begin
Load:
    tmp := a;
Store:
    a := tmp + 1;
CriticalSectionCheck:
    if a = 1 then
        Enterd_critical_section := Enterd_critical_section \union {self};
    end if;
end process;

end algorithm; *)
====
```

| State                                                 | Thread a             | Thread b             | Description                                                                                 |
| ----------------------------------------------------- | -------------------- | -------------------- | ------------------------------------------------------------------------------------------- |
| a = 0                                                 | Init                 | Init                 | Both threads are at the initial state                                                       |
| a = 0, thread1.tmp = 0                                | Load                 | Init                 | Thread a stores the value of `a` in a temporary variable                                    |
| a = 0, thread1.tmp = 0, thread2.tmp = 0               | Load                 | Load                 | Thread b stores the value of `a` in a temporary variable                                    |
| a = thread1.tmp + 1, thread1.tmp = 0, thread2.tmp = 0 | Store                | Load                 | Thread a sets `a` to `0 + 1` using the temporary variable                                   |
| a = thread2.tmp + 1, thread1.tmp = 0, thread2.tmp = 0 | Store                | Store                | Thread b sets `a` to `0 + 1` using the temporary variable                                   |
| a = 1                                                 | CriticalSectionCheck | Store                | Thread a enters the critical section since `a` is equal to `1`                              |
| a = 1                                                 | CriticalSectionCheck | CriticalSectionCheck | Thread b enters the critical section since `a` is equal to `1` at the same time as thread 1 |

[Boolean Flags Are Enough For Everyone](https://deadlockempire.github.io/#2-flags)

```c#
flag = false

while (true) {
  while (flag != false) {
    ;
  }
  flag = true;
  critical_section();
  flag = false;
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Integers, FiniteSets

(*--algorithm spec
variables
    threads = 1..2,
    flag = FALSE,
    threads_in_criticial_section = {};

define
    OnlyOneThreadEntersCriticalSection == Cardinality(threads_in_criticial_section) <= 1
end define;

process Thread \in threads
begin
SpinLock:
    while flag do
        skip;
    end while;
SetFlag:
    flag := TRUE;
CriticalSection1:
    threads_in_criticial_section := threads_in_criticial_section \union {self};
CriticalSection2:
    threads_in_criticial_section :=  threads_in_criticial_section \ {self};
UnsetFlag:
    flag := FALSE;
end process;

end algorithm; *)
====
```

| State        | Thread a         | Thread b         | Description                                                                                                                                        |
| ------------ | ---------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| flag = false | Init             | Init             | Both threads are at the initial state                                                                                                              |
| flag = false | SpinLock         | Init             | Thread a enters the spinlock                                                                                                                       |
| flag = false | SpinLock         | SpinLock         | Thread b enters the spinlock                                                                                                                       |
| flag = true  | SpinLock         | SetFlag          | Thread b sees that `flag` is `false` so it leaves the spinlock and sets `flag` to `true`                                                           |
| flag = true  | SetFlag          | SetFlag          | Thread a was in the spinlock and read the value of `flag` before thread 2 set it to `true`. Thread a leaves the spinlock and sets `flag` to `true` |
| flag = true  | CriticalSection1 | CriticalSection1 | Both threads enter the critical section at the same time                                                                                           |

[Simple counter](https://deadlockempire.github.io/#3-simpleCounter)

```c#
while (true) {
  counter++;
  if (counter == 5) {
    critical_section();
  }
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Integers, FiniteSets

(*--algorithm spec
variables
    threads = {3, 5},
    threads_in_critical_section = {},
    counter = 0;

define
    MutualExclusion == Cardinality(threads_in_critical_section) <= 1
end define;

process Thread \in threads
variables
    tmp = 0,
    done = FALSE;
begin
Loop:
    while ~done do
Load:
    tmp := counter;
Store:
    counter := tmp + 1;
EnterCriticalSection:
    if counter = self then
        threads_in_critical_section := threads_in_critical_section \union {self};
        done := TRUE;
    end if;
LeaveCriticalSection:
    threads_in_critical_section := threads_in_critical_section \ {self};
    end while;
end process;
end algorithm; *)
====
```

| State       | Thread a             | Thread b             | Description                                                                         |
| ----------- | -------------------- | -------------------- | ----------------------------------------------------------------------------------- |
| counter = 0 | Loop                 | Loop                 | Thread a updates the counter until it reaches 3 while thread 2 is in the Loop state |
| counter = 0 | Load                 | Loop                 |                                                                                     |
| counter = 1 | Store                | Loop                 |                                                                                     |
| counter = 1 | Loop                 | Loop                 |                                                                                     |
| counter = 1 | Load                 | Loop                 |                                                                                     |
| counter = 2 | Store                | Loop                 |                                                                                     |
| counter = 2 | Loop                 | Loop                 |                                                                                     |
| counter = 2 | Load                 | Loop                 |                                                                                     |
| counter = 3 | Store                | Loop                 |                                                                                     |
| counter = 3 | EnterCriticalSection | Loop                 | Counter reached 3, so thread 1 enters the critical section                          |
| counter = 3 | LeaveCriticalSection | Loop                 | While thread 1 is in the critical section, thread 2 updates the counter to 5        |
| counter = 3 | LeaveCriticalSection | Load                 |                                                                                     |
| counter = 4 | LeaveCriticalSection | Store                |                                                                                     |
| counter = 4 | LeaveCriticalSection | Loop                 |                                                                                     |
| counter = 4 | LeaveCriticalSection | Load                 |                                                                                     |
| counter = 5 | LeaveCriticalSection | Store                |                                                                                     |
| counter = 5 | LeaveCriticalSection | EnterCriticalSection | And thread 2 enters the critical section while thread 1 is still there              |

> Thread a is in the LeaveCriticalSection state but it has not executed the step yet.

[Confused counter](https://deadlockempire.github.io/#4-confusedCounter)

Thread A

```c#
business_logic();
first++;
second++;
if (second == 2 && first != 2) {
  Debug.Assert(false);
}
```

Thread B

```c#
business_logic();
first++;
second++;
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Integers

(*--algorithm spec
variables
    first = 0,
    second = 0,
    assertion_failed = FALSE;

define
    AssertionNeverFails == assertion_failed = FALSE
end define;

process ThreadA = "a"
variables
    tmp = 0;
begin
LoadFirst:
    tmp := first;
StoreFirst:
    first := tmp + 1;
LoadSecond:
    tmp := second;
StoreSecond:
    second := tmp + 1;
CriticalSection:
    if second = 2 /\ first # 2 then
        assertion_failed := TRUE;
    end if;
end process;

process ThreadB = "b"
variables
    tmp = 0;
begin
LoadFirst:
    tmp := first;
StoreFirst:
    first := tmp + 1;
LoadSecond:
    tmp := second;
StoreSecond:
    second := tmp + 1;
end process;
end algorithm; *)
====
```

| State                                                   | Thread a        | Thread b    | Description                                                                                                                                                                   |
| ------------------------------------------------------- | --------------- | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| first = 0, second = 0                                   | LoadFirst       | LoadFirst   | Both threads load `first` into `thread1.tmp`                                                                                                                                  |
| first = 0, second = 0, thread1.tmp = 0, thread2.tmp = 0 | StoreFirst      | StoreFirst  | Thread a updates `first` by setting it to `thread1.tmp + 1`                                                                                                                   |
| first = 1, second = 0, thread1.tmp = 0, thread2.tmp = 0 | LoadSecond      | StoreFirst  | Thread a loads `second` into `thread1.tmp`. Note that Thread b is still in the `StoreFirst` state since it has not executed yet                                               |
| first = 1, second = 0, thread1.tmp = 0, thread2.tmp = 0 | StoreSecond     | StoreFirst  | Thread a updates `second` by setting it to `thread1.tmp + 1`                                                                                                                  |
| first = 1, second = 1, thread1.tmp = 0, thread2.tmp = 0 | CriticalSection | StoreFirst  | Thread a moves to the `CriticalSection` state but does not execute yet                                                                                                        |
| first = 1, second = 1, thread1.tmp = 0, thread2.tmp = 0 | CriticalSection | StoreFirst  | Thread `2` updates `first` by setting it to `thread2.tmp + 1`. Note that `thread2.tmp` is still `0` since the variable was set in a previous state before Thread b got paused |
| first = 1, second = 1, thread1.tmp = 0, thread2.tmp = 0 | CriticalSection | LoadSecond  | Thread b loads `second` into `thread2.tmp`                                                                                                                                    |
| first = 1, second = 1, thread1.tmp = 0, thread2.tmp = 1 | CriticalSection | StoreSecond | Thread b updates `second` by setting it to `thread2.tmp + 1`. Note that `thread2.tmp` is `1`                                                                                  |
| first = 1, second = 1, thread1.tmp = 0, thread2.tmp = 1 | CriticalSection | Done        | Thread a resumes executions and the condition in the if the statement succeeds                                                                                                |

[Insuffient lock](https://deadlockempire.github.io/#L1-lock)

Two threads use a mutex to protect `i`. The mutex works as expected, the problem is that exists an execution order where thread 1 hits the assertion.

```c#
// Thread a
while (true) {
  Monitor.Enter(mutex);
  i = i + 2;
  critical_section();
  if (i == 5) {
    Debug.Assert(false);
  }
  Monitor.Exit(mutex);
}

// Thread b
while (true) {
  Monitor.Enter(mutex);
  i = i - 1;
  critical_section();
  Monitor.Exit(mutex);
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Integers

(*--algorithm spec
variables
    lock = FALSE,
    assertion_failed = FALSE,
    i = 0;

define
    AssertionNeverFails == assertion_failed = FALSE
end define;

process ThreadA = "a"
begin
Loop:
while TRUE do
AcquireLock:
    await lock = FALSE;
    lock  := TRUE;
Modify:
    i := i + 2;
If:
  if i = 5 then
    assertion_failed := TRUE;
  end if;
ReleaseLock:
    lock := FALSE;
end while;
end process;

process ThreadB = "b"
begin
Loop:
while TRUE do
AcquireLock:
    await lock = FALSE;
    lock := TRUE;
Modify:
    i := i - 1;
ReleaseLock:
    lock := FALSE;
end while;
end process;
end algorithm; *)
====
```

| State | Thread a    | Thread b    | Description                                                                                               |
| ----- | ----------- | ----------- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| i = 0 | Loop        | Loop        | Threads start                                                                                             |
| i = 0 | AcquireLock | AcquireLock | Thread a acquires the lock repeatedly until `i` reaches `4`. Thread b is stuck trying to acquire the lock |
| i = 0 | Modify      | AcquireLock |                                                                                                           |
| i = 2 | If          | AcquireLock |                                                                                                           |
| i = 2 | ReleaseLock | AcquireLock |                                                                                                           |
| i = 2 | AcquireLock | AcquireLock |                                                                                                           |
| i = 2 | Modify      | AcquireLock |                                                                                                           |
| i = 2 | If          | AcquireLock |                                                                                                           |
| i = 4 | ReleaseLock | AcquireLock |                                                                                                           |
| i = 4 | ReleaseLock | Modify      | Thread b finally acquires the lock                                                                        |
| i = 3 | ReleaseLock | If          |                                                                                                           |
| i = 3 | ReleaseLock | ReleaseLock |                                                                                                           |
| i = 3 | AcquireLock | AcquireLock |                                                                                                           |
| i = 3 | Modify      | AcquireLock | Thread a acquires the lock again                                                                          |
| i = 5 | If          | AcquireLock | Thread                                                                                                    | `i` is equal to `5` this time, thread 1 hits the assertion |

[Deadlock](https://deadlockempire.github.io/#L2-deadlock)

Note that the order in which each thread tries to acquire the locks is different.

```c#
// Thread a
Monitor.Enter(mutex);
Monitor.Enter(mutex2);
critical_section();
Monitor.Exit(mutex);
Monitor.Exit(mutex2);

// Thread b
Monitor.Enter(mutex2);
Monitor.Enter(mutex);
critical_section();
Monitor.Exit(mutex2);
Monitor.Exit(mutex);
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC

(*--algorithm spec
variables
    mutex1 = FALSE,
    mutex2 = FALSE;

process ThreadA = "a"
begin
AcquireLock1:
    await mutex1 = FALSE;
    mutex1 := TRUE;
AcquireLock2:
    await mutex2 = FALSE;
    mutex2 := TRUE;
ReleaseLocks:
    mutex1 := FALSE;
    mutex2 := FALSE;
end process;

process ThreadB = "b"
begin
AcquireLock1:
    await mutex2 = FALSE;
    mutex2 := TRUE;
AcquireLock2:
    await mutex1 = FALSE;
    mutex1 := TRUE;
ReleaseLocks:
    mutex2 := FALSE;
    mutex1 := FALSE;
end process;
end algorithm; *)
====
```

| State                          | Thread a     | Thread b     | Description                                                                     |
| ------------------------------ | ------------ | ------------ | ------------------------------------------------------------------------------- |
| mutex1 = FALSE, mutex2 = FALSE | AcquireLock1 | AcquireLock1 | Both threads start acquiring the locks                                          |
| mutex1 = TRUE, mutex2 = FALSE  | AcquireLock2 | AcquireLock1 | Thread a acquires the first lock and tries to acquire the second                |
| mutex1 = TRUE, mutex2 = TRUE   | AcquireLock2 | AcquireLock2 | Thread b acquires the second lock before thread 1 is able to acquire it         |
| mutex1 = TRUE, mutex2 = TRUE   | Deadlock     | Deadlock     | No thread can progress because one thread holds the lock the other thread needs |

[A More Complex Thread](https://deadlockempire.github.io/#L3-complexer)

```c#
// Thread a
while (true) {
  if (Monitor.TryEnter(mutex)) {
    Monitor.Enter(mutex3);
    Monitor.Enter(mutex);
    critical_section();
    Monitor.Exit(mutex);
    Monitor.Enter(mutex2);
    flag = false;
    Monitor.Exit(mutex2);
    Monitor.Exit(mutex3);
  } else {
    Monitor.Enter(mutex2);
    flag = true;
    Monitor.Exit(mutex2);
  }
}

// Thread b
while (true) {
  if (flag) {
    Monitor.Enter(mutex2);
    Monitor.Enter(mutex);
    flag = false;
    critical_section();
    Monitor.Exit(mutex);
    Monitor.Enter(mutex2);
  } else {
    Monitor.Enter(mutex);
    flag = false;
    Monitor.Exit(mutex);
  }
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Sequences, FiniteSets, Integers

NULL == <<"-1", -1>>

(*--algorithm spec
variables
    mutex = NULL,
    mutex2 = NULL,
    mutex3 = NULL,
    flag = FALSE;
macro enter(mutex, thread) begin
    await mutex = NULL \/ mutex[1] = thread;
    if mutex = NULL then
        mutex := <<thread, 1>>;
    else
        mutex := <<thread, mutex[2] + 1>>;
    end if;
end macro;

macro exit(mutex, thread) begin
    assert mutex[1] = thread;
    assert mutex[2] > 0;

    if mutex[2] = 1 then
        mutex := NULL;
    else
        mutex := <<mutex[1], mutex[2] - 1>>;
    end if;
end macro;

macro try_enter(mutex, thread) begin
    if mutex = NULL then
        mutex := <<thread, 1>>;
        try_enter_result := TRUE;
    elsif mutex[1] = thread then
        mutex := <<thread, mutex[2] + 1>>;
        try_enter_result := TRUE;
    else
        try_enter_result := FALSE;
    end if;
end macro;

process thread_a = "a"
variables
    try_enter_result = FALSE;
begin
Loop:
while TRUE do
TryEnterMutex: try_enter(mutex, "a");
CheckEnterMutex:
    if try_enter_result then
        EnterMutex3: enter(mutex3, "a");
        EnterMutex: enter(mutex, "a");
        ExitMutex: exit(mutex, "a");
        EnterMutex2: enter(mutex2, "a");
    else
        Else_EnterMutex2: enter(mutex2, "a");
        SetFlag: flag := TRUE;
        ExitMutex2: exit(mutex2, "a");
    end if;
end while;
end process;

process thread_b = "b"
begin
Loop:
while TRUE do
    CheckFlag:
    if flag then
        EnterMutex2:enter(mutex2, "b");
        EnterMutex: enter(mutex, "b");
        SetFlag:flag := FALSE;
        ExitMutex: exit(mutex, "b");
        ExitMutex2: enter(mutex2, "b");
    else
        Else_EnterMutex: enter(mutex, "b");
        Else_SetFlag: flag := FALSE;
        Else_ExitMutex: exit(mutex, "b");
    end if;
end while;
end process;
end algorithm; *)
====
```

> Variables that didn't change on transition to a new state were omitted.

| State                                                                     | Thread a         | Thread b        | Description                                                                                          |
| ------------------------------------------------------------------------- | ---------------- | --------------- | ---------------------------------------------------------------------------------------------------- |
| flag = false, mutex = NULL, mutex2 = NULL, mutex3 = NULL                  | Loop             | Loop            | Both threads start running                                                                           |
| ...                                                                       | TryEnterMutex    | Loop            | Thread `a` moves to the `TryEnterMutex` state but has not executed yet                               |
| ...                                                                       | TryEnterMutex    | CheckFlag       | Thread `b` moves to the `CheckFlag` state                                                            |
| ...                                                                       | TryEnterMutex    | Else_EnterMutex | Thread `b` checks that `flag` is `FALSE` and moves to the `else` branch                              |
| flag = false, mutex = <<"b", 1>>, mutex2 = NULL, mutex3 = NULL            | TryEnterMutex    | Else_SetFlag    | After acquiring `mutex`, thread `b` sets `flag` to `FALSE`                                           |
| ...                                                                       | CheckEnterMutex  | Else_SetFlag    | Thread `a` resumes execution and checks if `flag` is `TRUE`                                          |
| flag = false, mutex = <<"b", 1>>, mutex2 = <<"a", 1>>, mutex3 = NULL      | Else_EnterMutex2 | Else_SetFlag    | Thread `a` finds out that `flag` is `FALSE` and moves to the `else` branch                           |
| ...                                                                       | SetFlag          | Else_SetFlag    | Thread `a` `will` set `flag` to `TRUE`                                                               |
| ...                                                                       | SetFlag          | Else_ExitMutex  | Thread `b` resumes execution and releases `mutex` before thread `a` sets `flag` to `TRUE`            |
| flag = true, mutex = <<"b", 1>>, mutex2 = NULL, mutex3 = NULL             | ExitMutex2       | Else_ExitMutex  | Thread `a` sets `flag` to `TRUE`                                                                     |
| flag = true, mutex = NULL, mutex2 = NULL, mutex3 = NULL                   | Loop             | Else_ExitMutex  | Thread `a` releases `mutex2`                                                                         |
| ...                                                                       | TryEnterMutex    | Else_ExitMutex  | Thread `a` tries to acquire `mutex`                                                                  |
| ...                                                                       | TryEnterMutex    | Loop            | Thread `b` resumes execution                                                                         |
| ...                                                                       | CheckEnterMutex  | Loop            | Thread `a` checks if `mutex` has been acquired                                                       |
| flag = true, mutex = <<"a", 1>>, mutex2 = NULL, mutex3 = <<"a", 1>>       | EnterMutex3      | Loop            | `mutex3` was already acquired by thread `a`                                                          |
| flag = true, mutex = <<"a", 2>>, mutex2 = NULL, mutex3 = <<"a", 1>>       | EnterMutex       | Loop            | Thread `a` acquires `mutex` again                                                                    |
| flag = true, mutex = <<"a", 1>>, mutex2 = NULL, mutex3 = <<"a", 1>>       | ExitMutex        | Loop            | Thread `a` releases `mutex`                                                                          |
| ...                                                                       | EnterMutex2      | Loop            | Thread `a` will try to acquire `mutex2`                                                              |
| ...                                                                       | EnterMutex2      | CheckFlag       | Thread `b` resumes execution and checks if `flag` is `TRUE`                                          |
| flag = true, mutex = <<"a", 1>>, mutex2 = <<"b", 1>>, mutex3 = <<"a", 1>> | EnterMutex2      | EnterMutex2     | `flag` is `TRUE`, so thread `b` triers to acquire `mutex2`                                           |
| flag = true, mutex = <<"a", 1>>, mutex2 = <<"b", 1>>, mutex3 = <<"a", 1>> | EnterMutex2      | EnterMutex      | Thread `b` acquires `mutex`2 and tries to acquire `mutex` while thread `a` tries to acquire `mutex2. |
| ...                                                                       | Deadlock         | Deadlock        |                                                                                                      |

[Manual Reset Event](https://deadlockempire.github.io/#H1-ManualResetEvent)

```c#
// Thread a
while (true) {
  sync.Wait();
  if (counter % 2 == 1) {
    Debug.Assert(false);
  }
}

// Thread b
while (true) {
  sync.Reset();
  counter++;
  counter++;
  sync.Set();
}
```

```pluscal
---- MODULE spec ----
EXTENDS TLC, Integers

(*--algorithm spec
variables
    signal = FALSE,
    counter = 0;

process a = "a"
variables
    tmp = 0;
begin
Loop:
while TRUE do
    WaitSignal: await signal;
    LoadCounter: tmp := counter;
    CheckCounter:
    if tmp % 2 = 1 then
        assert FALSE;
    end if;
end while;
end process;

process b = "b"
variables
    tmp = 0;
begin
Loop:
while TRUE do
    ResetSignal: signal := FALSE;

    LoadCounter1: tmp := counter;
    IncCounter1: counter := tmp + 1;

    LoadCounter2: tmp := counter;
    IncCounter2: counter := tmp + 1;

    SetSignal: signal := TRUE;
end while;
end process;
end algorithm; *)
====
```

| State                                             | Thread a     | Thread b     | Description                                                                |
| ------------------------------------------------- | ------------ | ------------ | -------------------------------------------------------------------------- |
| signal = false, counter = 0                       | Loop         | Loop         | Both threads start running                                                 |
| signal = false, counter = 0                       | WaitSignal   | Loop         | Thread `a` blocks waiting for the signal                                   |
| signal = false, counter = 0                       | WaitSignal   | Loop         | Thread `b` resets the signal, it does not unblock threads that are waiting |
| signal = false, counter = 0, b.tmp = 0            | WaitSignal   | LoadCounter1 | Thread `b` loads `counter`                                                 |
| signal = false, counter = 0, b.tmp = 0            | WaitSignal   | IncCounter1  | Thread `b` increments `counter` by setting it to `tmp + 1`                 |
| signal = false, counter = 1, b.tmp = 0            | WaitSignal   | LoadCounter2 | Thread `b` loads `counter` again                                           |
| signal = false, counter = 1, b.tmp = 1            | WaitSignal   | IncCounter2  | Thread `b` increments `counter` by setting it to `tmp + 1`                 |
| signal = false, counter = 2, b.tmp = 1            | WaitSignal   | SetSignal    | Thread `b` signals the waiting thread                                      |
| signal = true, counter = 2, b.tmp = 1             | WaitSignal   | Loop         | Thread `b` goes back to the beginning of the loop                          |
| signal = true, counter = 2, a.tmp = 0, b.tmp = 1  | LoadCounter  | Loop         | Thread `a` loads `counter`                                                 |
| signal = true, counter = 2, a.tmp = 0, b.tmp = 1  | LoadCounter  | ResetSignal  | Thread `b` resets the signal                                               |
| signal = false, counter = 2, a.tmp = 0, b.tmp = 1 | LoadCounter  | LoadCounter1 | Thread `b` loads `counter`                                                 |
| signal = false, counter = 2, a.tmp = 0, b.tmp = 2 | LoadCounter  | IncCounter1  | Thread `b` increments `counter` by setting it to `tmp + 1`                 |
| signal = false, counter = 3, a.tmp = 0, b.tmp = 2 | LoadCounter  | LoadCounter2 | Thread `b` loads `counter`                                                 |
| signal = false, counter = 3, a.tmp = 3, b.tmp = 2 | CheckCounter | LoadCounter2 | Thread `a` resumes and checks if `counter` is odd and finds that it is     |

[Countdown Event](https://deadlockempire.github.io/#H2-CountdownEvent)

```c#
// Thread a
progress = progress + 20;
if (progress >= 20) {
  event.Signal();
event.Signal();
Atomic. Decrements the CountdownEvent's countdown timer by one. Throws an exception if the timer is already at zero (and you win the level).
}
event.Wait();

// Thread b
progress = progress + 30;
if (progress >= 30) {
  event.Signal();
}
progress = progress + 50;
if (progress >= 80) {
  event.Signal();
}
event.Wait();
```

```tlaplus

---- MODULE spec ----
EXTENDS TLC, Integers

(*--algorithm spec
variables
    signal = 3,
    progress = 0;

macro signal_signal() begin
    assert signal > 0;
    signal := signal - 1;
end macro;

macro signal_wait() begin
    await signal = 0;
end macro;

process a = "a"
variables
    tmp = 0;
begin
LoadProgres1: tmp := progress;
SetProgress: progress := tmp + 20;

LoadProgress2: tmp := progress;

CheckProgress: if tmp >= 20 then
    signal_signal();
end if;

WaitSignal: signal_wait();
end process;

process b = "b"
variables
    tmp = 0;
begin
LoadProgress1: tmp := progress;
SetProgress1: progress := tmp + 30;

LoadProgress2: tmp := progress;

CheckProgress1: if tmp >= 30 then
    signal_signal();
end if;

LoadProgress3: tmp := progress;
SetProgress2: progress := tmp + 50;

LoadProgress4: tmp := progress;

CheckProgress2: if tmp >= 80 then
    signal_signal();
end if;

signal_wait();
end process;
end algorithm; *)
====
```

| State                                             | Thread a      | Thread b       | Description                                                                                    |
| ------------------------------------------------- | ------------- | -------------- | ---------------------------------------------------------------------------------------------- |
| signal = 3, progress = 0, a.tmp = 0               | LoadProgress1 | LoadProgress1  | Both threads start running                                                                     |
| signal = 3, progress = 0, a.tmp = 0               | SetProgress   | LoadProgress1  | Thread `a` is waiting to set `progress`to `tmp + 20`                                           |
| signal = 3, progress = 0, a.tmp = 0, b.tmp = 0    | SetProgress   | SetProgress1   | Thread `b` will set `progress`to `tmp + 30`                                                    |
| signal = 3, progress = 30, a.tmp = 0, b.tmp = 0   | SetProgress   | LoadProgress2  | Thread `b` will load `progress` again                                                          |
| signal = 3, progress = 20, a.tmp = 0, b.tmp = 0   | LoadProgress2 | LoadProgress2  | Thread `a` resumes execution and sets `progress` to `tmp + 20` before loading `progress` again |
| signal = 3, progress = 20, a.tmp = 20, b.tmp = 0  | CheckProgress | LoadProgress2  | Thread `a` will check that `progress >= 20`                                                    |
| signal = 2, progress = 20, a.tmp = 20, b.tmp = 0  | WaitSignal    | LoadProgress2  | Thread `a` will wait for signal to reache `0`                                                  |
| signal = 2, progress = 20, a.tmp = 20, b.tmp = 20 | WaitSignal    | CheckProgress1 | Thread `b` will check that `progress >= 30` after loading it into `tmp`                        |
| signal = 2, progress = 20, a.tmp = 20, b.tmp = 20 | WaitSignal    | LoadProgress3  | Thread `b` will load `progress` into `tmp`                                                     |
| signal = 2, progress = 20, a.tmp = 20, b.tmp = 20 | WaitSignal    | SetProgress2   | Thread `b` will set `progress` to `tmp + 50`                                                   |
| signal = 2, progress = 70, a.tmp = 20, b.tmp = 20 | WaitSignal    | LoadProgress 4 | Thread `b` will load `progress`                                                                |
| signal = 2, progress = 70, a.tmp = 20, b.tmp = 20 | WaitSignal    | LoadProgress 4 | Thread `b` will check that `progress >= 80`                                                    |
| signal = 2, progress = 70, a.tmp = 20, b.tmp = 20 | DeadLock      | DeadLock       | Thread `a` is waiting for `signal` to reach `0` and thread `b` has already completed execution |

[Countdown Event Revisited](https://deadlockempire.github.io/#H3-CountdownEvent)

In this case, since two threads are updating `progress` without synchronizing, lost updates cause `event.Signal()` to be called more than the allowed number of times (3).

```c#
// Thread a
while (true) {
  progress = progress + 20;
  event.Signal();
  event.Wait();
  if (progress == 100) {
    Environment.Exit(0);
  }
}

// Thread b
while (true) {
  progress = progress + 30;
  event.Signal();
  progress = progress + 50;
  event.Signal();
  event.Wait();
  if (progress == 100) {
    Environment.Exit(0);
  }
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Integers

(*--algorithm spec
variables
    signal = 3,
    progress = 0;

define
    SignalNeverGoesBelowZero == signal >= 0
end define;

macro signal_signal() begin
    signal := signal - 1;
end macro;

macro signal_wait() begin
    await signal = 0;
end macro;

process a = "a"
variables
    exit = FALSE,
    tmp = 0;
begin
Loop:
while ~exit do
LoadProgres1: tmp := progress;
SetProgress: progress := tmp + 20;

Signal: signal_signal();

WaitSignal: signal_wait();

LoadProgress2: tmp := progress;
CheckProgress: if tmp = 100 then
    exit := TRUE;
end if;
end while;
end process;

process b = "b"
variables
    exit = FALSE,
    tmp = 0;
begin
Loop:
while ~exit do
LoadProgress1: tmp := progress;
SetProgress1: progress := tmp + 30;

Signal1: signal_signal();

LoadProgress2: tmp := progress;
SetProgress2: progress := tmp + 50;

Signal2: signal_signal();

WaitSignal: signal_wait();

LoadProgress3: tmp := progress;
CheckProgress1: if tmp = 100 then
    exit := TRUE;
end if;
end while;
end process;
end algorithm; *)
====
```

[The Barrier](https://deadlockempire.github.io/#H3-CountdownEvent)

In this case, `fireball_charge` will be `0` when thread `a` executes the if statement depending on the order of calls to `barrier.SignalAndWait`.

```c#
// Thread a
int fireballCharge=0;
System.Threading.Barrier barrier; // [phase 0, waiting for 2 threads]

while (true) {
  Interlocked.Increment(ref fireballCharge);
  barrier.SignalAndWait();
  if (fireballCharge < 2) {
    Debug.Assert(false);
  }
  fireball();
}

// Thread b
while (true) {
  Interlocked.Increment(ref fireballCharge);
  barrier.SignalAndWait();
}

// Thread c
while (true) {
  Interlocked.Increment(ref fireballCharge);
  barrier.SignalAndWait();
  barrier.SignalAndWait();
  fireballCharge = 0;
}
```

```pluscal
---- MODULE spec ----
EXTENDS TLC, Integers, Sequences

(*--algorithm spec
variables
    fireball_charge = 2,
    barrier = 2,
    barrier_blocked = {};

procedure barrier_signal_and_wait(thread) begin
    BarrierSignal:
        if barrier - 1 = 0 then
            \* Unblock threads waiting for the barrier.
            barrier_blocked := {};
            \* Reset the barrier.
            barrier := 2;
        else
            barrier := barrier - 1;
            barrier_blocked := barrier_blocked \union {thread};
        end if;

    BarrierAwait:
        await thread \notin barrier_blocked;

    return;
end procedure;


process a = "a"
begin
A_Loop:
while TRUE do
    A_IncrementFireball: fireball_charge := fireball_charge + 1;
    A_BarrierSignalAndWait: call barrier_signal_and_wait("a");
    A_CheckFireball: if fireball_charge < 2 then
        print("CheckFireball: fireball_charge < 2");
        assert FALSE;
    end if;
end while;
end process;

process b = "b"
begin
B_Loop:
while TRUE do
    B_IncrementFireball: fireball_charge := fireball_charge + 1;
    B_BarrierSignalAndWait: call barrier_signal_and_wait("b");
end while;
end process;

process c = "c"
begin
C_Loop:
while TRUE do
    C_IncrementFireball: fireball_charge := fireball_charge + 1;
    C_BarrierSignalAndWait1: call barrier_signal_and_wait("c");
    C_BarrierSignalAndWait2: call barrier_signal_and_wait("c");
    C_ResetFireball: fireball_charge := 0;
end while;
end process;
end algorithm; *)
====
```

| Action |
| ------ |

A increments fireball_charge
B increments fireball_charge
C increments fireball_charge
A signals and blocks
C signals and blocks, unblocking A and C
B signals and blocks
C signals and blocks, unblocking B and C
C resets fireball_charge to 0.
A checks fireball_charge, fireball_charge is 0.

[Semaphores](https://deadlockempire.github.io/#S1-simple)

```c#
// Thread a
while (true) {
  semaphore.Wait();
  critical_section();
  semaphore.Release();
}

// Thread b
while (true) {
  if (semaphore.Wait(500)) {
    critical_section();
    semaphore.Release();
  } else {
    semaphore.Release();
  }
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Integers

(*--algorithm spec
variables
    sema = 1;
    threads_in_critical_section = 0;

define
    CriticalSection == threads_in_critical_section <= 1
end define;

macro semaphore_wait(block) begin
    if block then
        await sema = 1;
        sema := 0;
        sema_acquired := TRUE;
    elsif sema = 0 then
        sema_acquired := TRUE;
    end if;
end macro;

macro semaphore_release() begin
    skip
end macro;

process a = "a"
variables
    sema_acquired = FALSE;
begin
Loop:
while TRUE do
    ResetSemaAcquired: sema_acquired := FALSE;
    SemaphoreWait: semaphore_wait(TRUE);
    CriticalSection_1: threads_in_critical_section := threads_in_critical_section + 1;
    CriticalSection_2: threads_in_critical_section := threads_in_critical_section - 1;
    SemaphoreRelease: semaphore_release();
end while;
end process;

process b = "b"
variables
    sema_acquired = FALSE;
begin
Loop:
while TRUE do
    ResetSemaAcquired: sema_acquired := FALSE;
    SemaphoreWait: semaphore_wait(FALSE);
    if sema_acquired then
        SemaphoreRelease_1: semaphore_release();
        CriticalSection_1: threads_in_critical_section := threads_in_critical_section + 1;
        CriticalSection_2: threads_in_critical_section := threads_in_critical_section - 1;
    else
        SemaphoreRelease_2: semaphore_release();
    end if;
end while;
end process;
end algorithm; *)
====
```

| Action |
| ------ |

Thread `a` waits to acquire the semaphore.
Thread `b` tries to acquire the semaphore with a `500ms` timeout, fails and releases the semaphore in the `else` branch.
Thread `a` acquires the semaphore and enters the critical section.
Thread `b` tries to acquire the semaphore with a `500ms` timeout, fails and releases the semaphore in the `else` branch again.
Thread `b` tries to acquire the semaphore with a `500ms` timeout, succeds and enters the critical section.
Both threads are in the critical section at the same time.

[Producer-consumer](https://deadlockempire.github.io/#S2-producerConsumer)

```c#
// Thread a
while (true) {
  if (semaphore.Wait(500)) {
    queue.Dequeue();
  } else {
    // Nothing in the queue.
  }
}

// Thread b
while (true) {
  semaphore.Release();
  queue.Enqueue(new Dragon());
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Sequences, Integers

(*--algorithm spec
variables
    sema = 0,
    queue = <<>>;

macro semaphore_wait(block) begin
    if block then
        await sema = 1;
        sema := 0;
        sema_acquired := TRUE;
    elsif sema = 1 then
        sema := 0;
        sema_acquired := TRUE;
    end if;
end macro;

macro semaphore_release() begin
    sema := 1;
end macro;

macro dequeue() begin
    assert Len(queue) > 0;
    queue := Tail(queue);
end macro;

macro enqueue() begin
    queue := Append(queue, "v");
end macro;

process a = "a"
variables
    sema_acquired = FALSE;
begin
Loop:
while TRUE do
    ResetSemaAcquired: sema_acquired := FALSE;
    SemaphoreWait:
        semaphore_wait(FALSE);
        if sema_acquired then
            Dequeue: dequeue();
        end if;

end while;
end process;

process b = "b"
begin
Loop:
while TRUE do
    ReleaseSema: semaphore_release();
    Enqueue: enqueue();
end while;
end process;
end algorithm; *)
====
```

| Action |
| ------ |
Thread `a` tries to acquire the semaphore with a `500ms` timeout, fails and goes back to the start of the loop.
Thread `b` releases the semaphore.
Thread `a` acquires the semaphore before thread `b` adds an item to the queue.
Thread `a` tries to dequeue from an empty queue.

[Producer-Consumer (variant)](https://deadlockempire.github.io/#S3-producerConsumer)

```c#
// Thread a
while (true) {
  queue.Enqueue(new Golem());
}

// Thread b
while (true) {
  if (queue.Count > 0) {
    queue.Dequeue();
  }
}
```

```tlaplus

---- MODULE spec ----
EXTENDS TLC, Sequences, Integers

(*--algorithm spec
variables
    queue = <<>>,
    is_queue_inconsistent = FALSE;

procedure enqueue() begin 
    AddItem: queue := Append(queue, "v");
    EnterInconsistentState: is_queue_inconsistent := TRUE;
    LeaveInconsistentState: is_queue_inconsistent := FALSE;
end procedure;

procedure dequeue() begin 
Dequeue:
    assert is_queue_inconsistent = FALSE;
    queue := Tail(queue);
end procedure;

process a = "a"
begin
Loop:
while TRUE do
    call enqueue();
end while;
end process;

process b = "b"
begin
Loop:
while TRUE do
    CheckQueueLen: 
    if Len(queue) > 0 then
        call dequeue();
    end if;
end while;
end process;
end algorithm; *)
====
```

| Action |
| ------ |
Thread `a` starts the operation to add an item to queue and the queue enters an incosistent state while being modified
Thread `b` finds out that the queue is not empty and tries to dequeue an item while the queue is still being modified by thread `a`

[Condition Variables](https://deadlockempire.github.io/#CV1-simple)

```c#
// Thread a
while (true) {
  Monitor.Enter(mutex);
  if (queue.Count == 0) {
    Monitor.Wait(mutex);
      release the lock, then sleep
      wait until woken up
      Monitor.Enter(mutex);
  }
  queue.Dequeue();
  Monitor.Exit(mutex);
}

// Thread b
while (true) {
  Monitor.Enter(mutex);
  if (queue.Count == 0) {
    Monitor.Wait(mutex);
  }
  queue.Dequeue();
  Monitor.Exit(mutex);
}

// Thread c
while (true) {
  Monitor.Enter(mutex);
  queue.Enqueue(42);
  Monitor.PulseAll(mutex);
  Monitor.Exit(mutex);
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Sequences, Integers

(*--algorithm spec
variables
    queue = <<>>,
    condition_variable = [a |-> FALSE, b |-> FALSE],
    mutex = "";

macro mutex_enter(thread) begin
    await mutex = "";
    mutex := thread;
end macro;

macro mutex_exit(thread) begin
    assert mutex = thread;
    mutex := "";
end macro;

macro mutex_pulse_all(thread) begin
    assert mutex = thread;
    condition_variable := [x \in DOMAIN condition_variable |-> TRUE];
end macro;

macro dequeue() begin
    assert Len(queue) > 0;
    queue := Tail(queue);
end macro;

procedure mutex_wait(thread) begin
    ReleaseMutex:
        assert mutex = thread;
        mutex := "";
    AwaitForConditionVariable: 
        await condition_variable[thread] = TRUE;
        condition_variable[thread] := FALSE;
    AcquireMutex: 
        mutex_enter(thread);
        return;
end procedure;

process a = "a"
begin
Loop:
while TRUE do
    AcquireMutex: mutex_enter("a");
    CheckQueueLen: 
        if Len(queue) = 0 then
            call mutex_wait("a");
        end if;
    Dequeue: dequeue();
    ReleaseMutex: mutex_exit("a");
end while;
end process;

process b = "b"
begin
Loop:
while TRUE do
    AcquireMutex: mutex_enter("b");
    CheckQueueLen: 
        if Len(queue) = 0 then
            call mutex_wait("b");
        end if;
    Dequeue: dequeue();
    ReleaseMutex: mutex_exit("b");
end while;
end process;

process c = "c"
begin
Loop:
while TRUE do
    AcquireMutex: mutex_enter("c");
    Enqueue: queue := Append(queue, 42);
    MutexPulseAll: mutex_pulse_all("c");
    ReleaseMutex: mutex_exit("c");
end while;
end process;
end algorithm; *)
====
```

| Action |
| ------ |
Thread `a` acquires the mutex first, sees that the queue is empty and waits for the condition variable signal before proceeding.
Thread `c` acquires the mutex, adds an item to the queue, signals the condition variable and releases the mutex.
Thread `b` acquires the mutex before thread `a` gets to run, dequeues an item from the queue and releases the mutex.
Thread `a` wakes up with the mutex acquired and tries to dequeue an item but finds out that queue is empty.

[Dragonfire](https://deadlockempire.github.io/#D1-Dragonfire)

```c#
// Thread a
while (true) {
  Monitor.Enter(firebreathing);
  incinerate_enemies();
  if (fireball.Wait(500)) {
    // Swoosh!
    blast_enemies();
    // Uh... that was tiring.
    // I'd better rest while I'm vulnerable...
    if (fireball.Wait(500)) {
      if (fireball.Wait(500)) {
        critical_section();
      }
    }
    // Safe now...
  }
  c = c - 1;
  c = c + 1;
  Monitor.Exit(firebreathing);
}

// Thread b
// This is stupid.
// The other head gets all the cool toys,
// ...and I get stuck recharging.
while (true) {
  if (c < 2) {
    // Let's do some damage!
    fireball.Release();
    c++;
  } else {
    // I hate being in here.
    critical_section();
  }
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Integers

(*--algorithm spec
variables
    mutex = "",
    critical_section = 0,
    c = 0,
    fireballs = 0;

define
    CriticalSection == critical_section <= 1
end define;

macro mutex_enter(thread) begin
    await mutex = "";
    mutex := thread;
end macro;

macro mutex_exit(thread) begin
    assert mutex = thread;
    mutex := "";
end macro;

macro fireball_wait() begin
    if fireballs > 0 then
        fireballs := fireballs - 1;
        ok := TRUE;
    else
        ok := FALSE;
    end if;
end macro;

process a = "a"
variables
    tmp = 0,
    ok = FALSE;
begin
Loop:
while TRUE do
    AcquireMutex: mutex_enter("a");
    \* incinerate_enemies();
    CheckFireball_1:
    fireball_wait();
    if ok then
        \* blast_enemies();
        CheckFireball_2:
        fireball_wait();
        if ok then
            CheckFireball_3:
            fireball_wait();
            if ok then
                EnterCriticalSection: critical_section := critical_section + 1;
                LeaveCriticalSection: critical_section := critical_section - 1;
            end if;
        end if;
    end if;

    LoadC_1:  tmp := c;
    DecrementC: c := tmp - 1;

    LoadC_2: tmp := c;
    IncrementC: c := tmp + 1;

    ReleaseMutex: mutex_exit("a");
end while;
end process;

process b = "b"
variables
    tmp = 0;
begin
Loop:
while TRUE do
    if c < 2 then
        FireballRelease: fireballs := fireballs + 1;
        LoadC: tmp := c;
        IncrementC: c := tmp + 1;
    else
        EnterCriticalSection: critical_section := critical_section + 1;
        LeaveCriticalSection: critical_section := critical_section - 1;
    end if;
end while;
end process;
end algorithm; *)
====
```

[Triple danger](https://deadlockempire.github.io/#D2-Sorcerer)

```c#
// Thread a
while (true) {
  Monitor.Enter(conduit);
  // I summon mana for you, dragon!
  // Incinerate the enemies!
  energyBursts.Enqueue(new EnergyBurst());
  Monitor.Exit(conduit);
}

// Thread b
while (true) {
  if (energyBursts.Count > 0) {
    Monitor.Enter(conduit);
    energyBursts.Dequeue();
    lightning_bolts(terrifying: true);
    Monitor.Exit(conduit);
  }
}

// Thread c
while (true) {
  if (energyBursts.Count > 0) {
    Monitor.Enter(conduit);
    energyBursts.Dequeue();
    fireball(mighty: true);
    Monitor.Exit(conduit);
  }
}
```

Provided without comment.

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Sequences, Integers

(*--algorithm spec
variables
    mutex = "",
    queue = <<>>;

macro mutex_enter(thread) begin
    await mutex = "";
    mutex := thread;
end macro;

macro mutex_exit(thread) begin
    assert mutex = thread;
    mutex := "";
end macro;

macro enqueue() begin
    queue := Append(queue, "v");
end macro;

macro dequeue() begin
    assert Len(queue) > 0;
    queue := Tail(queue);
end macro;

process a = "a"
begin
Loop:
while TRUE do
    AcquireMutex: mutex_enter("a");
    Enqueue: enqueue();
    ReleaseMutex: mutex_exit("a");
end while;
end process;

process b = "b"
begin
Loop:
while TRUE do
    if Len(queue) > 0 then
        AcquireMutex: mutex_enter("b");
        Dequeue: dequeue();
        ReleaseMutex: mutex_exit("b");
    end if;
end while;
end process;

process c = "c"
begin
Loop:
while TRUE do
    if Len(queue) > 0 then
        AcquireMutex: mutex_enter("c");
        Dequeue: dequeue();
        ReleaseMutex: mutex_exit("c");
    end if;
end while;
end process;
end algorithm; *)
====
```

Provided without comment.

[Boss fight](https://deadlockempire.github.io/#D4-Boss)

```c#
// Thread a
while (true) {
  darkness++;
  evil++;
  if (darkness != 2 && evil != 2) {
    if (fortress.Wait(500)) {
      fortress.Wait();
      Monitor.Enter(sanctum);
      Monitor.Wait(sanctum);
      critical_section();
      Monitor.Exit(sanctum);
    }
  }
}

// Thread b
while (true) {
  darkness++;
  evil++;
  if (darkness != 2 && evil == 2) {
    Monitor.Enter(sanctum);
    Monitor.Pulse(sanctum);
    Monitor.Exit(sanctum);
    critical_section();
  }
  fortress.Release();
  darkness = 0;
  evil = 0;
}
```

```tlaplus
---- MODULE spec ----
EXTENDS TLC, Integers, Sequences

(*--algorithm spec
variables 
    mutex = "",
    mutex_pulse_received = FALSE,
    darkness = 0,
    evil = 0,
    fortress = 0,
    threads_in_critical_section = 0;

define
    MutualExclusion == threads_in_critical_section <= 1
end define;

macro mutex_enter(thread) begin
    await mutex = "";
    mutex := thread;
end macro;

macro mutex_exit(thread) begin
    assert mutex = thread;
    mutex := "";
end macro;

macro mutex_pulse(thread) begin
    assert mutex = thread;
    mutex_pulse_received := TRUE;
end macro;

macro fortress_wait(block) begin
    if block then
        await fortress > 0;
    end if;

    if fortress = 0 then
        ok := FALSE;
    else
        fortress := fortress - 1;
        ok := TRUE;
    end if;
end macro;

procedure mutex_wait(thread) begin
    MutexWait_ReleaseMutex:
        assert mutex = thread;
        mutex := "";
    MutexWait_WaitForPulse:
        await mutex_pulse_received = TRUE;
        mutex_pulse_received := FALSE;
    MutexWait_AcquireMutex:
        mutex_enter(thread);
    return;
end procedure;

procedure inc_darkness() 
    variables
        tmp = 0;
begin 
    Inc_Load: tmp := darkness;
    Inc_Add: darkness := tmp + 1;
    return;
end procedure;

procedure inc_evil() 
    variables
        tmp = 0;
begin 
    Inc_Load: tmp := evil;
    Inc_Add: evil := tmp + 1;
    return;
end procedure;

procedure critical_section() begin 
    CriticalSection_Enter: threads_in_critical_section := threads_in_critical_section + 1;
    CriticalSection_Leave: threads_in_critical_section := threads_in_critical_section - 1;
    return;
end procedure;

process a = "a"
variables 
    ok = FALSE;
begin
Loop:
while TRUE do
    IncDarkness: call inc_darkness();
    IncEvil: call inc_evil();
    Check:
    if darkness # 2 /\ evil # 2 then
        FortressWait_1: fortress_wait(FALSE);
        if ok then 
            FortressWait_2: fortress_wait(TRUE);
            DecFortress: fortress :=  fortress - 1;
            AcquireMutex: mutex_enter("a");
            MutexWait: call mutex_wait("a");
            CriticalSection: call critical_section();
            ReleaseMutex: mutex_exit("a");
        end if;
    end if;
end while;
end process;

process b = "b"
begin
Loop:
while TRUE do
    IncDarkness: call inc_darkness();
    IncEvil: call inc_evil();
    Check:
    if darkness # 2 /\ evil = 2 then
        AcquireMutex: mutex_enter("b");
        MutexPulse: mutex_pulse("b");
        MutexExit: mutex_exit("b");
        CriticalSection: call critical_section();
    end if;
    FortressRelease: fortress := fortress + 1;
    ResetDarkness: darkness := 0;
    ResetEvil: evil := 0;
end while;
end process;
end algorithm; *)
====
```

Provided without comment.