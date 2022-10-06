---
title: "Concurrency, Parallelism, Cooperative scheduling and coroutines"
date: 2022-05-23T20:39:07-03:00
categories: ["algorithms", "os", "concurrency", "languages", "parallelism"]
draft: true
---

# Web services

Web services spend a lot of time waiting on [I/O] to the point where the main bottleneck is not in the operations performed by the application itself but in the time spent waiting on a I/O operation to finish.

## A simple server

A simple single thread server that's using HTTP as its protocol can just wait for a new request to arrive and handle the request immediately, effectively blocking any other request that arrived while the current request is being processed[^distributedsystems].

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/194180918-5ceab327-7826-4c44-b44e-4f4d10a81087.png" />
</p>
<p align="center">
  <i>Simple single-threaded server that receives a request and creates a response immediately.</i>
</p>

## Multiple workers, single accept queue

Instead of having one process that handles one request at a time, the work can be divided between several workers that consume from the same accept queue.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/194181358-2bb85ebf-308c-49d0-98b1-e5575033a0c6.png" />
</p>
<p align="center">
  <i>Multiple workers consuming the same accept queue.</i>
</p>

## Multiple workers, multiple accept queue

Instead of having one worker that handles one request at a time, the work can be divided between several workers that consume from distinct accept queues by using the [SO_REUSEPORT] socket option.

The SO_REUPORT option distributes connections evenly across workers that are listening to the same port[^so_reuseport]. Having multiple accept queues are lower contention.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/194182191-bf25e09f-39d9-49dc-a2ee-f9196a19b2a1.png" />
</p>
<p align="center">
  <i>Multiple workers listening to the same port.</i>
</p>

## Blocking I/O

Consider reading a file using the [read] syscall:

```c
read(fd, &buffer, 4096)
```

The thread will be put to sleep until the file is ready to be read which can be a waste of resources since the thread won't be able to do anything in the meantime.

## Non-blocking I/O

It can be desirable for a thread to continue executing while an I/O operation is not complete yet. The thread can request an I/O operation and the operating system will notify the thread when the operation is ready, in the meantime, the thread will proceed executing code, possibly code that's unrelated to the I/O operation that's being performed[^epoll].

A single thread with a task queue can execute several tasks concurrently increasing the overall amount of progress done at each time unit as long as the tasks perform non-blocking operations. The thread can also be brought to a halt if one or more tasks perform blocking operations, effectively blocking the whole runtime[^nodejs_event_loop].

Given a simple task queue, the thread starts executing tasks in [FIFO] order

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/194185701-8662c64c-ca19-4df2-80e3-04f022ac8b7a.png" />
</p>
<p align="center">
  <i>Single thread multiplexing several tasks.</i>
</p>

When a task needs to perform async I/O, [epoll] can be used, and the runtime that's managing the tasks(the main thread in this case) inserts the task in the back of the pending queue. The task will stay there until the async operation associated with it is complete.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/194186820-09df75a1-6f98-44f9-ac02-ee786bc0b31e.png" />
</p>
<p align="center">
  <i>Thread puts the blocked task in the pending queue.</i>
</p>

While the task that started an async operation is waiting for the operation to progress, the runtime starts executing another threads right away, given that there are tasks ready to be executed.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/194185846-48b5d333-0f89-403d-8965-4316ee3a48c4.png" />
</p>
<p align="center">
  <i>Thead starts executing another task.</i>
</p>

When the async operation started by the first task is ready, the operating systems notifies the runtime and then the task that's associated with the operation is moved from the pending queue to the ready queue because it can make progress again. Note that the thread that's executing tasks would just go to sleep if there were no tasks ready to be executed while the first task is pending instead of busy waiting and wasting [CPU cycle]s.

Since there's only one thread, tasks will be moved from the pending queue to the ready queue only when the current task yields control back to the runtime by performing an operation that puts it to sleep.

## Cooperative scheduling

The process just describe is also known as cooperative scheduling[^cooperative_scheduling] where a scheduler decides which task gets to run next after the currently running task cooperates by yielding control back to the scheduler after pre defined events. In contrast to cooperative scheduling, there is preemptive scheduling where a task is interrupted by the scheduler without the need for cooperation[^preemptive_scheduling]. Preemptive schedulers are the type of scheduler that will be found in most operating systems.

The main problem with cooperative scheduling is that a rogue task can block a thread or even the whole runtime depending on how many threads are in use.

# TODO: compare how languages do concurrency/parallelism

[i/o]: https://en.wikipedia.org/wiki/Input/output

[^distributedsystems]: https://www.distributed-systems.net/index.php/books/ds3/

[so_reuseport]: https://lwn.net/Articles/542629/

[^so_reuseport]: https://lwn.net/Articles/542629/

[read]: https://man7.org/linux/man-pages/man2/read.2.html

[^epoll]: https://man7.org/linux/man-pages/man7/epoll.7.html

[epoll]: https://man7.org/linux/man-pages/man7/epoll.7.html

[^nodejs_event_loop]: https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/

[fifo]: https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)
[cpu cycle]: https://en.wikipedia.org/wiki/Instruction_cycle

[^cooperative_scheduling]: https://en.wikipedia.org/wiki/Cooperative_multitasking
[^preemptive_scheduling]: https://en.wikipedia.org/wiki/Preemption_(computing)

[cloudflare_the_sad_state_of_linux_socket_balancing]: https://blog.cloudflare.com/the-sad-state-of-linux-socket-balancing/
