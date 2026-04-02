# 8. Tokio 深度探讨 🟡

> **你将学到：**
> - 运行时风格：多线程 vs 当前线程以及何时使用各自
> - `tokio::spawn`、`'static` 要求和 `JoinHandle`
> - 任务取消语义（drop 时取消）
> - 同步原语：Mutex、RwLock、Semaphore 和所有四种通道类型

## 运行时风格：多线程 vs 当前线程

Tokio 提供两种运行时配置：

```rust
// 多线程（#[tokio::main] 的默认值）
// 使用工作窃取线程池——任务可以在线程之间移动
#[tokio::main]
async fn main() {
    // N 个工作线程（默认 = CPU 核心数）
    // 任务是 Send + 'static
}

// 当前线程——所有内容在一个线程上运行
#[tokio::main(flavor = "current_thread")]
async fn main() {
    // 单线程——任务不需要是 Send
    // 更轻量，适合简单工具或 WASM
}

// 手动运行时构建：
let rt = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(4)
    .enable_all()
    .build()
    .unwrap();

rt.block_on(async {
    println!("Running on custom runtime");
});
```

```mermaid
graph TB
    subgraph "多线程（默认）"
        MT_Q1["线程 1<br/>任务 A、任务 D"]
        MT_Q2["线程 2<br/>任务 B"]
        MT_Q3["线程 3<br/>任务 C、任务 E"]
        STEAL["工作窃取：<br/>空闲线程从繁忙线程窃取任务"]
        MT_Q1 <--> STEAL
        MT_Q2 <--> STEAL
        MT_Q3 <--> STEAL
    end

    subgraph "当前线程"
        ST_Q["单线程<br/>任务 A → 任务 B → 任务 C → 任务 D"]
    end

    style MT_Q1 fill:#c8e6c9,color:#000
    style MT_Q2 fill:#c8e6c9,color:#000
    style MT_Q3 fill:#c8e6c9,color:#000
    style ST_Q fill:#bbdefb,color:#000
```

### tokio::spawn 和 'static 要求

`tokio::spawn` 将一个 future 放到运行时的任务队列上。因为它可能在*任何*工作线程上*任何*时候运行，future 必须是 `Send + 'static`：

```rust
use tokio::task;

async fn example() {
    let data = String::from("hello");

    // ✅ 有效：将所有权移动到任务中
    let handle = task::spawn(async move {
        println!("{data}");
        data.len()
    });

    let len = handle.await.unwrap();
    println!("Length: {len}");
}

async fn problem() {
    let data = String::from("hello");

    // ❌ 失败：data 是借用的，不是 'static
    // task::spawn(async {
    //     println!("{data}"); // 借用 `data` — 不是 'static
    // });

    // ❌ 失败：Rc 不是 Send
    // let rc = std::rc::Rc::new(42);
    // task::spawn(async move {
    //     println!("{rc}"); // Rc 是 !Send — 不能跨线程边界
    // });
}
```

**为什么需要 `'static`？** 生成的任务独立运行——它可能比创建它的作用域活得更久。编译器无法证明引用将保持有效，所以它要求拥有数据。

**为什么需要 `Send`？** 任务可能在与它暂停时不同的线程上恢复。在 `.await` 点之间持有的所有数据必须能安全地在线程之间发送。

```rust
// 常见模式：将共享数据克隆到任务中
let shared = Arc::new(config);

for i in 0..10 {
    let shared = Arc::clone(&shared); // 克隆 Arc，不是数据
    tokio::spawn(async move {
        process_item(i, &shared).await;
    });
}
```

### JoinHandle 和任务取消

```rust
use tokio::task::JoinHandle;
use tokio::time::{sleep, Duration};

async fn cancellation_example() {
    let handle: JoinHandle<String> = tokio::spawn(async {
        sleep(Duration::from_secs(10)).await;
        "completed".to_string()
    });

    // 通过 drop handle 来取消任务？不——任务继续运行！
    // drop(handle); // 任务在后台继续

    // 要真正取消，调用 abort()：
    handle.abort();

    // 等待被中止的任务返回 JoinError
    match handle.await {
        Ok(val) => println!("Got: {val}"),
        Err(e) if e.is_cancelled() => println!("Task was cancelled"),
        Err(e) => println!("Task panicked: {e}"),
    }
}
```

> **重要**：在 tokio 中，Drop `JoinHandle` 不会取消任务。
> 任务变成*分离的*并继续运行。你必须显式调用
> `.abort()` 来取消它。这与直接 drop `Future` 不同，
> 后者确实会取消/放弃底层计算。

### Tokio 同步原语

Tokio 提供异步感知的同步原语。关键原则：**不要在 `.await` 点之间使用 `std::sync::Mutex`**。

```rust
use tokio::sync::{Mutex, RwLock, Semaphore, mpsc, oneshot, broadcast, watch};

// --- Mutex ---
// 异步 mutex：lock() 方法是异步的，不会阻塞线程
let data = Arc::new(Mutex::new(vec![1, 2, 3]));
{
    let mut guard = data.lock().await; // 非阻塞锁
    guard.push(4);
} // Guard 在这里被 drop — 锁释放

// --- Channels ---
// mpsc：多生产者，单消费者
let (tx, mut rx) = mpsc::channel::<String>(100); // 有界缓冲区

tokio::spawn(async move {
    tx.send("hello".into()).await.unwrap();
});

let msg = rx.recv().await.unwrap();

// oneshot：单值，单消费者
let (tx, rx) = oneshot::channel::<i32>();
tx.send(42).unwrap(); // 不需要 await — 要么发送成功要么失败
let val = rx.await.unwrap();

// broadcast：多生产者，多消费者（所有接收者都能收到每条消息）
let (tx, _) = broadcast::channel::<String>(100);
let mut rx1 = tx.subscribe();
let mut rx2 = tx.subscribe();

// watch：单值，多消费者（只有最新值）
let (tx, rx) = watch::channel(0u64);
tx.send(42).unwrap();
println!("Latest: {}", *rx.borrow());
```

```mermaid
graph LR
    subgraph "通道类型"
        direction TB
        MPSC["mpsc<br/>N→1<br/>有界队列"]
        ONESHOT["oneshot<br/>1→1<br/>单值"]
        BROADCAST["broadcast<br/>N→N<br/>所有接收者都收到所有消息"]
        WATCH["watch<br/>1→N<br/>只有最新值"]
    end

    P1["生产者 1"] --> MPSC
    P2["生产者 2"] --> MPSC
    MPSC --> C1["消费者"]

    P3["生产者"] --> ONESHOT
    ONESHOT --> C2["消费者"]

    P4["生产者"] --> BROADCAST
    BROADCAST --> C3["消费者 1"]
    BROADCAST --> C4["消费者 2"]

    P5["生产者"] --> WATCH
    WATCH --> C5["消费者 1"]
    WATCH --> C6["消费者 2"]
```

## 案例研究：为通知服务选择正确的通道

你正在构建一个通知服务，其中：
- 多个 API 处理器产生事件
- 一个后台任务批量发送它们
- 一个配置监视器在运行时更新速率限制
- 一个关闭信号必须到达所有组件

**每个用哪些通道？**

| 需求 | 通道 | 为什么 |
|-------------|---------|-----|
| API 处理器 → 批处理器 | `mpsc`（有界） | N 个生产者，1 个消费者。有界以实现背压——如果批处理器落后，API 处理器会变慢而不是 OOM |
| 配置监视器 → 速率限制器 | `watch` | 只有最新配置重要。多个读者（每个 worker）看到当前值 |
| 关闭信号 → 所有组件 | `broadcast` | 每个组件必须独立接收关闭通知 |
| 单个健康检查响应 | `oneshot` | 请求/响应模式——一个值，然后结束 |

```mermaid
graph LR
    subgraph "通知服务"
        direction TB
        API1["API Handler 1"] -->|mpsc| BATCH["批处理器"]
        API2["API Handler 2"] -->|mpsc| BATCH
        CONFIG["配置监视器"] -->|watch| RATE["速率限制器"]
        CTRL["Ctrl+C"] -->|broadcast| API1
        CTRL -->|broadcast| BATCH
        CTRL -->|broadcast| RATE
    end

    style API1 fill:#d4efdf,stroke:#27ae60,color:#000
    style API2 fill:#d4efdf,stroke:#27ae60,color:#000
    style BATCH fill:#e8f4f8,stroke:#2980b9,color:#000
    style CONFIG fill:#fef9e7,stroke:#f39c12,color:#000
    style RATE fill:#fef9e7,stroke:#f39c12,color:#000
    style CTRL fill:#fadbd8,stroke:#e74c3c,color:#000
```

<details>
<summary><strong>🏋️ 练习：构建任务池</strong>（点击展开）</summary>

**挑战**：构建一个函数 `run_with_limit`，接受一个异步闭包列表和一个并发限制，最多同时执行 N 个任务。使用 `tokio::sync::Semaphore`。

<details>
<summary>🔑 答案</summary>

```rust
use std::future::Future;
use std::sync::Arc;
use tokio::sync::Semaphore;

async fn run_with_limit<F, Fut, T>(tasks: Vec<F>, limit: usize) -> Vec<T>
where
    F: FnOnce() -> Fut + Send + 'static,
    Fut: Future<Output = T> + Send + 'static,
    T: Send + 'static,
{
    let semaphore = Arc::new(Semaphore::new(limit));
    let mut handles = Vec::new();

    for task in tasks {
        let permit = Arc::clone(&semaphore);
        let handle = tokio::spawn(async move {
            let _permit = permit.acquire().await.unwrap();
            // 任务运行时持有 permit，然后 drop
            task().await
        });
        handles.push(handle);
    }

    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

// 用法：
// let tasks: Vec<_> = urls.into_iter().map(|url| {
//     move || async move { fetch(url).await }
// }).collect();
// let results = run_with_limit(tasks, 10).await; // 最多 10 个并发
```

**关键要点**：`Semaphore` 是 tokio 中限制并发的标准方式。每个任务在开始工作前获取一个 permit。当信号量满时，新任务异步等待（非阻塞）直到有槽位空出。

</details>
</details>

> **核心要点 — Tokio 深度探讨**
> - 服务器使用 `multi_thread`（默认）；CLI 工具、测试或 `!Send` 类型使用 `current_thread`
> - `tokio::spawn` 要求 `'static` futures——使用 `Arc` 或通道共享数据
> - Drop `JoinHandle` 不会取消任务——显式调用 `.abort()`
> - 根据需要选择同步原语：`Mutex` 用于共享状态，`Semaphore` 用于并发限制，`mpsc`/`oneshot`/`broadcast`/`watch` 用于通信

> **另见：** [第 9 章 — 何时 Tokio 不是最佳选择](ch09-when-tokio-isnt-the-right-fit.md) 了解 spawn 的替代方案，[第 12 章 — 常见陷阱](ch12-common-pitfalls.md) 了解 MutexGuard 跨 await 的 bug

***
