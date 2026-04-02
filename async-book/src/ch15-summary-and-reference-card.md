# 摘要和参考卡片

## 快速参考卡片

### 异步心智模型

```text
┌─────────────────────────────────────────────────────┐
│  async fn → 状态机 (enum) → impl Future             │
│  .await   → 轮询内部 future                         │
│  executor → loop { poll(); sleep_until_woken(); }   │
│  waker    → "嘿执行器，再轮询我"                   │
│  Pin      → "承诺我不会在内存中移动"                │
└─────────────────────────────────────────────────────┘
```

### 常见模式速查表

| 目标 | 使用 |
|------|-----|
| 并发运行两个 futures | `tokio::join!(a, b)` |
| 让两个 futures 竞速 | `tokio::select! { ... }` |
| 生成后台任务 | `tokio::spawn(async { ... })` |
| 在异步中运行阻塞代码 | `tokio::task::spawn_blocking(\| \| { ... })` |
| 限制并发 | `Semaphore::new(N)` |
| 收集多个任务结果 | `JoinSet` |
| 跨任务共享状态 | `Arc<Mutex<T>>` 或通道 |
| 优雅关闭 | `watch::channel` + `select!` |
| 一次处理 N 个 stream 项 | `.buffer_unordered(N)` |
| 给 future 加超时 | `tokio::time::timeout(dur, fut)` |
| 带退避的重试 | 自定义组合器（见第 13 章） |

### Pinning 快速参考

| 情况 | 使用 |
|-----------|-----|
| 在堆上 pin future | `Box::pin(fut)` |
| 在栈上 pin future | `tokio::pin!(fut)` |
| Pin 一个 `Unpin` 类型 | `Pin::new(&mut val)` — 安全，无开销 |
| 返回 pinned trait 对象 | `-> Pin<Box<dyn Future<Output = T> + Send>>` |

### 通道选择指南

| 通道 | 生产者 | 消费者 | 值 | 何时使用 |
|---------|-----------|-----------|--------|----------|
| `mpsc` | N | 1 | 流 | 工作队列、事件总线 |
| `oneshot` | 1 | 1 | 单个 | 请求/响应、完成通知 |
| `broadcast` | N | N | 所有接收者都收到所有 | 扇出通知、关闭信号 |
| `watch` | 1 | N | 只有最新 | 配置更新、健康状态 |

### Mutex 选择指南

| Mutex | 何时使用 |
|-------|----------|
| `std::sync::Mutex` | 锁持有时间很短，永远不在 `.await` 上跨越 |
| `tokio::sync::Mutex` | 锁必须在 `.await` 上持有 |
| `parking_lot::Mutex` | 高竞争，没有 `.await`，需要性能 |
| `tokio::sync::RwLock` | 多读者，少写者，锁跨越 `.await` |

### 决策快速参考

```text
需要并发？
├── I/O 密集型 → async/await
├── CPU 密集型 → rayon / std::thread
└── 混合型 → 对 CPU 部分使用 spawn_blocking

选择运行时？
├── 服务器应用 → tokio
├── 库 → 运行时无关（futures crate）
├── 嵌入式 → embassy
└── 极简 → smol

需要并发的 futures？
├── 可以是 'static + Send → tokio::spawn
├── 可以是 'static + !Send → LocalSet
├── 不能是 'static → FuturesUnordered
└── 需要跟踪/中止 → JoinSet
```

### 常见错误消息和修复

| 错误 | 原因 | 修复 |
|-------|-------|-----|
| `future is not Send` | 在 `.await` 上持有 `!Send` 类型 | 将值作用域化以便在 `.await` 之前 drop，或使用 `current_thread` 运行时 |
| `borrowed value does not live long enough` 在 spawn 中 | `tokio::spawn` 要求 `'static` | 使用 `Arc`、`clone()` 或 `FuturesUnordered` |
| `the trait Future is not implemented for ()` | 缺少 `.await` | 在异步调用后添加 `.await` |
| 在 poll 中 `cannot borrow as mutable` | 自引用借用 | 正确使用 `Pin<&mut Self>`（见第 4 章） |
| 程序静默挂起 | 忘记调用 `waker.wake()` | 确保每个 `Pending` 路径都注册并触发 waker |

### 延伸阅读

| 资源 | 为什么 |
|----------|---------|
| [Tokio 教程](https://tokio.rs/tokio/tutorial) | 官方实践指南 — 对第一个项目非常出色 |
| [Async Book（官方）](https://rust-lang.github.io/async-book/) | 在语言层面涵盖 `Future`、`Pin`、`Stream` |
| [Jon Gjengset — Crust of Rust: async/await](https://www.youtube.com/watch?v=ThjvMReOXYM) | 2 小时深入探索内部结构与现场编码 |
| [Alice Ryhl — 使用 Tokio 的 Actor](https://ryhl.io/blog/actors-with-tokio/) | 有状态服务的生产架构模式 |
| [Without Boats — Pin、Unpin 以及为什么 Rust 需要它们](https://without.boats/blog/pin/) | 语言设计者的原始动机 |
| [Tokio mini-Redis](https://github.com/tokio-rs/mini-redis) | 完整的异步 Rust 项目 — 可供研究的生产级代码 |
| [Tower 文档](https://docs.rs/tower) | axum、tonic、hyper 使用的中间件/服务架构 |

***

*异步 Rust 训练指南 完*
