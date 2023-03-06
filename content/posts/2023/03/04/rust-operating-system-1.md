+++
title = "Writing a RISC-V operating system kernel in Rust"
summary = "Introducing a new series of blog posts where we build an operating system kernel from the ground up."
date = 2023-03-04T00:00:00-06:00
draft = false
slug = "rust-operating-system-intro"
+++

Welcome to a new series of blog posts where you can read (and follow) along as I slowly chip away at my sanity to build an operating system kernel for RISC-V in Rust. The target audience will have completed Dr. Ahmed Gheith's CS 429H Computer Architecture and be preparing to take CS 439H (operating systems) the next semester.

Our kernel will run on machines that support the RV32IMAC instruction set architecture. That’s RISC-V, 32-bit word-size, with integer instructions, multiplication, atomic instructions, and compressed instructions. For now, don’t worry too much about these details: we’re targeting basically the lowest common denominator of realistic (non-toy) RISC-V processors. From now on, we’ll say RISC-V for short (similar to how ARM or x86 refer to many processors with similar architectures).

RISC-V defines the set of instructions the processor must support, but that’s really only important for the compiler — we’re only rarely going to be directly writing assembly. The kernel is the part of the operating system that directly interfaces with low-level hardware, and the details there can vary widely between physical devices (in fact, the Linux kernel has probably thousands of files dedicated to supporting different platforms with the exact same processor).

In particular, we’re going to focus on supporting the QEMU “virt” device, which is roughly based on SiFive’s platform. QEMU is not very well documented, but we can generally consult the SiFive manuals for details. If that fails, we’ll turn to the QEMU source code.

Finally, the majority of the kernel will be written in Rust. This makes less of a difference than you might expect because kernel software does a ***lot*** of low-level, unsafe stuff that is impossible for the compiler to prove correct. Rust basically functions like a more ergonomic C++ in this context. We could also write the kernel in plain C, like the Linux or macOS kernel, or even directly in assembly, but it would be painful to part with the niceties of a higher-level language.

## Posts

* [p0: Running Rust code on RISC-V in QEMU]({{< ref "/posts/2023/03/05/rust-operating-system-2" >}})

## Resources and inspiration

* Dr. Ahmed Gheith's provided starter code for CS 439H (honors operating systems).
* [blog_os](https://github.com/phil-opp/blog_os) by Philipp Oppermann et al.
* [core-os-riscv](https://github.com/skyzh/core-os-riscv) by Alex Chi.
* ["The Adventures of OS: Making a RISC-V Operating System using Rust"](http://osblog.stephenmarz.com) by Stephen Marz.
* [RISC-V from scratch 2: Hardware layouts, linker scripts, and C runtimes](https://twilco.github.io/riscv-from-scratch/2019/04/27/riscv-from-scratch-2.html#lifting-the--veil) by Tyler Wilcock
