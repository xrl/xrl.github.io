---
layout: post
title:  "IntelliJ/Rust are becoming buddies"
date:   2016-10-09 12:58:32 -0700
categories: rust
---

IntelliJ/Rust are becoming buddies
---

Most of the time I do everything in Sublime Edit 3 -- I like the syntax highlighting, intuitive multi-caret support, and speed of grepping
through my project or jumping to a file based on fuzzy filename matching. But it's good to try new dev environments and the more I write Rust the more I want to
leverage the rich type system to be more productive on large projects. And while I'm at it I want to support the Rust commercial support ecosystem: I'm
giving the IntelliJ IDE (by Idea Software) a full shake.

IntelliJ CE has been exclusive Rust editor. And over the past few months it has gotten better and better. While still considered a work in progress it has helped me gain productivity.

In this post I want to go over everyday usage and hopefully convince you to give Intellij and the Rust plugin a try.

Setup
---

You'll want to get the IntelliJ CE (read: free to use) edition. Go to the [jetbrains download page](https://www.jetbrains.com/idea/download/).
Once you have the base IDE installed you can add the Rust plugin from inside IntelliJ CE (`IntelliJ > Preferences > Plugins > Browse Repositores`, search for Rust, click install.
Or: [follow the IntelliJ rust's instructions (click "install" for the pop up with actual instructions)](https://intellij-rust.github.io/).

You will also need to clone down the entire Rust compiler project from github:

```
git clone https://github.com/rust-lang/rust.git
```

put that somewhere sane. I do all my coding from inside of `~/code/`.

Then you need to go to `IntelliJ IDEA > Preferences > Languages & Frameworks > Rust` and set `Standard Lib` to something like `/Users/xavierlange/code/rust/rust/src` (note this is the `rust/src` folder inside of the cloned rust compiler project).

What about Racer?
---

You may have read about Rust's racer project for a common IDE engine, enabling a variety of introspection and code
creation behaviors common to Rust. The Intellij Rust plugin won't be using that. The Idea folks have forged their own path,
in their own words from [the FAQ](https://intellij-rust.github.io/docs/faq.html):


    We would be able to leverage IntelliJ Platform infrastructure
    for incremental analysis and indexing. With our own analysis
    we can provide more flexible quick fixes, intentions and
    typing assistance.


OK, that's fine. If they are offering a more seamless experience with less config then that keeps me closer to what I like about Sublime. I'm not a gear head when it comes to IDEs -- less config is better for me.

Key combos + helpful pointers for IntelliJ
---

 * `cmd-b` and click on any type, in your crate or in the stdlib: jump to the definition.
 * `cmd-[` or `cmd-]`: goes backwards or forwards in your 'jump' history. Great so you can get back to your code.
 * `cmd-shift-o`: grep filenames in your project, hit `enter` to open file. Not as fuzzy as Sublime Edit 3, won't tolerate misspellings.
 * `alt-shift` and click: add another caret to the editor. Hit `esc` to leave multi-caret mode. Strictly less powerful than Sublime because you cannot click and drag carets across multiple lines. But can be handy.
 * `ctrl-r`: run your test suite. You'll have to configure your run configurations but there's an easy cargo target. Just set the command to `test` and make sure you click `single instance only`. I highly recommend you click `Show backtrace on panic` which sets the environmental variable `RUST_BACKTRACE=1`. Super handy.
 * `ctrl-shift-r`: run the current test based on your cursor (but make sure you select the 'Cargo Test' target when you're done! it won't switch back automatically)
 * `cmd-shift-r`: grep through all the code in your project
 * `cmd-1`/`cmd-4`: toggle project drawer/toggle run drawer

Those keyboard shortcuts are the MVPs.

The integration with compiler errors is great. I run Rust nightly and the plugin has no problem highlighting errors so you can click and jump to definition. Handy when working through a backlog of compiler errors -- which is very common when writing Rust!

Git integration is very workable; there's also built in terminal if you want to skip the wizard-y interface.

Warts
---

 * Multi-caret is less useful than Sublime 3. Click and drag is key.
 * Cannot derive types in situations: loops, iterators, closures. Means you can't jump to definition.
 * Sometimes cannot cross crate boundaries. I use a single git repo with two root modules: the library module and my test suite. The rust plugin cannot jump from the test suite to the library. Fortunately it just disable the hover.
 * `cmd-1`, `cmd-2`, etc are reserved for opening the variety of drawers. I prefer jumping to tabs.
 * Can get lost in the preference. For example, Rust plugin related entries are found in at least two spots.
 * Plugin cannot jump to lines based on backtrace
 * Keyboard shortcuts could be more ergonomic but I've gotten used to them

In any case
---

The usual suspects of code completion, attractive rendering, and mostly-zippy interactions are great. As with most kitchen sink IDEs: you're best served on a fast machine. I hope you'll try the Rust integration.

