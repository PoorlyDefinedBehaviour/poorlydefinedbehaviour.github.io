---
title: "Why Rc<T> is not Send"
date: 2022-03-27T17:20:07-03:00
categories: ["rust"]
draft: false
---

# Why Rc<T> cannot be sent between threads

We get an compile error if we try to send `Rc<T>` to another thread:

```rust
use std::rc::Rc;

fn main() {
  let rc = Rc::new(1);
  std::thread::spawn(|| {
    println!("{}", *rc);
  })
  .join();
}

error[E0277]: `Rc<i32>` cannot be shared between threads safely
   --> src/main.rs:5:3
    |
5   |   std::thread::spawn(|| {
    |   ^^^^^^^^^^^^^^^^^^ `Rc<i32>` cannot be shared between threads safely
    |
    = help: the trait `Sync` is not implemented for `Rc<i32>`
    = note: required because of the requirements on the impl of `Send` for `&Rc<i32>`
    = note: required because it appears within the type `[closure@src/main.rs:5:22: 7:4]`
note: required by a bound in `spawn`
   --> /home/bruno/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/src/rust/library/std/src/thread/mod.rs:625:8
    |
625 |     F: Send + 'static,
    |        ^^^^ required by this bound in `spawn`

For more information about this error, try `rustc --explain E0277`.
```

The compile error is triggered because the closure passed to [std::thread::spawn](https://doc.rust-lang.org/std/thread/fn.spawn.html) must be [Send](https://doc.rust-lang.org/std/marker/trait.Send.html). Types that implement `Send` are types that can be transferred across thread boundaries.

## Rc primer

`Rc<T>` is a smart pointer that can be used to hold multiple references to `T` when `T` is owned by several objects. It kinda looks like a [std::shared_ptr](https://en.cppreference.com/w/cpp/memory/shared_ptr) from C++ except it does not increment and decrement the reference count atomically, if you need thread safe reference counting in Rust take a look at [Arc](https://doc.rust-lang.org/std/sync/struct.Arc.html).

> A value contained in an `Rc<T>` will be dropped when the last reference to it is dropped. It is possible to create reference cycles and leak memory as well.

If we need a new reference to a `T`, the `Rc<T>` can just be cloned:

```rust
let a = Rc::new(1);
let b = Rc::clone(&a);
```

### Rc internals

If we take a look at the `Rc<T>` [source code](https://github.com/rust-lang/rust/blob/5aba816672d08a076eaa8005a109968af8ce1083/library/alloc/src/rc.rs#L284) we sill see that it is actually kinda simple:

```rust
pub struct Rc<T: ?Sized> {
    ptr: NonNull<RcBox<T>>,
    phantom: PhantomData<RcBox<T>>,
}
```

> `?Sized` means the size of `T` does not need to be known at compile-time. It's fine to accept a `T` that's not `Sized` because `Rc<T>` is `Sized`.

A `Rc<T>` pretty must boils down to a struct with two counters and pointer to a value of type `T`. In the `Rc<T>` source code, the struct is called `RcBox<T>`:

```rust
// Note that Cell is used for internal mutability.
struct RcBox<T: ?Sized> {
    /// How many references we have to this value.
    strong: Cell<usize>,
    // Weak ref? What's that?
    weak: Cell<usize>,
    /// The actual value
    value: T,
}
```

When a `Rc<T>` is created, its `strong` count will be `1` because there is only one reference to the value inside of it. If we need more references (the point of using `Rc<T>`) we can just clone the `Rc<T>`

```rust
  let a: Rc<String> = Rc::new(String::from("hello world"));
  let b: Rc<String> = a.clone();
```

Cloning an `Rc<T>` means increasing its `strong` count and creating a copy of the `RcBox<T>`.

```rust
impl<T: ?Sized> Clone for Rc<T> {
    #[inline]
    fn clone(&self) -> Rc<T> {
        unsafe {
            // inner() returns the &RcBox<T> that's in the Rc<T> struct.
            self.inner().inc_strong();
            Self::from_inner(self.ptr)
        }
    }
}
```

`inc_strong` literally just increments the `strong` counter besides some safety checks:

```rust
 #[inline]
fn inc_strong(&self) {
  let strong = self.strong();

  // We want to abort on overflow instead of dropping the value.
  // The reference count will never be zero when this is called;
  // nevertheless, we insert an abort here to hint LLVM at
  // an otherwise missed optimization.
  if strong == 0 || strong == usize::MAX {
    abort();
  }
  self.strong_ref().set(strong + 1);
}
```

and `from_inner` just copies the pointer to `RcBox<T>`:

```rust
unsafe fn from_inner(ptr: NonNull<RcBox<T>>) -> Self {
  Self { ptr, phantom: PhantomData }
}
```

After the clone, this is how things look like:

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/160302841-b04e1b5e-aab1-4afd-b608-51431eaab181.png" />
</p>

## Why Rc<T> is not Send after all?

Every time a `Rc<T>` is cloned, its `strong` count is incremented. If we had two or more threads trying to clone an `Rc<T>` at the same time, there would be a race condition since the access to the `strong` count that's in the `RcBox<T>` is not synchronized.

<p align="center">
  <img src="https://user-images.githubusercontent.com/17282221/160303759-f89c1cf8-406f-4290-81ce-7e0f86559e3c.png" />
</p>

# References

https://github.com/rust-lang/rust  
https://doc.rust-lang.org/std/marker/trait.Send.html  
https://doc.rust-lang.org/nomicon/send-and-sync.html  
https://doc.rust-lang.org/std/rc/struct.Rc.html  
https://doc.rust-lang.org/std/sync/struct.Arc.html  
https://doc.rust-lang.org/std/ptr/struct.NonNull.html
