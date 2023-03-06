+++
title = "p0: Running Rust code on RISC-V in QEMU"
summary = "The goal for this project is to run some code on a RISC-V virtual machine without an operating system, and start learning how to debug it."
date = 2023-03-05T00:00:00-06:00
draft = false
slug = "running-rust-code-on-risc-v-in-qemu"
toc = true
+++

## 📝 Overview

The goal for this project is to run some code on a RISC-V virtual machine and start learning how to debug it. Key terms are **bolded**, and you may want to search online for more information about them.

## 🛠️ Setting up the toolchain

The Rust standard toolchain already supports RISC-V, so we can use `rustc` to cross-compile. In a command prompt, install the nightly toolchain of Rust and add RISC-V as a target.

```bash
# install Rust - https://rustup.rs
$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# install nightly toolchain and make a new rust project in ./p0
$ rustup run nightly cargo new --name kernel p0

$ cd p0
# from now on, all paths and commands will be relative to the root of p0

# set the nightly toolchain as the default for this project
$ rustup override set nightly

$ tree
.
├── Cargo.toml
└── src
    └── main.rs

$ cat Cargo.toml
[package]
name = "kernel"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]

$ cat src/main.rs
fn main() {
    println!("Hello, world!");
}
```

Cargo is the Rust package manager and helpfully provides wrappers around most of the Rust toolchain, so `cargo` will be your main point of contact with Rust. A Rust **package** (described by a `Cargo.toml` **manifest** file) is a collection of files that provide one or more **crates**. A crate is either a library or an executable program, referred to as either a **library crate** or a **binary crate**, respectively. A **target** is a platform that you want your code to run on. Crates are usually inferred automatically based on the layout of the project — e.g., if there is a file called `src/main.rs` , Cargo assumes you wanted to add a binary crate using `main.rs` and all of its dependencies.

This can be confusing, and it gets worse when we introduce modules, so it’s worth spending some time playing around with `cargo` to understand the difference between ***packages***, ***targets***, and ***crates***.

## ⚙️ Compiling a program for RISC-V

Let’s try compiling the default binary target.

```bash
$ cargo build
   Compiling kernel v0.1.0 (.../p0)
    Finished dev [unoptimized + debuginfo] target(s) in 0.15s
```

Well, that works! I guess we’re done.

Let’s take a look at the binary executable Rust outputted at `target/debug/kernel`:

```bash
$ file target/debug/kernel
target/debug/kernel: Mach-O 64-bit executable arm64
```

Oops. Your output might be different, but unless you are running on a 32-bit RISC-V machine, it is unlikely to be RISC-V. You may have noticed the package manifest never mentions RISC-V. Rust assumes that every crate can be compiled for every target. That’s true for most crates, but not for an operating system kernel, so we need to tell the Rust toolchain that it should build a binary for a different target than the one the compiler is running on (a process called ***cross-compiling***).

We need to create an additional configuration file, called `.cargo/config.toml` (note the leading dot), which tells `cargo` how to compile and run for each target. These options are separate because they can change depending on your development environment. Let’s go ahead and create that file:

```toml
# .cargo/config.toml

[build]
target = "riscv32imac-unknown-none-elf"
```

We’re telling the Rust compiler to target rv32imac, as advertised. The value of `target` is commonly referred to as the ******[triple](https://doc.rust-lang.org/nightly/rustc/platform-support.html)******, though the astute reader may notice that the triple does not always contain three values. Our triple mentions ELF: that will be important later.

So now we should be good to go! Right?

```bash
$ cargo run
Compiling kernel v0.1.0 (.../p0)
error[E0463]: can't find crate for `std`
  |
  = note: the `riscv32imac-unknown-none-elf` target may not support the standard library
  = note: `std` is required by `kernel` because it does not declare `#![no_std]`
  = help: consider building the standard library from source with `cargo build -Zbuild-std`

# ... more errors, but we will tackle them one at a time
```

Ah, right. Rust programs by default are compiled together with the Rust standard library (`std`). Rust doesn’t provide a pre-compiled `std` crate for RISC-V, and even if it did, `std` uses many functionalities that are dependent on the operating system (e.g. `std::thread`). We can’t use the operating system to implement the operating system!

We’ll have to modify our `src/main.rs` to tell Rust we don’t want the standard library. Just like the compiler suggested, we can add `#![no_std]` to the top. `#![no_std]` is a crate-level *********attribute********* which changes the way the crate is compiled. In general, you will see `#` used to denote compiler options (similar to its use for preprocessor directives in C, but much more limited).

We’ll also have to remove `println!` (for now) because it is defined by the standard library. Let’s replace it with a simple loop for now.

```rust
// src/main.rs

#![no_std]

fn main() {
	loop {}
}
```

And if we try running again

```bash
$ cargo run

Compiling kernel v0.1.0 (.../p0)
error: `#[panic_handler]` function required, but not found

error: could not compile `kernel` due to previous error
```

We need a panic handler! This is the function that gets called when the program invokes the `panic!` macro. The standard library typically provides one, but since we’ve opted out of the standard library, we will need to provide our own.

```rust
#![no_std]

// "import" PanicInfo from [core]
use core::panic::PanicInfo;

fn main() {
	loop {}
}

#[panic_handler]
fn on_panic(info: &PanicInfo) -> ! {
	loop {}
}
```

Let’s discuss the `on_panic` function’s signature.

- `#[panic_handler]` is an *********attribute********* — an instruction to the compiler — in this case, to call `on_panic` when any code uses the `panic!` macro.
- `info` is its only argument. The type of `info` is an immutable reference to a `PanicInfo` struct.
- `-> !` means `on_panic` never returns.

We also had to add a `use` statement to make `PanicInfo` available in this scope (the whole file). Alternatively, we could also have fully-qualified the type of `info: &core::panic::PanicInfo`.

Ok, back to the compiler:

```bash
$ cargo run
Compiling kernel v0.1.0 (.../p0)
error: requires `start` lang_item

error: could not compile `kernel` due to previous error
```

We’re missing another thing: somewhere for the program to start! It turns out that `main` is not actually the entry point for most programs — if you compile a program with `std`, there is a (very small) amount of setup that needs to happen before `main`. The real, true, actual entry point is typically called `_start` (or some variation of that) and is usually provided by the standard library or compiler for whatever language you are using. In C, this is usually [`crt0`](https://en.wikipedia.org/wiki/Crt0).

In Rust (and most higher-level languages), the compiler changes (”mangles”) the name of every function to make it unique across the whole crate. Let’s rename `main` to `_start` and tell the Rust compiler not to mangle the name — there really should only be one `_start` in the crate!

```rust
#[no_mangle]
fn _start() {
	loop {}
}
```

Then the compiler will complain that there’s no main function. As it recommends, we can fix that by adding `#![no_main]` to the top of the file.

Finally, if we run `cargo build` everything should succeed.

```bash
$ cargo build
   Compiling kernel v0.1.0 (.../p0)
    Finished dev [unoptimized + debuginfo] target(s) in 0.17s

$ file target/riscv32imac-unknown-none-elf/debug/kernel

target/riscv32imac-unknown-none-elf/debug/kernel: ELF 32-bit LSB executable, 
UCB RISC-V, RVC, soft-float ABI, version 1 (SYSV), statically linked, 
with debug_info, not stripped
```

Nice! There’s a lot of information here, but the important parts are `ELF 32-bit`, `RISCV`, and `statically linked`. We’ve successfully compiled a freestanding Rust executable for our target architecture.

## 🏃 Running the program

If we try running now, we’ll get a different output:

```bash
$ cargo run
target/riscv32imac-unknown-none-elf/debug/kernel: cannot execute binary file
```

Oops again. We’re (probably) not working on a RISC-V machine, so we can’t even run the program directly. We need to *******emulate******* a RISC-V machine, and to do so we turn to our friend, QEMU.

Unlike other virtualisation softwares you might be familiar with (Docker), QEMU is a whole-system emulation framework. Amazingly, it is actually decently performant (and more than fast enough for our use-case). It is not a physical *simulation* — it does not pretend to run circuits — but it gives us the illusion of running code on “bare-metal” hardware while still allowing us to debug comfortably from our local computer.

We need to tell Cargo to run our program using QEMU. Add a `runner` to your `.cargo/config.toml` file:

```toml
# .cargo/config.toml

# ... from before

[target.riscv32imac-unknown-none-elf]
runner = """ qemu-system-riscv32
  -cpu rv32
  -machine virt
  -m 150M
  -s
  -nographic
  -bios """
```

We’ll add more flags as our kernel becomes more complete, but we’ll start with the basics:

- `-cpu rv32` means QEMU will emulate a 32-bit processor.
- `-machine virt` tells QEMU which platform to emulate.
- `-m 150M` means we want to emulate 150MB of physical RAM.
- `-s` enables remote debugging (we’ll see why this is useful shortly).
- `-nographic` disables the QEMU graphic display (we might eventually add VGA output, at which point we’ll need to remove this to see what’s happening).
- `-bios` tells QEMU to skip the **bootloader** and just run our program directly on startup (as if it were a BIOS). We could also use `-kernel` to have QEMU run a full bootloader (which a real operating system like Linux would require), but that process is more complicated and unnecessary as we’re just getting started.

```bash
$ cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `qemu-system-riscv32 -cpu rv32 -machine virt -m 150M -serial 'mon:stdio' -device virtio-rng-device -bios target/riscv32imac-unknown-none-elf/debug/kernel`
```

… and then it hangs. Did it work? How do we know what’s happening?

### 💡 What happens when the system starts

Boom. The switch is flipped. Current is flowing through the processor. What happens? The (full) answer is complicated and dives deep into microarchitectural components that aren’t super interesting from an operating systems perspective, so we’ll refine our question: when the processor starts execution, what is its initial state?

The answer depends on the platform, but for a virt machine the program counter is initially set to point to the **reset vector** (by default, address `0x1000`). QEMU pretends like the bootloader loads our program into memory (notice that we haven’t actually configured a hard drive yet, so this isn’t quite realistic, but it’s a useful simplification for now) and then jumps to the start of physical memory at `0x8000_0000`.

But don’t take my word for it. We can use `gdb` (the GNU debugger) to step through the process one instruction at a time!

You’ll need to add `-S` (capitalised) to the QEMU run configuration to make QEMU wait to run until the debugger is connected. You can do this from the `cargo run` command:

```bash
$ cargo run -- -S
   Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `qemu-system-riscv32 -cpu rv32 -machine virt -m 150M -serial 'mon:stdio' -device virtio-rng-device -bios target/riscv32imac-unknown-none-elf/debug/kernel -S`
```

Everything after `--` gets added to the `runner` command, so we just appended `-S` right after the `-bios [target]` flag. The output looks the same, but QEMU is actually paused, waiting for us to debug it. 

Next, we’ll open `gdb` in another terminal:

```bash
$ gdb
...
$ (gdb) file target/riscv32imac-unknown-none-elf/debug/kernel
Reading symbols from target/riscv32imac-unknown-none-elf/debug/kernel...
$ (gdb) target remote localhost:1234
Remote debugging using localhost:1234
0x00001000 in ?? ()
```

We’re in the debugger now, connected to QEMU, and we can see the program counter. We can print any registers we want, look at any place in memory, etc.

Let’s ask QEMU to print the code in memory wherever we are.

```bash
$ (gdb) x/6i $pc
=> 0x1000:      auipc   t0,0x0
   0x1004:      addi    a2,t0,40
   0x1008:      csrr    a0,mhartid
   0x100c:      lw      a1,32(t0)
   0x1010:      lw      t0,24(t0)
   0x1014:      jr      t0
```

So we’re loading some values, then jumping to the address in `t0`. Let’s step forward until we are about to jump:

```bash
(gdb) x/6xi 0x1000
=> 0x1000:	auipc	t0,0x0
   0x1004:	addi	a2,t0,40
   0x1008:	csrr	a0,mhartid
   0x100c:	lw	a1,32(t0)
   0x1010:	lw	t0,24(t0)
   0x1014:	jr	t0
(gdb) si
0x00001004 in ?? ()
(gdb) si
0x00001008 in ?? ()
(gdb) si
0x0000100c in ?? ()
(gdb) si
0x00001010 in ?? ()
(gdb) si
0x00001014 in ?? ()
(gdb) x/i $pc
=> 0x1014:	jr	t0
(gdb) p/x $t0
$1 = 0x80000000
```

Great, so we’re jumping to `0x80000000`. And if we look at the instructions there, we should find our kernel!

```bash
(gdb) x/10i $t0
   0x80000000:  unimp
   0x80000002:  unimp
   0x80000004:  unimp
   0x80000006:  unimp
   0x80000008:  unimp
   0x8000000a:  unimp
   0x8000000c:  unimp
   0x8000000e:  unimp
   0x80000010:  unimp
   0x80000012:  unimp
```

Whelp… (you can use `q` to exit GDB, and `C-a x` to exit QEMU).

### 🔗 Linking the program

```bash
(gdb) si
0x00001004 in ?? ()
(gdb) si
0x00001008 in ?? ()
(gdb) si
0x0000100c in ?? ()
(gdb) si
0x00001010 in ?? ()
(gdb) si
0x00001014 in ?? ()
(gdb) x/i $pc
=> 0x1014:      jr      t0
```

What went wrong?

So far, we’ve assumed that the standard compiler and linker settings will Just Work for our kernel.  Clearly we’ll need to put in a bit more effort to make it work. Let’s start by understanding the problem.

How does QEMU know where the various parts of our kernel program should live in memory? We’ve discovered that `_start` isn’t loaded at `0x8000_0000` and we started running some random garbage instead.

**ELF**, or Executable and Linkable Format, is a highly flexible type of file that encodes all of a program's data. This includes the code required for execution, static and constant variables, references to libraries, and addressing information. We'll explore ELF in more detail later, but for now, it's enough to know that the output of the linker for a RISC-V binary target is an ELF file that QEMU reads to understand the kernel's memory layout. In a non-virtualized system, a bootloader in ROM would handle this at runtime, so by the time execution reaches our kernel, it has been loaded into memory.

Let’s poke around our kernel’s ELF file and see what’s happening. To do so, we’ll use `readelf`, a very useful program with a ton of functionality and options.

```bash
$ readelf -l target/riscv32imac-unknown-none-elf/debug/kernel

Elf file type is EXEC (Executable file)
Entry point 0x110b4
There are 4 program headers, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  PHDR           0x000034 0x00010034 0x00010034 0x00080 0x00080 R   0x4
  LOAD           0x000000 0x00010000 0x00010000 0x000b4 0x000b4 R   0x1000
  LOAD           0x0000b4 0x000110b4 0x000110b4 0x00004 0x00004 R E 0x1000
  GNU_STACK      0x000000 0x00000000 0x00000000 0x00000 0x00000 RW  0

 Section to Segment mapping:
  Segment Sections...
   00     
   01     
   02     .text 
   03
```

If we look at the `PhysAddr` column, everything is positioned at pretty small addresses. We know that’s not correct. Because we have some very specific requirements for memory layout, we need to write a **************************linker script************************** that specifies where everything should be. Linker scripts can be intimidating, so let’s step through one together. Create a file called `src/script.ld` and add:

```
# src/script.ld

OUTPUT_ARCH("riscv")
ENTRY(_start)
```

We’re linking for RISC-V, and the program’s entry point is `_start`.

```
# ...

MEMORY {
  ram   (wxa) : ORIGIN = 0x80000000, LENGTH = 128M
}
```

There’s one region of memory (as far as the linker is concerned). It starts at `0x8000_0000` and is 128MB long. Notice that this is less than the amount we requested from QEMU; that’s OK! And `wxa` means RAM can be written, executed, and accessed once the program is loaded.

```
# ...

PHDRS {
  text PT_LOAD;
  data PT_LOAD;
  bss PT_LOAD;
}
```

`PHDRS` stands for “program headers.” There are three kinds of headers of our program: `text` is code, `data` is global variables and constant values (like strings), and `bss` is any global value that is initialised to zero. (Having a `bss` is a common optimisation since programs typically have a lot of initially-zero values). These program headers (also called segments) are the chunks that are actually loaded into memory at runtime. The compiler will choose to put every ****************section**************** of data into one of these ****************segments****************. We need to tell the linker the relationship between sections and segments:

```
# ...

SECTIONS {
  . = ORIGIN(ram); # start at 0x8000_0000

  .text : { # put code first
    *(.text.init) # start with anything in the .text.init section
    *(.text .text.*) # then put anything else in .text
  } >ram AT>ram :text # put this section into the text segment

  PROVIDE(_global_pointer = .); # this is magic, google "linker relaxation"

  .rodata : { # next, read-only data
    *(.rodata .rodata.*)
  } >ram AT>ram :text # goes into the text segment as well (since instructions are generally read-only)

  .data : { # and the data section
    *(.sdata .sdata.*) *(.data .data.*)
  } >ram AT>ram :data # this will go into the data segment

  .bss :{ # finally, the BSS
    PROVIDE(_bss_start = .); # define a variable for the start of this section
    *(.sbss .sbss.*) *(.bss .bss.*)
    PROVIDE(_bss_end = .); # ... and one at the end
  } >ram AT>ram :bss # and this goes into the bss segment

}
```

There’s a lot of magic here and you should take some time to play around with it and understand the effect this has.

Next, we need to tell the compiler to use this linker script. We could modify the `.cargo/config.toml` file, or we can write a `build.rs` script (the details of both are not super important). We’ll be doing the latter. Create a new file in the root directory (not in `./src`!) called `build.rs`:

```rust
// build.rs

fn main() {
  // Use the linker script.
  println!("cargo:rustc-link-arg=-Tsrc/script.ld");
  // Don't do any magic linker stuff.
  println!("cargo:rustc-link-arg=--omagic");
}
```

Now let’s look at the ELF file:

```bash
$ cargo build
   Compiling kernel v0.1.0 (.../p0)
    Finished dev [unoptimized + debuginfo] target(s) in 0.19s
$ readelf --segments target/riscv32imac-unknown-none-elf/debug/kernel
Elf file type is EXEC (Executable file)
Entry point 0x80000000
There is 1 program header, starting at offset 52

Program Headers:
  Type           Offset   VirtAddr   PhysAddr   FileSiz MemSiz  Flg Align
  LOAD           0x000094 0x80000000 0x80000000 0x00004 0x00004 R E 0x2

 Section to Segment mapping:
  Segment Sections...
   00     .text
```

We only have one segment at the moment because our Rust program is so simple, it has no global variables or constants. Just one function, which does nothing — hence one `ret` statement which takes 4 bytes. So it looks like the linker is working!

Now, if we run QEMU and GDB it again, we should see something different.

```bash
$ gdb target/riscv32imac-unknown-none-elf/debug/kernel
...
Reading symbols from target/riscv32imac-unknown-none-elf/debug/kernel...
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
0x00001000 in ?? ()
(gdb) si
0x00001004 in ?? ()
(gdb) si
0x00001008 in ?? ()
(gdb) si
0x0000100c in ?? ()
(gdb) si
0x00001010 in ?? ()
(gdb) si
0x00001014 in ?? ()
(gdb) si
kernel::_start () at src/main.rs:10
10          loop {}
(gdb) si
0x80000002      10          loop {}
(gdb) si
0x80000002      10          loop {}
(gdb) si
0x80000002      10          loop {}
```

Nice! So the debugger is showing us that we are running our Rust code! Mission accomplished!

### 👩‍🔧 Fixing some mistakes

There are two (subtle) correctness issue with our implementation so far. When the bootloader jumps to `0x8000_0000`, it is essentially making a function call in the standard C calling convention (see the RISC-V manual chapter on “Calling Convention”). Rust uses a different (and currently, not formally specified) calling convention. We got lucky that for a very simple function like `_start`, they appear to generate the same code, but we need to tell the compiler that `_start` should be callable using the C calling convention before we add much else. Change the signature of `_start` to:

```rust
extern "C" fn _start() -> ! {
```

This means that `_start` is callable from external C code, and that it should never return.

<details>
<summary>EXERCISE: Run the program in GDB and set a breakpoint on `_start`. What is the value of the `sp` register at that point? (Click for answer.)</summary>

```
$ gdb target/riscv32imac-unknown-none-elf/debug/kernel
Reading symbols from target/riscv32imac-unknown-none-elf/debug/kernel...
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
0x00001000 in ?? ()
(gdb) break _start
Breakpoint 1 at 0x80000000: file src/main.rs, line 16.
(gdb) continue
Continuing.

Breakpoint 1, kernel::_start () at src/main.rs:16
16	    asm!(
(gdb) p/x $sp
$1 = 0x0
```
</details></p>

<details>    
<summary>EXERCISE: What happens if you attempt to dereference `0x0` in GDB? What about `0x9000_0000`? Why? (Click for answer.)</summary>

```bash
(gdb) x 0x90000000
0x90000000:	Cannot access memory at address 0x90000000
(gdb) x 0x0
0x0:	Cannot access memory at address 0x0
```
</details></p>

Second, we don’t have a stack! That’s bad, because the C calling convention requires `sp` point to a valid (and aligned) stack. So before we can enter proper `C` code, we need to set up a stack.

While we’re setting up, we only need a relatively small initialisation stack. Once we have some fancier data structures and a better understanding of the runtime environment, we can migrate to a bigger stack somewhere else.

We can ask the linker to reserve a small amount of space separate from the code or data for our stack. Add the following to `src/script.ld`:

```
SECTIONS {
	# ... everything from before
  
	# . is now at the end of all code/data
  . = ALIGN(16) # our stack needs to be 16-byte aligned, per the C calling convention
  PROVIDE(_init_stack_top = . + 0x1000) # reserve 0x1000 bytes for the initialisation stack
}
```

The linker will now provide a symbol, `_init_stack_top`, which we can use to refer to the initialisation stack. Let’s use this to set our `sp` as soon as we enter `_start`. We’ll have to drop into assembly to mess with registers:

```rust
// src/main.rs

#[no_mangle]
extern "C" fn _start() {
    use core::arch::asm;
    asm!("la sp, _init_stack_top");

    loop {}
}
```

`asm!` tells the Rust compiler to emit one (or more) assembly instructions in place. It is extremely powerful, but more restrictive than plain assembly, and we sometimes need to tell Rust what kinds of things our assembly can do. You can refer to [https://doc.rust-lang.org/reference/inline-assembly.html](https://doc.rust-lang.org/reference/inline-assembly.html) for specifics.

If we try to compile, we get an error:

```bash
$ cargo build
   Compiling kernel v0.1.0 (.../p0)
error[E0133]: use of inline assembly is unsafe and requires unsafe function or block
 --> src/main.rs:9:5
  |
9 |     asm!("la sp, _init_stack_top");
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ use of inline assembly
  |
  = note: inline assembly is entirely unchecked and can cause undefined behavior

For more information about this error, try `rustc --explain E0133`.
```

We’ve reached our first `unsafe`! I’ll spare you the extended discussion of Undefined Behavior ™️ (a favourite topic of Rustaceans) and give you some suggested reading:

- [https://doc.rust-lang.org/nomicon/meet-safe-and-unsafe.html](https://doc.rust-lang.org/nomicon/meet-safe-and-unsafe.html)
- [https://doc.rust-lang.org/nomicon/what-unsafe-does.html](https://doc.rust-lang.org/nomicon/what-unsafe-does.html)
- [https://doc.rust-lang.org/nomicon/working-with-unsafe.html](https://doc.rust-lang.org/nomicon/working-with-unsafe.html)

We could add the `unsafe` block around `asm!`, but this still does not solve our violation of the C calling ABI, since `sp` would still not be valid at the moment QEMU jumps to `_start`. The compiler could still try to insert instructions before our `asm!` block to allocate space on the stack.

Instead, we will mark `_start` as an unsafe, naked function and make another function, called `entry`, which is our “real” entry point into Safe Rust. `_start` will do as little work as possible to establish the invariants that the C calling convention requires before jumping to `entry`.

```rust
// src/main.rs

#![feature(naked_functions)] // we need to use a new feature!

// ... from before

#[naked]
#[no_mangle]
unsafe extern "C" fn _start() -> ! {
  use core::arch::asm;
  asm!(
    // before we use the `la` pseudo-instruction for the first time,
    //  we need to set `gp` (google linker relaxation)
    ".option push",
    ".option norelax",
    "la gp, _global_pointer",
    ".option pop",
    
    // set the stack pointer
    "la sp, _init_stack_top",

    // "tail-call" to {entry} (call without saving a return address)
    "tail {entry}",
    entry = sym entry, // {entry} refers to the function [entry] below
    options(noreturn) // we must handle "returning" from assembly
  );
}

extern "C" fn entry() -> ! {
  loop {}
}
```

If you debug the program now, you should see it enter `entry` with a valid stack pointer. Now we’re really done!

## 0️⃣ Clearing the BSS

Before we can use any global variables from Rust, we need to set every byte in the `bss` segment to `0`. But where is the BSS? Our linker script already defines two symbols, `_bss_start` and `_bss_end`! We can do this in Rust, but it’s pretty straightforward to implement in assembly in our `_start` function so we’ll do it there.

<details>
<summary>EXERCISE: write code in assembly to clear the BSS. (Click for answer.)</summary>
    
```rust
// src/main.rs

// ...
    "la sp, _init_stack_top",

    // clear the BSS
    "la t0, _bss_start",
    "la t1, _bss_end",
    "bgeu t0, t1, 2f",
"1:",
    "sb zero, 0(t0)",
    "addi t0, t0, 1",
    "bne t0, t1, 1b",
"2:",
    // BSS is clear!
```

You can verify in GDB that everything between `_bss_start` (inclusive) and `_bss_end` (exclusive) is zero.
</details></p>

Now we’re done! That’s a lot for one blog post, so next time we’ll try to get a message to print to the QEMU console.