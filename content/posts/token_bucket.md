---
title: "Token bucket"
date: 2022-03-20T22:21:07-03:00
categories: ["today-i-learned", "algorithms", "rate-limiting"]
draft: false
---

# Intro

[Token bucket](https://en.wikipedia.org/wiki/Token_bucket) is an algorithm that can be used to rate limit requests received by a service.

# How it works

The algorithm is called token bucket because of they way it works: imagine we have a bucket with `x` tokens where each accepted request consumes one token from the bucket and a token is added back to the bucket at an interval.

A bucket with `1` token that is refilled each second means the service accepts one request per second.

A bucket with `5` tokens that are refilled every `1/5` seconds means the service accepts `5` requests per second.

A bucket with `x` tokens that are refilled every `1/x` seconds means the service accepts `x` requests per second.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/159195529-7cfe3a52-c280-4255-a099-0e5b9fe3902b.png"/>
</p>

Requests that are received when the bucket is empty can just be dropped or enqueued to be handled later.

# Implementation in Rust

The source code can be found [here](https://github.com/PoorlyDefinedBehaviour/token_bucket).

The bucket will accept `x` requests per second.

```rust
#[derive(Debug)]
struct Config {
  /// The number of requests that can be accepted every second.
  requests_per_second: usize,
}

#[derive(Debug)]
struct Bucket {
  config: Config,
  /// How many requests we can accept at this time.
  tokens: AtomicUsize,
  /// Sends are actually never made in this channel.
  /// It is used only for the worker thread to know when the bucket
  /// has been dropped and exit.
  close_channel_sender: Sender<()>,
}
```

A thread is spawned to refill the bucket every `1/Config::requests_per_second`, at this rate the bucket will accept around `Config::requests_per_second` requests per second.

```rust
impl Bucket {
  pub fn new(config: Config) -> Arc<Self> {
    let (sender, receiver) = crossbeam_channel::unbounded::<()>();

    let tokens = AtomicUsize::new(1);

    let bucket = Arc::new(Self {
      config,
      tokens,
      close_channel_sender: sender,
    });

    let bucket_clone = Arc::downgrade(&bucket);
    std::\thread::spawn(move || Bucket::add_tokens_to_bucket_on_interval(bucket_clone, receiver));

    bucket
  }

  fn add_tokens_to_bucket_on_interval(bucket: Weak<Bucket>, receiver: Receiver<()>) {
    let interval = {
      match bucket.upgrade() {
        None => {
          error!(
            "unable to define interval to add tokens to bucket because bucket has been dropped"
          );
          return;
        }
        Some(bucket) => Duration::from_secs_f64(1.0 / (bucket.config.requests_per_second as f64)),
      }
    };

    debug!(?interval, "will add tokens to bucket at interval");

    let ticker = crossbeam_channel::tick(interval);

    loop {
      select! {
        recv(ticker) -> _ => match bucket.upgrade() {
          None => {
            debug!("cannot upgrade Weak ref to Arc, exiting");
            return;
          }
          Some(bucket) => {
            let _ = bucket
              .tokens
              .fetch_update(Ordering::SeqCst, Ordering::SeqCst, |tokens| {
                Some(std::cmp::min(tokens + 1, bucket.config.requests_per_second))
              });
          }
        },
        recv(receiver) -> message => {
          // An error is returned when we try to received from a channel that has been closed
          // and this channel will only be closed when the bucket is dropped.
          if message == Err(RecvError) {
            debug!("bucket has been dropped, won't add try to add tokens to the bucket anymore");
            return;
          }
        }
      }
    }
  }
}
```

And a function can be called to find out if there's enough tokens in the bucket to accept a request. A token is consumed if the request is accepted.

```rust
impl Bucket {
  ...

  /// Returns true if there's enough tokens in the bucket.
  pub fn acquire(&self) -> bool {
    self
      .tokens
      .fetch_update(Ordering::SeqCst, Ordering::SeqCst, |tokens| {
        Some(if tokens > 0 { tokens - 1 } else { tokens })
      })
      .map(|tokens_in_the_bucket| tokens_in_the_bucket > 0)
      .unwrap_or(false)
  }
}
```

# References

https://en.wikipedia.org/wiki/Token_bucket
