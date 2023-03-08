+++
title = "p1: Printing to the console."
summary = "Bring back printline-debugging (featuring an overview of memory-mapped I/O)."
date = 2023-03-07T00:00:00-06:00
draft = true
slug = "p1-printing-to-the-console"
+++

## 📝 Overview

The goal of this project is to create a safe abstraction for printing characters out to the console and to initialize a kernel heap for dynamic allocation.

## 🖨️ Printing to the console

Adding print statements is an extremely useful technique for quickly identifying bugs and letting the world know what is happening inside our kernel.

The CPU typically runs much faster than serial communication protocols, sending bursts of characters at the same time (during one print statement, for example) with long idle periods in between.

In hardware, there is typically a universal asynchronous receiver/transmitter (UART) chip which implements **serial** communication with the outside world. So how do we talk to the UART circuitry?

You could imagine it might be nice for the hardware to have a special instruction, say, `mv-to-console`, which took a register argument and just sent the value of that register out to the console. Alas, then we would also want to add `mv-to-screen` when it comes time to display images, and `mv-to-sound` for sound cards, and so on. We would need to keep adding instructions for each device, which is not very scalable and limits our invent new devices that are compatible with old processors.

Instead, maybe we could have an instruction called `iomv` that takes two registers: one specifies the device to send data to, and the other actually contains the data to send. That solves our scalability problem, but we have still increased the complexity of our ISA. (This is how i386 handles some I/O.)

Modern architectures instead use an abstraction called **memory-mapped I/O** (**mmio**). Whenever a core issues an instruction that interacts with memory (either load or store), there is special circuitry that determines what **device** the target address actually refers to. Each device can claim a range of addresses so it can implement different features depending on what address within its range is used.

The **memory map** is platform-dependent and typically specified in a manual. We've actually already investigated this in p0: DRAM is just another device in the memory map! QEMU so helpfully initialises the memory map of our virtual machine [here](https://github.com/qemu/qemu/blob/master/hw/riscv/virt.c#L77). We're interested in the `VIRT_UART0` device, which apparently starts at `0x10000000` and goes for `0x100` bytes. We'll see later that it is possible to programmatically discover these mappings, if a bit tedious.

QEMU supports the NS16550A UART device, so we can consult some [online resources](https://www.lammertbies.nl/comm/info/serial-uart) to figure out what addresses within the mapped range do interesting things. This is extremely low-level and not very relevant to operating systems, so we'll fast forward and just configure it reasonably:

```rust

extern "C" fn entry() -> ! {
  {
    use core::ptr::write_volatile;
    let addr = 0x1000_0000 as *mut u8;
    // Set data size to 8 bits.
    unsafe { write_volatile(addr.offset(3), 0b11) };
    // Enable FIFO.
    unsafe { write_volatile(addr.offset(2), 0b1) };
    // Enable receiver buffer interrupts.
    unsafe { write_volatile(addr.offset(1), 0b1) };
  }

  // UART is now set up! Let's print a message.
  for byte in "Hello, world!\n".bytes() {
    unsafe { write_volatile(addr, byte) };
  }

  loop {}
}
```

We use `core::ptr::write_volatile` to tell the compiler it *must not* re-order or elide any of our loads/stores. Otherwise, the compiler is allowed to be clever about memory operations (for example, look what happens [here](https://godbolt.org/z/n4hf7TWY6)), which is not what we want when interacting with I/O devices.

We also need to tell QEMU to actually emulate the serial device by adding `-serial mon:stdio` to the `runner` option in `.cargo/config.toml`. Now, running the kernel with `cargo run` should cause our special message to get printed in the QEMU console!

### A safe abstraction for UART

We've been using `unsafe` a lot in our code, and that's not necessarily bad, but we need to be careful to justify why our unsafe code will not lead to undefined behaviour. For code that is often repeated (with the same justification), it makes a lot of sense to refactor the unsafe code out into its own library, and provide safe abstraction around it. The abstraction is then responsible for making sure any special conditions required by the unsafe code are satisfied.

Let's create a new **module** for our UART driver. In `main.rs`, add

```rust
// src/main.rs
// ...

mod uart;
```

Then create a new file called `src/uart.rs`:

```rust
// src/uart.rs

//! This module provides access to the UART console.

/// Represents an initialised UART device.
pub struct Device {
  base: usize
}
```

`uart::Device` will be our safe abstraction: as long as the `base` field is valid, then methods on `Device` will be safe.

```rust
// ...

impl Device {
  /// Create a new UART device.
  /// # Safety
  /// `base` must be the base address of a UART device.
  pub unsafe fn new(base: usize) -> Self {
    use core::ptr::write_volatile;
    let addr = base as *mut u8;
    // Set data size to 8 bits.
    unsafe { write_volatile(addr.offset(3), 0b11) };
    // Enable FIFO.
    unsafe { write_volatile(addr.offset(2), 0b1) };
    // Enable receiver buffer interrupts.
    unsafe { write_volatile(addr.offset(1), 0b1) };
    // Return a new, initialised UART device.
    Device { base }
  }

  pub fn put(&self, character: u8) {
    let ptr = self.base as *mut u8;
    // UNSAFE: fine as long as self.base is valid
    unsafe { core::ptr::write_volatile(ptr, character); }
  }
}
```

This might seem dangerous because the caller of `Device::put` has no idea that there might be unsafe things happening inside, but the only way for someone outside of the `uart` module to create a new `Device` is by calling `Device::new`, which explicitly states the requirements that the caller should verify to avoid undefined behavior. So we will still need to use `unsafe` once to obtain an initialised UART device, but not every time we want to write a character.

It is generally a good practice to leave a comment next to `unsafe {}` code blocks explaining what invariants the block might violate and why it should not lead to undefined behaviour. Likewise, unsafe functions should explain what invariants the caller should uphold to avoid undefined behaviour.

Our code for `entry` can be greatly simplified now:

```rust
extern "C" fn entry() -> ! {
  // UNSAFE: correct address for QEMU virt device
  let console = unsafe { uart::Device::new(0x1000_0000) };
  for byte in "Hello, world!".bytes() {
    console.put(byte);
  }
  loop {}
}
```

We'd still like to be able to use `println!` (the Rust macro for printing) with our heap. To do so, we'll need to create a static global variable, called `CONSOLE`, and a macro to write to it.

Add the following to the UART driver:

```rust
// src/uart.rs

// ...

static CONSOLE: Spinlock<Option<Device>> = Spinlock::new(None);

/// Initialise the UART debugging console.
/// # Safety
/// `base` must point to the base address of a UART device.
pub unsafe fn init_console(base: usize) {
  let mut console = CONSOLE.lock();
  *console = Some(unsafe { Device::new(base)} );
}

/// Prints a formatted string to the [CONSOLE].
#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ({
    {
        use core::fmt::Write;
        $crate::uart::CONSOLE.lock().as_mut().map(|writer| {
            writer.write_fmt(format_args!($($arg)*)).unwrap()
        });
    }
    });
}

/// println prints a formatted string to the [CONSOLE] with a trailing newline character.
#[macro_export]
macro_rules! println {
    ($fmt:expr) => ($crate::print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => ($crate::print!(concat!($fmt, "\n"), $($arg)*));
}
```

The macro syntax is magic and totally unimportant for now; the only important part is that we need to implement the `core::fmt::Write` trait for `Device` to be able to use the `concat!` macro, which is provided by the built-in `core` crate.

```rust
// src/uart.rs

// ...

impl core::fmt::Write for Device {
  fn write_str(&mut self, s: &str) -> core::fmt::Result {
    for c in s.bytes() {
        self.put(c);
    }
    Ok(())
  }
}
```

// TODO: explain spinlock and add spinning_top to the dependencies
// TODO: call `uart::init_console` from `entry`

## 📦 Initialising the kernel heap

The kernel, like any complex piece of software, might eventually want to dynamically allocate memory. To do so, we need to set up a heap, and inform Rust to use it.

We assume familiarity with dynamic allocation (`malloc`/`free`), so we'll go ahead and use a library for our implementation. Modify the package manifest to add:

```toml
# below [dependencies]

linked_list_allocator = "0.10.5"
```

and run `cargo check` to update the dependency. Next, add a new module for the heap:

```rust
// src/main.rs

// ...
mod heap;
```

and in a new file:

```
// src/heap.rs
//! Provides the kernel heap.

use linked_list_allocator::LockedHeap;

#[global_allocator]
static ALLOCATOR: LockedHeap = LockedHeap::empty();
```

The `#[global_allocator]` attribute tells Rust that this is the heap it should use for anything that requires dynamic allocation. Having a global allocator isn't strictly necessary, but it gives us access to the built-in `alloc` crate, which provides many useful data structures.

Before we can use the heap, we need to initialise it. But how do we know what region of memory is safe to use? Let the linker tell us!

```
# src/script.ld

SECTIONS {
  # ...
  PROVIDE(_kernel_heap_bottom = _init_stack_top);
  PROVIDE(_kernel_heap_top = ORIGIN(ram) + LENGTH(ram));
  PROVIDE(_kernel_heap_size = _kernel_heap_top - _kernel_heap_bottom);
}
```

This will create new symbols which refer to the entire region of physical memory after the stack. (We might want to shrink the kernel heap later, but for now this is perfectly fine.)

In `heap.rs`, we'll declare some static variables and an initialisation routine:

```rust
// src/heap.rs

/// Initialise the kernel heap.
/// # Safety
/// Must be called at most once.
pub unsafe fn init() {
    // access the linker values from Rust
    extern "C" {
        static _kernel_heap_bottom: *mut u8;
        static _kernel_heap_size: usize;
    }

    // note that interacting with static variables is inherently unsafe, but that's OK!
    let heap_bottom = unsafe { _kernel_heap_bottom };
    let heap_size = unsafe { _kernel_heap_size };

    // SAFETY: Fine to call at most once.
    unsafe { ALLOCATOR.lock().init(heap_bottom, heap_size) };
}
```

You should take a look at the [documentation](https://docs.rs/linked_list_allocator/0.10.5/linked_list_allocator/struct.Heap.html#method.init) for `linked_list_allocator::Heap::init` to see why our initialisation call does not cause undefined behavior.

Now we just need to call `heap::init` from `entry`:

```rust
// src/main.rs

// ...

extern "C" fn entry() -> ! {
  // ...
  
  // UNSAFE: Called exactly once, right here.
  unsafe { heap::init() };

  loop {}
}
```