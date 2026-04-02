# 7. 执行器和运行时 🟡

> **你将学到：**
> - 执行器做什么：高效地 poll + sleep
> - 六个主要运行时：mio、io_uring、tokio、async-std、smol、embassy
> - 选择正确运行时的决策树
> - 为什么运行时无关的库设计很重要

## 执行器做什么

执行器有两项工作：
1. **轮询 futures** 当它们准备好取得进展时
2. **高效睡眠** 当没有 futures 就绪时（使用 OS I/O 通知 API）

```mermaid
graph TB
    subgraph Executor["执行器（如 tokio）"]
        QUEUE["任务队列"]
        POLLER["I/O 轮询器<br/>(epoll/kqueue/io_uring)"]
        THREADS["工作线程池"]
    end

    subgraph Tasks
        T1["任务 1<br/>(HTTP 请求)"]
        T2["任务 2<br/>(数据库查询)"]
        T3["任务 3<br/>(文件读取)"]
    end

    subgraph OS["操作系统"]
        NET["网络栈"]
        DISK["磁盘 I/O"]
    end

    T1 --> QUEUE
    T2 --> QUEUE
    T3 --> QUEUE
    QUEUE --> THREADS
    THREADS -->|"poll()"| T1
    THREADS -->|"poll()"| T2
    THREADS -->|"poll()"| T3
    POLLER <-->|"注册/通知"| NET
    POLLER <-->|"注册/通知"| DISK
    POLLER -->|"唤醒任务"| QUEUE

    style Executor fill:#e3f2fd,color:#000
    style OS fill:#f3e5f5,color:#000
```

### mio：基础层

[mio](https://github.com/tokio-rs/mio)（Metal I/O）不是执行器——它是最底层的跨平台 I/O 通知库。它封装了 `epoll`（Linux）、`kqueue`（macOS/BSD）和 IOCP（Windows）。

```rust
// 概念性的 mio 用法（简化版）：
use mio::{Events, Interest, Poll, Token};
use mio::net::TcpListener;

let mut poll = Poll::new()?;
let mut events = Events::with_capacity(128);

let mut server = TcpListener::bind("0.0.0.0:8080")?;
poll.registry().register(&mut server, Token(0), Interest::READABLE)?;

// 事件循环——阻塞直到有事发生
loop {
    poll.poll(&mut events, None)?; // 睡眠直到 I/O 事件
    for event in events.iter() {
        match event.token() {
            Token(0) => { /* 服务器有新连接 */ }
            _ => { /* 其他 I/O 就绪 */ }
        }
    }
}
```

大多数开发者从不直接接触 mio——tokio 和 smol 构建在其之上。

### io_uring：基于完成态的 Future

Linux 的 `io_uring`（内核 5.1+）代表了从 mio/epoll 使用的基于就绪态 I/O 模型的根本转变：

```text
基于就绪态（epoll / mio / tokio）：
  1. 问："这个 socket 可读吗？"     → epoll_wait()
  2. 内核："是的，它就绪了"         → EPOLLIN 事件
  3. 应用：  read(fd, buf)          → 可能仍然短暂阻塞！

基于完成态（io_uring）：
  1. 提交："从这个 socket 读取到这个缓冲区"  → SQE
  2. 内核：异步执行读取
  3. 应用：获得包含数据的已完成结果          → CQE
```

```mermaid
graph LR
    subgraph "就绪态模型（epoll）"
        A1["应用：就绪了吗？"] --> K1["内核：是的"]
        K1 --> A2["应用：现在 read()"]
        A2 --> K2["内核：给你数据"]
    end

    subgraph "完成态模型（io_uring）"
        B1["应用：帮我读这个"] --> K3["内核：工作中..."]
        K3 --> B2["应用：收到结果 + 数据"]
    end

    style B1 fill:#c8e6c9,color:#000
    style B2 fill:#c8e6c9,color:#000
```

**所有权挑战**：io_uring 要求内核拥有缓冲区直到操作完成。这与 Rust 标准 `AsyncRead` trait 借用缓冲区的设计冲突。这就是为什么 `tokio-uring` 有不同的 I/O trait：

```rust
// 标准 tokio（基于就绪态）— 借用缓冲区：
let n = stream.read(&mut buf).await?;  // buf 是借用的

// tokio-uring（基于完成态）— 获得缓冲区所有权：
let (result, buf) = stream.read(buf).await;  // buf 移入，然后返回
let n = result?;
```

```rust
// Cargo.toml: tokio-uring = "0.5"
// 注意：仅限 Linux，需要内核 5.1+

fn main() {
    tokio_uring::start(async {
        let file = tokio_uring::fs::File::open("data.bin").await.unwrap();
        let buf = vec![0u8; 4096];
        let (result, buf) = file.read_at(buf, 0).await;
        let bytes_read = result.unwrap();
        println!("Read {} bytes: {:?}", bytes_read, &buf[..bytes_read]);
    });
}
```

| 方面 | epoll (tokio) | io_uring (tokio-uring) |
|--------|--------------|----------------------|
| **模型** | 就绪通知 | 完成通知 |
| **系统调用** | epoll_wait + read/write | 批处理 SQE/CQE ring |
| **缓冲区所有权** | 应用保留 (&mut buf) | 所有权转移（移动 buf） |
| **平台** | Linux, macOS (kqueue), Windows (IOCP) | 仅 Linux 5.1+ |
| **零拷贝** | 否（用户空间拷贝） | 是（注册的缓冲区） |
| **成熟度** | 生产就绪 | 实验性 |

> **何时使用 io_uring**：高吞吐量文件 I/O 或网络，其中系统调用开销是瓶颈（数据库、存储引擎、服务 100k+ 连接的代理）。对于大多数应用，带 epoll 的标准 tokio 是正确选择。

### tokio：包含电池的运行时

Rust 生态系统中的主导异步运行时。被 Axum、Hyper、Tonic 和大多数生产级 Rust 服务器使用。

```rust
// Cargo.toml:
// [dependencies]
// tokio = { version = "1", features = ["full"] }

#[tokio::main]
async fn main() {
    // 产生一个带有工作窃取调度器的多线程运行时
    let handle = tokio::spawn(async {
        tokio::time::sleep(std::time::Duration::from_secs(1)).await;
        "done"
    });

    let result = handle.await.unwrap();
    println!("{result}");
}
```

**tokio 特性**：Timer、I/O、TCP/UDP、Unix sockets、信号处理、同步原语（Mutex、RwLock、Semaphore、channels）、fs、process、tracing 集成。

### async-std：标准库镜像

用异步版本镜像 `std` API。不如 tokio 流行，但对初学者更简单。

```rust
// Cargo.toml:
// [dependencies]
// async-std = { version = "1", features = ["attributes"] }

#[async_std::main]
async fn main() {
    use async_std::fs;
    let content = fs::read_to_string("hello.txt").await.unwrap();
    println!("{content}");
}
```

### smol：极简运行时

小而零依赖的异步运行时。适合想要异步但不引入 tokio 的库。

```rust
// Cargo.toml:
// [dependencies]
// smol = "2"

fn main() {
    smol::block_on(async {
        let result = smol::unblock(|| {
            // 在线程池上运行阻塞代码
            std::fs::read_to_string("hello.txt")
        }).await.unwrap();
        println!("{result}");
    });
}
```

### embassy：嵌入式异步（no_std）

嵌入式系统的异步运行时。不需要堆分配，不需要 `std`。

```rust
// 运行在微控制器上（如 STM32、nRF52、RP2040）
#[embassy_executor::main]
async fn main(spawner: embassy_executor::Spawner) {
    // 用 async/await 闪烁 LED——不需要 RTOS！
    let mut led = Output::new(p.PA5, Level::Low, Speed::Low);
    loop {
        led.set_high();
        Timer::after(Duration::from_millis(500)).await;
        led.set_low();
        Timer::after(Duration::from_millis(500)).await;
    }
}
```

### 运行时决策树

```mermaid
graph TD
    START["选择运行时"]

    Q1{"构建<br/>网络服务器？"}
    Q2{"需要 tokio 生态<br/>(Axum、Tonic、Hyper)？"}
    Q3{"构建库？"}
    Q4{"嵌入式 /<br/>no_std？"}
    Q5{"想要最小<br/>依赖？"}

    TOKIO["🟢 tokio<br/>最佳生态，最流行"]
    SMOL["🔵 smol<br/>极简，无生态锁定"]
    EMBASSY["🟠 embassy<br/>嵌入式优先，无分配"]
    ASYNC_STD["🟣 async-std<br/>std-like API，适合学习"]
    AGNOSTIC["🔵 运行时无关<br/>仅使用 futures crate"]

    START --> Q1
    Q1 -->|是| Q2
    Q1 -->|否| Q3
    Q2 -->|是| TOKIO
    Q2 -->|否| Q5
    Q3 -->|是| AGNOSTIC
    Q3 -->|否| Q4
    Q4 -->|是| EMBASSY
    Q4 -->|否| Q5
    Q5 -->|是| SMOL
    Q5 -->|否| ASYNC_STD

    style TOKIO fill:#c8e6c9,color:#000
    style SMOL fill:#bbdefb,color:#000
    style EMBASSY fill:#ffe0b2,color:#000
    style ASYNC_STD fill:#e1bee7,color:#000
    style AGNOSTIC fill:#bbdefb,color:#000
```

### 运行时对比表

| 特性 | tokio | async-std | smol | embassy |
|---------|-------|-----------|------|---------|
| **生态** | 主导 | 小 | 极简 | 嵌入式 |
| **多线程** | ✅ 工作窃取 | ✅ | ✅ | ❌（单核） |
| **no_std** | ❌ | ❌ | ❌ | ✅ |
| **Timer** | ✅ 内置 | ✅ 内置 | 通过 `async-io` | ✅ 基于 HAL |
| **I/O** | ✅ 自有抽象 | ✅ std 镜像 | ✅ 通过 `async-io` | ✅ HAL 驱动 |
| **Channels** | ✅ 丰富集合 | ✅ | 通过 `async-channel` | ✅ |
| **学习曲线** | 中等 | 低 | 低 | 高（硬件） |
| **二进制大小** | 大 | 中 | 小 | 极小 |

<details>
<summary><strong>🏋️ 练习：运行时对比</strong>（点击展开）</summary>

**挑战**：用三个不同的运行时（tokio、smol 和 async-std）编写相同的程序。程序应该：
1. 获取一个 URL（用 sleep 模拟）
2. 读取一个文件（用 sleep 模拟）
3. 打印两个结果

这个练习表明 async/await 代码是相同的——只有运行时设置不同。

<details>
<summary>🔑 答案</summary>

```rust
// ----- tokio 版本 -----
// Cargo.toml: tokio = { version = "1", features = ["full"] }
#[tokio::main]
async fn main() {
    let (url_result, file_result) = tokio::join!(
        async {
            tokio::time::sleep(std::time::Duration::from_millis(100)).await;
            "Response from URL"
        },
        async {
            tokio::time::sleep(std::time::Duration::from_millis(50)).await;
            "Contents of file"
        },
    );
    println!("URL: {url_result}, File: {file_result}");
}

// ----- smol 版本 -----
// Cargo.toml: smol = "2", futures-lite = "2"
fn main() {
    smol::block_on(async {
        let (url_result, file_result) = futures_lite::future::zip(
            async {
                smol::Timer::after(std::time::Duration::from_millis(100)).await;
                "Response from URL"
            },
            async {
                smol::Timer::after(std::time::Duration::from_millis(50)).await;
                "Contents of file"
            },
        ).await;
        println!("URL: {url_result}, File: {file_result}");
    });
}

// ----- async-std 版本 -----
// Cargo.toml: async-std = { version = "1", features = ["attributes"] }
#[async_std::main]
async fn main() {
    let (url_result, file_result) = futures::future::join(
        async {
            async_std::task::sleep(std::time::Duration::from_millis(100)).await;
            "Response from URL"
        },
        async {
            async_std::task::sleep(std::time::Duration::from_millis(50)).await;
            "Contents of file"
        },
    ).await;
    println!("URL: {url_result}, File: {file_result}");
}
```

**关键要点**：跨运行时，异步业务逻辑是相同的。只有入口点和 timer/IO API 不同。这就是为什么编写运行时无关的库（仅使用 `std::future::Future`）是有价值的。

</details>
</details>

> **核心要点 — 执行器和运行时**
> - 执行器的工作：被唤醒时轮询 futures，使用 OS I/O API 高效睡眠
> - **tokio** 是服务器的默认选择；**smol** 用于最小占用；**embassy** 用于嵌入式
> - 你的业务逻辑应该依赖 `std::future::Future`，而不是特定运行时
> - io_uring（Linux 5.1+）是高性能 I/O 的未来，但生态系统仍在成熟中

> **另见：** [第 8 章 — Tokio 深度探讨](ch08-tokio-deep-dive.md) 了解 tokio 细节，[第 9 章 — 何时 Tokio 不是最佳选择](ch09-when-tokio-isnt-the-right-fit.md) 了解替代方案

***
