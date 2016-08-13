---
layout: post
title:  "Trait Bound Is Not Satisfied"
date:   2016-08-13 12:58:32 -0700
categories: rust traits types rtps
---
I'm trying to use Rust's trait system to build out a large library. The idea: leave room
for testing out multiple implementations in the running system -- the ability to switch things out so I can benchmark
as the library grows. The first implementation will use the simplest data types (read: `Vec`s with linear scans).

Now I'm running in to issues with the type checker and `PartialEq` and I think my use of traits has something to do
with it. I was hoping `#[derive(PartialEq)]` would suffice but I'm missing something.

Let's back up and look at the trait code:

```rust
// history_cache_trait.rs
use std::error::Error;

pub use super::super::super::common_types::*;


pub type ErrorStr = &'static str;
pub type AddResult = Result<(),ErrorStr>;

pub trait HistoryCacheTrait {
    fn new() -> Self;
    fn add_change(&mut self, change: CacheChange) -> AddResult;
    fn remove_change(&mut self, change: CacheChange) -> AddResult;
    fn get_change();
    fn get_seq_num_min();
    fn get_seq_num_max();
}
```

The code above is used for tracking state changes of a RTPS Writer. I imagine it will grow to become one of the
more complex bits of my library. A cache change is a wrapper around a buffer (so I won't muddy this post with its definition).

The first implementer, making heavy use of the `unimplemented!()` macro to side step actual implementation while I build things out:

```rust
// history_cache/mod.rs
use std::default::Default;

use super::super::common_types::*;
use super::{ HistoryCacheTrait, AddResult };

#[derive(Default)]
pub struct HistoryCache {
    changes: Vec<CacheChange>
}

impl HistoryCacheTrait for HistoryCache {
    // SNIP

    fn remove_change(&mut self, change: CacheChange) -> AddResult {
        let found = false;
        let i = 0;
        for c in self.changes.iter() {
            if c == change {

            }
        }

        unimplemented!()
    }

    // SNIP

}
```

when compiling the code I get this error:

```
   Compiling rtps v0.1.0 (file:///Users/xavierlange/code/dds/rtps)
error[E0277]: the trait bound `&common_types::cache_change::CacheChange: std::cmp::PartialEq<common_types::cache_change::CacheChange>` is not satisfied
  --> src/entity/history_cache/mod.rs:25:16
   |
25 |             if c == change {
   |                ^^^^^^^^^^^
   |
   = help: the following implementations were found:
   = help:   <common_types::cache_change::CacheChange as std::cmp::PartialEq>
```

