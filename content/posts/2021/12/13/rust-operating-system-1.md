+++
title = "Writing a RISC-V Operating System in Rust"
summary = "My toxic trait is thinking that I can write an operating system in a language I'm still learning for an architecture I've never studied. How far can we go?"
date = 2021-12-13T14:55:13-06:00
draft = false
slug = "rust-operating-system-1"
+++

Welcome to a new series of blog posts where you can read (and follow) along as I slowly chip away at my sanity to build an operating system kernel for RISC-V in Rust.

I'll start by tackling some of your most frequent questions:

1. *Q: Why?*\
   *A:*  The operating system kernel is a lot like Pandora's box. [metaphor for opening the secrets]
2. *Q: Why not x86 or ARM?*\
   *A:* x86 is the absolute worst (it boots into 16-bit mode! In the year 2021!!!). I would rather take an artificial intelligence class than write another x86 kernel. As for ARM, I found it more difficult to find good, public, free resources documenting the architecture.
3. *Q: Why not C/C++?*\
   *A:* C is a nightmare and C++ is five nightmares in one. (And after taking computer architecture and operating systems, I've lived all six of them together.)

## Resources and citations

* Dr. Ahmed Gheith's provided starter code for CS 439H (honors operating systems).
* [blog_os](https://github.com/phil-opp/blog_os) by Philipp Oppermann et al.
* [core-os-riscv](https://github.com/skyzh/core-os-riscv) by Alex Chi.
* ["The Adventures of OS: Making a RISC-V Operating System using Rust"](http://osblog.stephenmarz.com) by Stephen Marz.

