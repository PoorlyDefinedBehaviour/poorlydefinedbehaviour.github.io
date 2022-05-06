---
title: "Contributing to Rust and tokio"
date: 2022-04-17T14:29:20-03:00
draft: false
---

# Contributing for the first time

I have been trying to force myself to do harder things lately in order to practice and learn new things. Since i'm doing [Rust] full time now, i thought it would be a good a idea to contribute to the ecosystem, so i went and enabled notifications for a bunch of [Rust] related projects and for the [Rust] project itself.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/167045101-fc8e894d-d5e6-4b75-9963-c97cc48fd557.png" />
</p>
<p align="center">
  <i>I thought i would be able to keep up with the notifications. I was wrong. (obviously)</i>
</p>

I actually go through a few notifications each day in hope to find something to work on.

# First tokio contribution

[Rust] supports [async await] but it does not come with a [runtime](https://www.ncameron.org/blog/what-is-an-async-runtime/) by default. It is left for the user to define which runtime their program will use and [tokio] is the most popular one.

I was going through my notifications as usual and one issue caught my attention: someone wanted to add a method to get the address the [UdpSocket] is connected to.

It seemed easy enough so i went and claimed it:

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/167046100-d789d0d8-aa0d-49aa-8caf-ccbec9a15d52.png" />
</p>

The [implementation](https://github.com/tokio-rs/tokio/pull/4611/files) was actually pretty simple since [mio] already had a method that does exact what i needed.

> From [mio] Github repository: Mio is a fast, low-level I/O library for Rust focusing on non-blocking APIs and event notification for building high performance I/O apps with as little overhead as possible over the OS abstractions.

Everything went as expected and my change got released on tokio [1.18.0](https://github.com/tokio-rs/tokio/pull/4641).

# First Rust contribution

A few days went by and a [Rust] issue caught my attention: a compiler message was incorrect, it turns out, fixing compiler messages is one of the main ways people start contributing to the [Rust] compiler.

Anyway, [Rust] is known for its nice error messages, it does have good error messages indeed but they come at a development cost. The [Rust] compiler has several functions and methods just to decide which error message to show the user.

## The offender

It is actually valid to add `:` after a type variable

```rust
fn foo<T:>(t: T) {
  t.clone();
}
```

> note the `:` after `T`

The compiler would then complain that [Clone] is not impleted for `T` and suggest it to be implemented

```terminal
error[E0599]: no method named `clone` found for type parameter `T` in the current scope
 --> src/lib.rs:2:7
  |
2 |     t.clone();
  |       ^^^^^ method not found in `T`
  |
  = help: items from traits can only be used if the type parameter is bounded by the trait
help: the following trait defines an item `clone`, perhaps you need to restrict type parameter `T` with it:
  |
1 | fn foo<T: Clone:>(t: T) {
  |        ~~~~~~~~
```

> note there is an extra `:` after `Clone` in the suggestion

I thought it was easy enough and decided to fix it.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/167047347-0354ba5b-44d7-42d4-b179-412f760758d8.png" />
</p>

Testing that the compiler error messages are correct is pretty easy, [Rust] calls this type of test and `ui` test.

All i need to do was to addd a file containing the code that's supposed to error to the `ui` folder which contains `ui` tests

```rust
~src/test/ui/traits/issue-95898.rs

// Test for #95898: The trait suggestion had an extra `:` after the trait.
// edition:2021

fn foo<T:>(t: T) {
    t.clone();
    //~^ ERROR no method named `clone` found for type parameter `T` in the current scope
}

fn main() {}
```

> `~^ ERROR` says the error is expected and the test fails if the error does not occur

and a `.stderr` file containing the expected error message

```terminal
error[E0599]: no method named `clone` found for type parameter `T` in the current scope
  --> $DIR/issue-95898.rs:5:7
   |
LL |     t.clone();
   |       ^^^^^ method not found in `T`
   |
   = help: items from traits can only be used if the type parameter is bounded by the trait
help: the following trait defines an item `clone`, perhaps you need to restrict type parameter `T` with it:
   |
LL | fn foo<T: Clone>(t: T) {
   |        ~~~~~~~~

error: aborting due to previous error

For more information about this error, try `rustc --explain E0599`.
```

> Note that we expect the suggestion to be correct in the `.stderr` file

It took me some time to get used to the compiler but the [fix](https://github.com/rust-lang/rust/pull/95991/files) was really easy thanks to [WaffleLapkin] who was working on a similar issue.

# Second tokio contribution

[tokio] has the [join!] macro that can be used when we want to wait for several futures to complete before doing something.

> Think [Promise.all] if javascript is your thing.

```rust
async fn process_something_1() { ... }
async fn process_something_2() { ... }

#[tokio::main]
async fn main() {
  let (result_1, result_2) = tokio::join!(result_1, result_2);
  ...
}
```

[Future] based concurrency is a [cooperative](https://en.wikipedia.org/wiki/Cooperative_multitasking) model, it is pretty easy for one task to monopolize processing time if we are not careful. One way to work around this problem is to not allow a task to run forever without being interrupted by [giving each task a budget](https://tokio.rs/blog/2020-04-preemption) and force the task to yield control back to the runtime whenever its budget is exceeded.

> Task and Future will be used interchangeably

[tokio] does the budget think per task and each time a task interacts with a resource, its budget is decreased until it reaches 0 and control is yielded back to the scheduler.

Each task [starts](https://github.com/tokio-rs/tokio/blob/c5ff797dcfb44f003cc9cb2080a5f2544c3cca7f/tokio/src/coop.rs#L55) with a budget of 128 and the budget is consumed when interacting with a resource (a [Semaphore], for example)

```rust
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::Semaphore;

async fn foo() {
  // Consuming a resource decreases the budget by 1.
  tokio::time::sleep(Duration).await;
}

async fn foo() {
  // Consuming a resource decreases the budget by 1.
  let _permit = permits.clone().acquire_owned().await.unwrap();
}

// main is a task with a budget of 128
#[tokio::main]
async fn main() {
  let permits = Arc::new(Semaphore::new(1));
  // NOTE: join! creates a new task with a budget of 128
  let _ = tokio::join!(
    foo(),
    bar(Arc::clone(&permits)),
  );
}
```

The point of giving a budget to each task to stop bad tasks from starving other tasks but it turns out, it is still possible for one task to starve other tasks because [join!] polls every future inside [the same task](https://docs.rs/tokio/latest/tokio/macro.join.html#runtime-characteristics) which means every future passed to [join!] shares the same task budget of `128`.

A task can starve other tasks by just consuming the whole budget of the task that invoked [join!] so by the time the other tasks passed to [join!] are polled, the budget is already 0 which causes them to yield control back to the runtime.

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
  let permits = Arc::new(Semaphore::new(1));

  // join! polls futures in the order they are passed to it.
  tokio::join!(
    // This future will be polled first.
    non_cooperative_task(Arc::clone(&permits)),
    // This future will be polled second.
    poor_little_task(permits)
  );
}

async fn non_cooperative_task(permits: Arc<Semaphore>) {
  // This future will yield back to the runtime after the loop runs 128 times.
  loop {
    let _permit = permits.clone().acquire_owned().await.unwrap();
  }
}

// `non_cooperative_task` has been polled and now it is this futures turn.
// The bad thing is that `non_cooperative_task` consumed the whole budget
// and there's nothing left for this future to spend.
async fn poor_little_task(permits: Arc<Semaphore>) {
  loop {
    // Even though this future should be able to acquire the Semaphore,
    // acquire_owned().await will return Poll::Pending because the budget of
    // the current task is 0.
    let _permit = permits.clone().acquire_owned().await.unwrap();
    // This println! never gets to run.
    println!("Hello!")
  }
}

```

> [Poll] is the type returned when the runtime checks if a [Future] is completed.
> [Poll::pending] means the [Future] is not ready. In this case, the [Future] is actually ready but since it has no budget to spend, it pretends it isn't ready.

## First try

At first i thought we would just be able to give each future passed to [join!] its own budget instead of letting them share the current task budget.

> By current task budget, i mean the budget of the task that invoked `join!`

[join!] is implemented as declarative macro

```rust
macro_rules! join {
    (@ {
        // One `_` for each branch in the `join!` macro. This is not used once
        // normalization is complete.
        ( $($count:tt)* )

        // Normalized join! branches
        $( ( $($skip:tt)* ) $e:expr, )*

    }) => {{
        use $crate::macros::support::{maybe_done, poll_fn, Future, Pin};
        use $crate::macros::support::Poll::{Ready, Pending};

        // Safety: nothing must be moved out of `futures`. This is to satisfy
        // the requirement of `Pin::new_unchecked` called below.
        let mut futures = ( $( maybe_done($e), )* );

        poll_fn(move |cx| {
            let mut is_pending = false;

            $(
                // Extract the future for this branch from the tuple.
                let ( $($skip,)* fut, .. ) = &mut futures;

                // Safety: future is stored on the stack above
                // and never moved.
                let mut fut = unsafe { Pin::new_unchecked(fut) };

                // Try polling
                if fut.poll(cx).is_pending() {
                    is_pending = true;
                }
            )*

            if is_pending {
                Pending
            } else {
                Ready(($({
                    // Extract the future for this branch from the tuple.
                    let ( $($skip,)* fut, .. ) = &mut futures;

                    // Safety: future is stored on the stack above
                    // and never moved.
                    let mut fut = unsafe { Pin::new_unchecked(fut) };

                    fut.take_output().expect("expected completed future")
                },)*))
            }
        }).await
    }};

    // ===== Normalize =====

    (@ { ( $($s:tt)* ) $($t:tt)* } $e:expr, $($r:tt)* ) => {
        $crate::join!(@{ ($($s)* _) $($t)* ($($s)*) $e, } $($r)*)
    };

    // ===== Entry point =====

    ( $($e:expr),* $(,)?) => {
        $crate::join!(@{ () } $($e,)*)
    };
}
```

So i went on and just gave each future its own budget before polling them.

```rust
macro_rules! join {
  ...
            $(
                // Extract the future for this branch from the tuple.
                let ( $($skip,)* fut, .. ) = &mut futures;

                // Safety: future is stored on the stack above
                // and never moved.
                let mut fut = unsafe { Pin::new_unchecked(fut) };

                // Try polling
                if crate::coop::budget(|| fut.poll(cx)).is_pending() {
                    is_pending = true;
                }
            )*
  ...
}
```

> Note that i added `crate::coop::budget`

Turns out this doesn't work since. It is still pretty easy to create a future that never yields even though it consumes its whole budget:

```rust
...
loop {
    tokio::join!(sem.acquire());
}
```

The future would spend its budget but not the budget of the surrounding task, causing it to never yield.

## Second try

Each time the task created by [join!] is polled, poll a different future first so as time goes by, every future gets a chance to make progress.

I took a look at [select!] and it is able to do just that (up to 64 branches) so i took note and modified [join!].

```rust
macro_rules! join {
  ...
  let mut start = 0;
  ...
  // BRANCHES is the number of futures passed to join!.
  for i in 0..BRANCHES {
    let branch;
    #[allow(clippy::modulo_one)]
    {
      branch = (start + i) % BRANCHES;
    }
    match {
    $(
      // $crate::count! will return the number of tokens passed to it
      // up to 64 tokens.
      $crate::count!( $($skip)* ) => {
        // Extract the future for this branch from the tuple.
        let ( $($skip,)* fut, .. ) = &mut futures;

        // Safety: future is stored on the stack above
        // and never moved.
        let mut fut = unsafe { Pin::new_unchecked(fut) };

        // Try polling
        if crate::coop::budget(|| fut.poll(cx)).is_pending() {
          is_pending = true;
        }
      }
    )*
    }


  #[allow(clippy::modulo_one)]
  {
    start = (start + 1) % BRANCHES;
  }
  ...
}
```

This actually works but `$crate::count!` can only count up to 64:

```rust
#[macro_export]
#[doc(hidden)]
macro_rules! count {
    () => {
        0
    };
    (_) => {
        1
    };
    (_ _) => {
        2
    };
    (_ _ _) => {
        3
    };
    ...
    // up to 64
}
```

aaaand... [join!] accepts up to 125 futures without changing the [recursion limit] so this solution wasn't accepted because it would be a breaking change.

## Third try

Start the polling round in a different future each time still seems like a good idea. To bypass `$crate::count!`'s limitation, i decided to use a [procedural macro].

> Not actually showing code for this one because it is too long

Turns out people don't like [procedural macro]s very much and it was not accepted.

## Fourth try

Still the same solution but implemented in a different way. What if instead of using `$crate::count!` inside the macro to get the index of a future, we counted up front?

[join!] already does some normalization before actually processing the input, so i modified the normalization branches to pair the future index with the future itself.

```rust
macro_rules! join {
     (@ {
        // One `_` for each branch in the `join!` macro. This is not used once
        // normalization is complete.
        ( $($count:tt)* )

        // The expression `0+1+1+ ... +1` equal to the number of branches.
        ( $($total:tt)* )

        // Normalized join! branches
        $( ( $($skip:tt)* ) ( $($branch_index:tt)* ) $e:expr, )*

    }) => {{
        ...
        let mut start = 0;
        ...
        // BRANCHES is the number of futures passed to join!.
        for i in 0..BRANCHES {
            let branch;
            #[allow(clippy::modulo_one)]
            {
                branch = (start + i) % BRANCHES;
            }

            $(
              {
                const INDEX: u32 = $($branch_index)*;
                if branch == INDEX {
                    // Extract the future for this branch from the tuple.
                    let ( $($skip,)* fut, .. ) = &mut futures;

                    // Safety: future is stored on the stack above
                    // and never moved.
                    let mut fut = unsafe { Pin::new_unchecked(fut) };

                    // Try polling
                    if crate::coop::budget(|| fut.poll(cx)).is_pending() {
                      is_pending = true;
                    }
                }
              }
          )*
      }

      #[allow(clippy::modulo_one)]
      {
        start = (start + 1) % BRANCHES;
      }
      ...
    }};


   // ===== Normalize =====

    (@ { ( $($s:tt)* ) ( $($n:tt)* ) $($t:tt)* } $e:expr, $($r:tt)* ) => {
                                                          // i'm new ▼
        $crate::join!(@{ ($($s)* _) ($($n)* + 1) $($t)* ($($s)*) ($($n)*) $e, } $($r)*)
    };

    // ===== Entry point =====

    ( $($e:expr),* $(,)?) => {
                  // i'm new ▼
        $crate::join!(@{ () (0) } $($e,)*)
    };
}
```

It works but could be faster. Say we pass 5 futures to `join!`, how many times would the if statements that check if it is the future's turn to be polled conditions be checked?

## Fifth try (the last one)

The same idea still, poll a different future first every time, except we avoid checking if statement conditions without necessity.

```rust
macro_rules! join {
    (@ {
        // One `_` for each branch in the `join!` macro. This is not used once
        // normalization is complete.
        ( $($count:tt)* )

        // The expression `0+1+1+ ... +1` equal to the number of branches.
        ( $($total:tt)* )

        // Normalized join! branches
        $( ( $($skip:tt)* ) $e:expr, )*

    }) => {{
        use $crate::macros::support::{maybe_done, poll_fn, Future, Pin};
        use $crate::macros::support::Poll::{Ready, Pending};

        // Safety: nothing must be moved out of `futures`. This is to satisfy
        // the requirement of `Pin::new_unchecked` called below.
        let mut futures = ( $( maybe_done($e), )* );

        // Each time the future created by poll_fn is polled, a different future will be polled first
        // to ensure every future passed to join! gets a chance to make progress even if
        // one of the futures consumes the whole budget.
        //
        // This is number of futures that will be skipped in the first loop
        // iteration the next time.
        let mut skip_next_time: u32 = 0;

        poll_fn(move |cx| {
            const COUNT: u32 = $($total)*;

            let mut is_pending = false;

            let mut to_run = COUNT;

            // The number of futures that will be skipped in the first loop iteration.
            let mut skip = skip_next_time;

            skip_next_time = if skip + 1 == COUNT { 0 } else { skip + 1 };

            // This loop runs twice and the first `skip` futures
            // are not polled in the first iteration.
            loop {
            $(
                if skip == 0 {
                    if to_run == 0 {
                        // Every future has been polled
                        break;
                    }
                    to_run -= 1;

                    // Extract the future for this branch from the tuple.
                    let ( $($skip,)* fut, .. ) = &mut futures;

                    // Safety: future is stored on the stack above
                    // and never moved.
                    let mut fut = unsafe { Pin::new_unchecked(fut) };

                    // Try polling
                    if fut.poll(cx).is_pending() {
                        is_pending = true;
                    }
                } else {
                    // Future skipped, one less future to skip in the next iteration
                    skip -= 1;
                }
            )*
            }

            if is_pending {
                Pending
            } else {
                Ready(($({
                    // Extract the future for this branch from the tuple.
                    let ( $($skip,)* fut, .. ) = &mut futures;

                    // Safety: future is stored on the stack above
                    // and never moved.
                    let mut fut = unsafe { Pin::new_unchecked(fut) };

                    fut.take_output().expect("expected completed future")
                },)*))
            }
        }).await
    }};
}
```

# Thanks

Honestly, i had a lot of help from [Darksonn].

[rust]: https://www.rust-lang.org/
[future]: https://doc.rust-lang.org/std/future/trait.Future.html
[async await]: https://doc.rust-lang.org/std/keyword.async.html
[tokio]: https://github.com/tokio-rs/tokio
[udpsocket]: https://docs.rs/tokio/latest/tokio/net/struct.UdpSocket.html
[mio]: https://github.com/tokio-rs/mio
[clone]: https://doc.rust-lang.org/std/clone/trait.Clone.html
[wafflelapkin]: https://github.com/WaffleLapkin
[join!]: https://docs.rs/tokio/latest/tokio/macro.join.html
[promise.all]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/all
[semaphore]: https://docs.rs/tokio/0.2.6/tokio/sync/struct.Semaphore.html
[poll]: https://doc.rust-lang.org/stable/std/task/enum.Poll.html
[poll::pending]: https://doc.rust-lang.org/stable/std/task/enum.Poll.html
[select!]: https://docs.rs/tokio/latest/tokio/macro.select.html
[recursion limit]: https://doc.rust-lang.org/reference/attributes/limits.html
[procedural macro]: https://doc.rust-lang.org/reference/procedural-macros.html
[darksonn]: https://github.com/Darksonn
