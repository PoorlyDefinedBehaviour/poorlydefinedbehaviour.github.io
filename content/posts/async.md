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

When the async operation started by the first task is ready, the operating system notifies the runtime and then the task that's associated with the operation is moved from the pending queue to the ready queue because it can make progress again. Note that the thread that's executing tasks would just go to sleep if there were no tasks ready to be executed while the first task is pending instead of busy waiting and wasting [CPU cycles][cpu cycle].

Since there's only one thread, tasks will be moved from the pending queue to the ready queue only when the current task yields control back to the runtime by performing an operation that puts it to sleep.

## Cooperative scheduling

The process just described is also known as cooperative scheduling[^cooperative_scheduling] where a scheduler decides which task gets to run next after the currently running task cooperates by yielding control back to the scheduler when specific operations are performed, like calling a function `sleep(ms)` for example. In contrast to cooperative scheduling, there is a type of scheduling known as preemptive scheduling where a task is interrupted by the scheduler without the need for cooperation[^preemptive_scheduling]. Preemptive schedulers are the type of scheduler used to schedule processes that will be found in most operating systems.

The main problem with cooperative scheduling is that a rogue task can block a thread or even the whole runtime depending on how many threads the runtime are available for the runtime to use.

## Coroutines

Coroutines allow program execution to be stopped and then resumed at a later time.

## Go and stackful coroutines

Go has [goroutines] which are a cheap lightweight thread abstraction managed by the [Go runtime][^nindalf_how_goroutines_work]. They are usually executed over several threads based on how many cores are available in the system.

On Linux, threads usually have a stack size of 2MB[^pthread_create] and the action of switching between threads is called [context switching][context_switch] which involves transitioning into [kernel mode][kernel] and [syscalls][syscall], then copying several registers[^nindalf_how_goroutines_work], stack pointer and program counter and storying them away so the thread can resume execution at a later time. Turns out that creating threads and switching between them can provide significant overhead.

Goroutines use a technique known as `M:N` scheduling, [green threads] or stackful coroutines where `M` goroutines, which are also known as tasks or lightweight threads, are multiplexed over `N` system threads. The tasks, that need to hold less maintenance state, cooperate with a scheduler that runs in [user mode][protection_ring] which removes the need to switch to kernel-mode whenever a task needs to run.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/194678599-544514e5-1bf9-4f68-b17c-e19c8a333654.png" />
</p>
<p align="center">
  <i>M goroutines being scheduled over N threads.</i>
</p>

Back in the day, Goroutines used a segmented stack. The goroutine started with a small stack and whenever the goroutine needed to put something on the stack and but it was out of space, a new `segment` would be created and that segment would become part of the goroutine's stack[^cloudflare_how_stacks_are_handled_in_go].

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/194678763-e46a29a3-1c42-4500-8332-cf2e0ddf9171.png" />
</p>
<p align="center">
  <i>A segmented stack.</i>
</p>

Go ended up abandoning segmented stacks because it had a problem where given an almost full stack, a call would force a new segment to be allocated and immediately deallocated when the call returned[^docs_contiguous_stacks].

Go abandoned the segmented stacks approach in favor of the same approach used to create growable arrays like Rust's [Vec]: when the stack is full, just allocate a bigger stack, copy the data from the old stack to it and deallocate the old stack. Note that with this approach, it is possible to allocate memory that won't actually be used if a lot of stack space is needed by a goroutine only at the beginning of its execution.

## Rust and green threads

Rust actually had a runtime that supported both threads and green threads back in the day. The standard libraries would have to evolve to support both threading models which was something considered too much work, so the green thread support was removed[^rfc_0000-remove-runtime].

## Rust, stackless coroutines and compiler generated state machines

Rust supports the async await[^rust_async_book_async_await] model with a little compiler help which is a type of coroutines that also allows execution to be stopped and resumed, in this case, execution is suspended and resumed at `await` points.

```rust
async fn f() -> i32 {
  let x = g().await;
  let y = h().await;
  x + y
}
```

A runtime schedules tasks, [Futures][future] in this case, that cooperate by yielding control back to the runtime whenever an async operation is not ready at an `await` point. An `await` point is whenever `await` is found inside of a async context.

```rust
async fn f() -> i32 {
  let x = g().await; <- await point
  ..
  let y = h().await; <- await point
  x + y
}
```

The [Future] trait has a single method called that is meant to be called in order to make the computation move forward.

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

The `async` keyword tell the Rust compiler to generate a type that implements the [Future] trait, the future will have its `poll` method called possibly multiple times by the runtime until is able to signal that the computation is completed by producing a [Poll::Ready][poll] value. Each time it is polled, it will try to make as much progress as possible.

The async function `f` is transformed in a state machine (simplified) that tries to move through as many states as possible every time the future is polled. After taking a look at how `poll` is structured, it becomes clear why futures do not make progress until polled for the first time.

```rust
async fn f() -> i32 {
  let x = g().await;
  let y = h().await;
  x + y
}
```

```rust
enum Future_f {
  /// The initial state
  State0,
  /// The state after the first await point.
  State1 { x: i32 },
  /// The state after the second await point.
  State2 { x: i32, y: i32 },
  /// Completed
  State3,
}

impl Future for Future_f {
  type Output = i32;

  fn poll(mut self: Pin<&mut Self>, cx: &mut task::Context<'_>) -> Poll<Self::Output> {
    use Future_f::*;
    loop {
      match *self {
        State0 => match g().poll(cx) {
          Poll::Pending => return Poll::Pending,
          Poll::Ready(x) => {
            *self = State1 { x };
          }
        },
        State1 { x } => match h().poll(cx) {
          Poll::Pending => return Poll::Pending,
          Poll::Ready(y) => {
            *self = State2 { x, y };
          }
        },
        State2 { x, y } => {
          *self = State3;
          return Poll::Ready(x + y);
        }
        State3 => {
          panic!("polled completed future");
        }
      }
    }
  }
}
```

Note that instead of having a stack, each task holds only the data necessary to transition to the next state aka the coroutines are stackless. Having lightweight tasks allows the runtime to handle huge numbers of concurrent operations at once. This type of coroutines is known as stackless because unlike goroutines they do not need a stack to hold data necessary to complete the computation.

## Tokio

Go ships with its own runtime and scheduler[^morsmachine_go_scheduler] that decides which and when goroutines get a chance to run but Rust doesn't.

Rust isn't capable to execute futures by default, so a asynchronous runtime is needed. Enter tokio, an asynchronous runtime for the Rust programming language. It provides both a single-threaded and multi-threaded runtime for executing asynchronous code[^tokio_tutorial].

Both stackful and stackless coroutines shine in applications where most of the time is spent waiting on I/O instead of performing CPU heavy computations.

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
[goroutines]: https://go.dev/tour/concurrency/1
[go runtime]: https://go.dev/blog/go119runtime
[green threads]: https://en.wikipedia.org/wiki/Green_thread

[^nindalf_how_goroutines_work]: https://blog.nindalf.com/posts/how-goroutines-work/
[^pthread_create]: https://man7.org/linux/man-pages/man3/pthread_create.3.html

[context_switch]: https://en.wikipedia.org/wiki/Context_switch
[syscall]: https://en.wikipedia.org/wiki/System_call
[kernel]: https://en.wikipedia.org/wiki/Kernel_(operating_system)
[protection_ring]: https://en.wikipedia.org/wiki/Protection_ring
[vec]: https://doc.rust-lang.org/std/vec/struct.Vec.html

[^cloudflare_how_stacks_are_handled_in_go]: https://blog.cloudflare.com/how-stacks-are-handled-in-go/
[^docs_contiguous_stacks]: https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub

[fibers under the magnifying glass]: https://www.open-std.org/JTC1/SC22/WG21/docs/papers/2018/p1364r0.pdf

[^rfc_0000-remove-runtime]: https://github.com/aturon/rfcs/blob/remove-runtime/active/0000-remove-runtime.md
[^rust_async_book_async_await]: https://rust-lang.github.io/async-book/03_async_await/01_chapter.html

[future]: https://doc.rust-lang.org/std/future/trait.Future.html

[^morsmachine_go_scheduler]: https://morsmachine.dk/go-scheduler
[^tokio_tutorial]: https://tokio.rs/tokio/tutorial

[poll]: https://doc.rust-lang.org/stable/std/task/enum.Poll.html
