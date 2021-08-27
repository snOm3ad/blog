+++
author = "Oliver T."
title = "Lifetimes in Rust"
date = "2021-08-20"
description = ""
tags = [
    "rust",
    "lifetimes",
    "static",
    "threads",
    "move",
]
+++
### The Problem At A Glance

I think a lot of people who try to learn Rust eventually run into a problem with lifetimes. Most of the time the compiler is pretty helpful in what it is you should change to make things work. Other times, it may be harder to pinpoint the error! For example, I think everyone who first started learning Rust has at some point forgotten the `move` keyword right before a closure. It looks something like this:

```rust
use std::thread;

fn main() {
    let id = 1;
    let h = thread::spawn(|| {
        println!("{}", id);
    });
    let _ = h.join();
    println!("{}", id);
}
```

If you try and compile this, you would get an error saying that the closure may outlive the function (`main`) which owns `id`. Which sounds pretty clear, but it does not explain where the problem stems from nor should it try to, error messages should be short and concise! However, we still need to fill this gap in our understanding, so let's dip our toes in the waters of lifetimes.

### Understanding The Problem

In order to better understand what is going on, let us take a look at the function signature of `thread::spawn`:

```rust
pub fn spawn<F, T>(f: F) -> JoinHandle<T> 
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
```

The first time I saw this, it looked extremely complicated but became fairly manageable once broken down.

The most important part of the signature, for the purposes of understanding the problem, is `F: Send + 'static`, which is equivalent to saying `F: Send` and `F: 'static`.

Let's focus on the latter, `F: 'static` means that the type `F` must be *bounded* by a `'static` lifetime  [^1]. 

[^1]: There is a distinction between `T: <trait>` and `T: <lifetime>`, see [here for more](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md).

> When a type `T` is bounded by a `'static` lifetime it means that you must be able to hold to that type for an indefinitely period of time, including up to the end of the program.

The second key thing to understanding this problem is what restricts the lifetime of a closure. In Rust the lifetime of a closure is limited to the shortest lifetime of the borrows it makes from the context in which it is created.

Since our closure borrows a variable (`id`) located inside the stack frame of `main` it follows that a reference to that variable cannot possibly live indefinitely long, because the whole thing gets dropped by the end of `main`. Thus violating the accord that we just saw, this is what the `note` part of the compiler error message tells us in a more concise way.

```
note: function requires argument type to outlive `'static`
 --> src/main.rs:5:13
  |
5 |       let h = thread::spawn(|| {
  |  _____________^
6 | |         println!("{:?}", id);
7 | |     });
  | |______^
```

### Solutions

Now that we understand the problem, let's look at possible solutions. There are two ways to solve this problem:

- Add the `move` keyword right before the closure definition.
- Extend the lifetime of the borrow to `'static`.

The compiler's `help` section seems to always suggests the first one, perhaps because it is often easier. It even goes as far as telling us why it would solve the problem, by forcing the closure to take ownership of the variable it will no longer need to borrow it. 

So the lifetime of the closure is no longer limited by the lifetime of its borrows (because there are none) meaning it can live for an indefinite period of time.

```
help: to force the closure to take ownership of `id` (and any other referenced variables), use the `move` keyword
  |
5 |     let h = thread::spawn(move || {
  |                           ^^^^^^^
```

You can also solve this problem by extending the lifetime of the borrow to `'static`. To do so, you would need to make sure the variable `id` does not get dropped as soon as `main` exits, in other words extend the lifetime of `id` beyond that of `main`. The easiest way to do this is to use the `const` keyword, if we change our code to:

```rust
use std::thread;

fn main() {
    const ID: u8 = 0;
    let h = thread::spawn(|| {
        println!("{}", ID);
    });
    let _ = h.join();
    println!("{}", ID);
}
```

### Caveats

A common problem when using the first solution is that when the type is not `Copy` it cannot be used again in the context in which it was created. Take the following example,

```rust
use std::thread;

fn main() {
    let id = vec![1];
    let h = thread::spawn(move || {
        println!("{}", id);
    });
    let _ = h.join();
    println!("{}", id);
}
```

Here we get an error saying that we are attempting to borrow a value that has been moved. This is because, as the compiler stated earlier, the `move` keyword forces the closure to take ownership of all its borrows.

```
error[E0382]: borrow of moved value: `id`
```

The common and most preferred way to fix for this problem is to clone the variable and pass the copy to the closure. While this works for almost any case I can think of, there are times where it is undesirable to copy the data.

For those cases, most heap allocated standard library types provide a method called `leak`, which allows you to get a reference to their inner data. Note that this means you are responsible for freeing the memory that was allocated, if you don't then your program will leak memory. While this is mostly undesirable behavior, there are cases where it becomes necessary such as FFI. It is possible to extend the lifetime of heap allocated data to that of `'static` by simply reassigning, like this:

```rust
use std::thread;

fn main() {
    let id = vec![1];
    let id: &'static [i32] = id.leak();
    let h = thread::spawn(move || {
        println!("{}", id);
    });
    let _ = h.join();
    println!("{}", id);
}
```
