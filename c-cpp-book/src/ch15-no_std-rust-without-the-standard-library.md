# `no_std` — 没有标准库的 Rust

> **你将学到什么：** 如何使用 `#![no_std]` 为裸机和嵌入式目标编写 Rust——`core` 和 `alloc` crate 分割、panic 处理程序，以及这与没有 `libc` 的嵌入式 C 的比较。

如果你来自嵌入式 C，你已经习惯于在没有 `libc` 或使用最小运行时的情况下工作。Rust 有一个一等价的：**`#![no_std]`** 属性。

## 什么是 `no_std`？

当你把 `#![no_std]` 添加到 crate 根目录时，编译器会移除隐式的 `extern crate std;`，只链接 **`core`**（和可选的 **`alloc`**）。

| 层 | 它提供什么 | 需要 OS / 堆？ |
|-------|-----------------|---------------------|
| `core` | 原始类型、`Option`、`Result`、`Iterator`、数学、`slice`、`str`、原子操作、`fmt` | **否**——在裸机上运行 |
| `alloc` | `Vec`、`String`、`Box`、`Rc`、`Arc`、`BTreeMap` | 需要全局分配器，但**不需要 OS** |
| `std` | `HashMap`、`fs`、`net`、`thread`、`io`、`env`、`process` | **是**——需要 OS |

> **嵌入式开发者的经验法则：** 如果你的 C 项目链接到 `-lc` 并使用 `malloc`，你可能可以使用 `core` + `alloc`。如果它在没有 `malloc` 的裸机上运行，坚持只使用 `core`。

## 声明 `no_std`

```rust
// src/lib.rs  (or src/main.rs for a binary with #![no_main])
#![no_std]

// You still get everything in `core`:
use core::fmt;
use core::result::Result;
use core::option::Option;

// If you have an allocator, opt in to heap types:
extern crate alloc;
use alloc::vec::Vec;
use alloc::string::String;
```

For a bare-metal binary you also need `#![no_main]` and a panic handler:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {} // hang on panic — replace with your board's reset/LED blink
}

// Entry point depends on your HAL / linker script
```

## 你失去什么（和替代方案）

| `std` 功能 | `no_std` 替代方案 |
|---------------|---------------------|
| `println!` | `core::write!` 到 UART / `defmt` |
| `HashMap` | `heapless::FnvIndexMap` (fixed capacity) or `BTreeMap` (with `alloc`) |
| `Vec` | `heapless::Vec` (stack-allocated, fixed capacity) |
| `String` | `heapless::String` or `&str` |
| `std::io::Read/Write` | `embedded_io::Read/Write` |
| `thread::spawn` | Interrupt handlers, RTIC tasks |
| `std::time` | Hardware timer peripherals |
| `std::fs` | Flash / EEPROM drivers |

## Notable `no_std` crates for embedded

| Crate | Purpose | Notes |
|-------|---------|-------|
| [`heapless`](https://crates.io/crates/heapless) | Fixed-capacity `Vec`, `String`, `Queue`, `Map` | No allocator needed — all on the stack |
| [`defmt`](https://crates.io/crates/defmt) | Efficient logging over probe/ITM | Like `printf` but deferred formatting on the host |
| [`embedded-hal`](https://crates.io/crates/embedded-hal) | Hardware abstraction traits (SPI, I²C, GPIO, UART) | Implement once, run on any MCU |
| [`cortex-m`](https://crates.io/crates/cortex-m) | ARM Cortex-M intrinsics & register access | Low-level, like CMSIS |
| [`cortex-m-rt`](https://crates.io/crates/cortex-m-rt) | Runtime / startup code for Cortex-M | Replaces your `startup.s` |
| [`rtic`](https://crates.io/crates/rtic) | Real-Time Interrupt-driven Concurrency | Compile-time task scheduling, zero overhead |
| [`embassy`](https://crates.io/crates/embassy-executor) | Async executor for embedded | `async/await` on bare metal |
| [`postcard`](https://crates.io/crates/postcard) | `no_std` serde serialization (binary) | Replaces `serde_json` when you can't afford strings |
| [`thiserror`](https://crates.io/crates/thiserror) | Derive macro for `Error` trait | Works in `no_std` since v2; prefer over `anyhow` |
| [`smoltcp`](https://crates.io/crates/smoltcp) | `no_std` TCP/IP stack | When you need networking without an OS |

## C vs Rust: bare-metal comparison

A typical embedded C blinky:

```c
// C — bare metal, vendor HAL
#include "stm32f4xx_hal.h"

void SysTick_Handler(void) {
    HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);
}

int main(void) {
    HAL_Init();
    __HAL_RCC_GPIOA_CLK_ENABLE();
    GPIO_InitTypeDef gpio = { .Pin = GPIO_PIN_5, .Mode = GPIO_MODE_OUTPUT_PP };
    HAL_GPIO_Init(GPIOA, &gpio);
    HAL_SYSTICK_Config(HAL_RCC_GetHCLKFreq() / 1000);
    while (1) {}
}
```

The Rust equivalent (using `embedded-hal` + a board crate):

```rust
#![no_std]
#![no_main]

use cortex_m_rt::entry;
use panic_halt as _; // panic handler: infinite loop
use stm32f4xx_hal::{pac, prelude::*};

#[entry]
fn main() -> ! {
    let dp = pac::Peripherals::take().unwrap();
    let gpioa = dp.GPIOA.split();
    let mut led = gpioa.pa5.into_push_pull_output();

    let rcc = dp.RCC.constrain();
    let clocks = rcc.cfgr.freeze();
    let mut delay = dp.TIM2.delay_ms(&clocks);

    loop {
        led.toggle();
        delay.delay_ms(500u32);
    }
}
```

**Key differences for C devs:**
- `Peripherals::take()` returns `Option` — ensures the singleton pattern at compile time (no double-init bugs)
- `.split()` moves ownership of individual pins — no risk of two modules driving the same pin
- All register access is type-checked — you can't accidentally write to a read-only register
- The borrow checker prevents data races between `main` and interrupt handlers (with RTIC)

## When to use `no_std` vs `std`

```mermaid
flowchart TD
    A[Does your target have an OS?] -->|Yes| B[Use std]
    A -->|No| C[Do you have a heap allocator?]
    C -->|Yes| D["Use #![no_std] + extern crate alloc"]
    C -->|No| E["Use #![no_std] with core only"]
    B --> F[Full Vec, HashMap, threads, fs, net]
    D --> G[Vec, String, Box, BTreeMap — no fs/net/threads]
    E --> H[Fixed-size arrays, heapless collections, no allocation]
```

# Exercise: `no_std` ring buffer

🔴 **Challenge** — combines generics, `MaybeUninit`, and `#[cfg(test)]` in a `no_std` context

In embedded systems you often need a fixed-size ring buffer (circular buffer) that
never allocates.  Implement one using only `core` (no `alloc`, no `std`).

**Requirements:**
- Generic over element type `T: Copy`
- Fixed capacity `N` (const generic)
- `push(&mut self, item: T)` — overwrites oldest element when full
- `pop(&mut self) -> Option<T>` — returns oldest element
- `len(&self) -> usize`
- `is_empty(&self) -> bool`
- Must compile with `#![no_std]`

```rust
// Starter code
#![no_std]

use core::mem::MaybeUninit;

pub struct RingBuffer<T: Copy, const N: usize> {
    buf: [MaybeUninit<T>; N],
    head: usize,  // next write position
    tail: usize,  // next read position
    count: usize,
}

impl<T: Copy, const N: usize> RingBuffer<T, N> {
    pub const fn new() -> Self {
        todo!()
    }
    pub fn push(&mut self, item: T) {
        todo!()
    }
    pub fn pop(&mut self) -> Option<T> {
        todo!()
    }
    pub fn len(&self) -> usize {
        todo!()
    }
    pub fn is_empty(&self) -> bool {
        todo!()
    }
}
```

<details>
<summary>Solution</summary>

```rust
#![no_std]

use core::mem::MaybeUninit;

pub struct RingBuffer<T: Copy, const N: usize> {
    buf: [MaybeUninit<T>; N],
    head: usize,
    tail: usize,
    count: usize,
}

impl<T: Copy, const N: usize> RingBuffer<T, N> {
    pub const fn new() -> Self {
        Self {
            // SAFETY: MaybeUninit does not require initialization
            buf: unsafe { MaybeUninit::uninit().assume_init() },
            head: 0,
            tail: 0,
            count: 0,
        }
    }

    pub fn push(&mut self, item: T) {
        self.buf[self.head] = MaybeUninit::new(item);
        self.head = (self.head + 1) % N;
        if self.count == N {
            // Buffer is full — overwrite oldest, advance tail
            self.tail = (self.tail + 1) % N;
        } else {
            self.count += 1;
        }
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.count == 0 {
            return None;
        }
        // SAFETY: We only read positions that were previously written via push()
        let item = unsafe { self.buf[self.tail].assume_init() };
        self.tail = (self.tail + 1) % N;
        self.count -= 1;
        Some(item)
    }

    pub fn len(&self) -> usize {
        self.count
    }

    pub fn is_empty(&self) -> bool {
        self.count == 0
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn basic_push_pop() {
        let mut rb = RingBuffer::<u32, 4>::new();
        assert!(rb.is_empty());

        rb.push(10);
        rb.push(20);
        rb.push(30);
        assert_eq!(rb.len(), 3);

        assert_eq!(rb.pop(), Some(10));
        assert_eq!(rb.pop(), Some(20));
        assert_eq!(rb.pop(), Some(30));
        assert_eq!(rb.pop(), None);
    }

    #[test]
    fn overwrite_on_full() {
        let mut rb = RingBuffer::<u8, 3>::new();
        rb.push(1);
        rb.push(2);
        rb.push(3);
        // Buffer full: [1, 2, 3]

        rb.push(4); // Overwrites 1 → [4, 2, 3], tail advances
        assert_eq!(rb.len(), 3);
        assert_eq!(rb.pop(), Some(2)); // oldest surviving
        assert_eq!(rb.pop(), Some(3));
        assert_eq!(rb.pop(), Some(4));
        assert_eq!(rb.pop(), None);
    }
}
```

**Why this matters for embedded C devs:**
- `MaybeUninit` is Rust's equivalent of uninitialized memory — the compiler
  won't insert zero-fills, just like `char buf[N];` in C
- The `unsafe` blocks are minimal (2 lines) and each has a `// SAFETY:` comment
- The `const fn new()` means you can create ring buffers in `static` variables
  without a runtime constructor
- The tests run on your host with `cargo test` even though the code is `no_std`

</details>


