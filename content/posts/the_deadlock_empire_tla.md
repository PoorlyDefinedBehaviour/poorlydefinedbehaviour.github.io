---
title: "The deadlock empire: TLA+ solutions"
date: 2024-07-11T00:00:00-00:00
categories: ["formal methods", "distributed systems"]
draft: true
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
    entered_critical_section = {};

define
    OnlyOneThreadEntersCriticalSection == Cardinality(entered_critical_section) <= 1
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
        entered_critical_section := entered_critical_section \union {self};
    end if; 
end process;

end algorithm; *)
====
```

| State | Thread 1 | Thread 2 | Description |
--- | --- | --- | --- |
| a = 0 | Init | Init | Both threads are at the initial state |
| a = 0, thread1.tmp = 0 | Load | Init | Thread 1 stores the value of `a` in a temporary variable |
| a = 0, thread1.tmp = 0, thread2.tmp = 0 | Load | Load | Thread 2 stores the value of `a` in a temporary variable |
| a = thread1.tmp + 1, thread1.tmp = 0, thread2.tmp = 0 | Store | Load | Thread 1 sets `a` to `0 + 1` using the temporary variable |
| a = thread2.tmp + 1, thread1.tmp = 0, thread2.tmp = 0 | Store | Store | Thread 2 sets `a` to `0 + 1` using the temporary variable |
| a = 1 | CriticalSectionCheck | Store | Thread 1 enters the critical section since `a` is equal to `1` |
| a = 1 | CriticalSectionCheck | CriticalSectionCheck | Thread 2 enters the critical section since `a` is equal to `1` at the same time as thread 1 | 

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

| State | Thread 1    | Thread 2 | Description |
| ------| ----------- | ---------| ------------|
| flag = false | Init | Init | Both threads are at the initial state |
| flag = false | SpinLock | Init | Thread 1 enters the spinlock |
| flag = false | SpinLock | SpinLock | Thread 2 enters the spinlock |
| flag = true | SpinLock | SetFlag | Thread 2 sees that `flag` is `false` so it leaves the spinlock and sets `flag` to `true` |
| flag = true | SetFlag | SetFlag | Thread 1 was in the spinlock and read the value of `flag` before thread 2 set it to `true`. Thread 1 leaves the spinlock and sets `flag` to `true` |
| flag = true | CriticalSection1 | CriticalSection1 | Both threads enter the critical section at the same time |

