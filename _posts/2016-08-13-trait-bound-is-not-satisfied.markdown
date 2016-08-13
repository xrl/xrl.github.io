---
layout: post
title:  "Trait Bound Is Not Satisfied"
date:   2016-08-13 12:58:32 -0700
categories: rust traits types rtps
---
I'm using [Rust](https://www.rust-lang.org/) and its [trait system](https://doc.rust-lang.org/book/traits.html) to
build out a large library. The idea: leave room
for testing out multiple implementations and get some succinct contracts. This will give me the ability to
A/B test and benchmark different parts as the library grows. The first round of implementations will use the simplest data
types (read: `Vec`s with linear scans).

Now I'm running in to issues with the type checker and `PartialEq` and I think my use of traits has something to do
with it. I was hoping `#[derive(PartialEq)]` would suffice but I'm missing something.

Let's back up and look at the trait code:

```rust
// history_cache_trait.rs
pub use super::super::super::common_types::*;

pub type ErrorStr = &'static str;
pub type HistoryCacheResult = Result<(),ErrorStr>;

pub trait HistoryCacheTrait {
    fn new() -> Self;
    fn add_change(&mut self, change: CacheChange) -> HistoryCacheResult;
    fn remove_change(&mut self, change: CacheChange) -> HistoryCacheResult;
    fn get_change();
    fn get_seq_num_min();
    fn get_seq_num_max();
}
```

The code above is used for tracking state changes. I imagine it will grow to become one of the
more complex bits of my library. A cache change is a wrapper around a buffer but I won't muddy this post with its definition.

The first implementer makes heavy use of the `unimplemented!()` macro to side step actual implementation but here's the suspicious implentation:

```rust
// history_cache/mod.rs
use std::default::Default;

use super::super::common_types::*;
use super::{ HistoryCacheTrait, HistoryCacheResult };

#[derive(Default)]
pub struct HistoryCache {
    changes: Vec<CacheChange>
}

impl HistoryCacheTrait for HistoryCache {
    // SNIP
    fn remove_change(&mut self, change: CacheChange) -> HistoryCacheResult {
        for c in &self.changes {
            if *c == change {

            }
        }

        unimplemented!()
    }
    // SNIP
}
```

when compiling the code I get this error (courtesy of [new formatting in Rust nightly](https://blog.rust-lang.org/2016/08/10/Shape-of-errors-to-come.html)):

```
   Compiling rtps v0.1.0 (file:///Users/xavierlange/code/dds/rtps)
error[E0277]: the trait bound `&common_types::cache_change::CacheChange: std::cmp::PartialEq<common_types::cache_change::CacheChange>` is not satisfied
  --> src/entity/history_cache/mod.rs:23:16
   |
23 |             if c == change {
   |                ^^^^^^^^^^^
   |
   = help: the following implementations were found:
   = help:   <common_types::cache_change::CacheChange as std::cmp::PartialEq>
```

I added the `#[derive(PartialEq)]` to `CacheChange` and even then I'm using the concrete type `CacheChange` and not some
interface. I am using the `PartialEq` inside of the `HistoryCache`'s `HistoryCacheTrait` impl. Is that the issue? The
one interesting bit I see in the error is `&common_types::cache_change::CacheChange` (focus on the ampersand) --
looks like a borrow is perhaps messing things up?

It can get confusing to read the error. I'm pretty confident it's not the `#[derive(PartialEq)]` causing the issue. And I
have read code which derefs (`*`) values. And I think I read somewhere that `for` makes an iterator which borrows values.
That would explain the `&` in the error! Let's try this definition:

```rust
    fn remove_change(&mut self, change: CacheChange) -> HistoryCacheResult {
        for c in &self.changes {
            if *c == change {

            }
        }

        unimplemented!()
    }
```

Now we have success:

```
   Compiling rtps v0.1.0 (file:///Users/xavierlange/code/dds/rtps)
    Finished debug [unoptimized + debuginfo] target(s) in 0.90 secs
```

My rust-fu went up a little bit by putting together all the pieces. Hope this post saves you some time when you hit your
next `trait bound` issue (hint: make sure you and error text agree about the borrow state of the variable!).