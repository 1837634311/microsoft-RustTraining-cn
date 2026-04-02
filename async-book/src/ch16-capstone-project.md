# 终极项目：异步聊天服务器

这个项目将书中的模式整合到一个单一的生产级应用程序中。你将使用 tokio、通道、streams、优雅关闭和正确的错误处理构建一个**多房间异步聊天服务器**。

**预计时间**：4–6 小时 | **难度**：★★★

> **你将练习的内容：**
> - `tokio::spawn` 和 `'static` 要求（第 8 章）
> - 通道：`mpsc` 用于消息，`broadcast` 用于房间，`watch` 用于关闭（第 8 章）
> - Streams：从 TCP 连接读取行（第 11 章）
> - 常见陷阱：取消安全性、跨 `.await` 的 MutexGuard（第 12 章）
> - 生产模式：优雅关闭、背压（第 13 章）
> - 异步 traits 用于可插拔后端（第 10 章）

## 问题

构建一个 TCP 聊天服务器，其中：

1. **客户端**通过 TCP 连接并加入命名房间
2. **消息**被广播到同一房间的所有客户端
3. **命令**：`/join <room>`、`/nick <name>`、`/rooms`、`/quit`
4. 服务器在 Ctrl+C 时优雅关闭——完成处理中的消息

```mermaid
graph LR
    C1["客户端 1<br/>(Alice)"] -->|TCP| SERVER["聊天服务器"]
    C2["客户端 2<br/>(Bob)"] -->|TCP| SERVER
    C3["客户端 3<br/>(Carol)"] -->|TCP| SERVER

    SERVER --> R1["#general<br/>广播通道"]
    SERVER --> R2["#rust<br/>广播通道"]

    R1 -->|消息| C1
    R1 -->|消息| C2
    R2 -->|消息| C3

    CTRL["Ctrl+C"] -->|watch| SERVER

    style SERVER fill:#e8f4f8,stroke:#2980b9,color:#000
    style R1 fill:#d4efdf,stroke:#27ae60,color:#000
    style R2 fill:#d4efdf,stroke:#27ae60,color:#000
    style CTRL fill:#fadbd8,stroke:#e74c3c,color:#000
```

## 步骤 1：基本 TCP 接受循环

从一个接受连接并回显行的服务器开始：

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:8080").await?;
    println!("Chat server listening on :8080");

    loop {
        let (socket, addr) = listener.accept().await?;
        println!("[{addr}] Connected");

        tokio::spawn(async move {
            let (reader, mut writer) = socket.into_split();
            let mut reader = BufReader::new(reader);
            let mut line = String::new();

            loop {
                line.clear();
                match reader.read_line(&mut line).await {
                    Ok(0) | Err(_) => break,
                    Ok(_) => {
                        let _ = writer.write_all(line.as_bytes()).await;
                    }
                }
            }
            println!("[{addr}] Disconnected");
        });
    }
}
```

**你的任务**：验证这能编译并用 `telnet localhost 8080` 工作。

## 步骤 2：使用 Broadcast 通道的房间状态

每个房间是一个 `broadcast::Sender`。同一房间的所有客户端订阅以接收消息。

```rust
use std::collections::HashMap;
use std::sync::Arc;
use tokio::sync::{broadcast, RwLock};

type RoomMap = Arc<RwLock<HashMap<String, broadcast::Sender<String>>>>;

fn get_or_create_room(rooms: &mut HashMap<String, broadcast::Sender<String>>, name: &str) -> broadcast::Sender<String> {
    rooms.entry(name.to_string())
        .or_insert_with(|| {
            let (tx, _) = broadcast::channel(100); // 100 消息缓冲区
            tx
        })
        .clone()
}
```

**你的任务**：实现房间状态，以便：
- 客户端从 `#general` 开始
- `/join <room>` 切换房间（取消订阅旧的，订阅新的）
- 消息被广播到发送者当前房间的所有客户端

<details>
<summary>💡 提示 — 客户端任务结构</summary>

每个客户端任务需要两个并发循环：
1. **从 TCP 读取** → 解析命令或广播到房间
2. **从广播接收器读取** → 写入 TCP

使用 `tokio::select!` 来同时运行两者：

```rust
loop {
    tokio::select! {
        // 客户端发送了一行
        result = reader.read_line(&mut line) => {
            match result {
                Ok(0) | Err(_) => break,
                Ok(_) => {
                    // 解析命令或广播消息
                }
            }
        }
        // 收到房间广播
        result = room_rx.recv() => {
            match result {
                Ok(msg) => {
                    let _ = writer.write_all(msg.as_bytes()).await;
                }
                Err(_) => break,
            }
        }
    }
}
```

</details>

## 步骤 3：命令

实现命令协议：

| 命令 | 动作 |
|---------|--------|
| `/join <room>` | 离开当前房间，加入新房间，在两者中都宣布 |
| `/nick <name>` | 更改显示名称 |
| `/rooms` | 列出所有活动房间和成员数量 |
| `/quit` | 优雅断开连接 |
| 其他任何内容 | 作为聊天消息广播 |

**你的任务**：从输入行解析命令。对于 `/rooms`，你需要从 `RoomMap` 读取——使用 `RwLock::read()` 以避免阻塞其他客户端。

## 步骤 4：优雅关闭

添加 Ctrl+C 处理，以便服务器：
1. 停止接受新连接
2. 向所有房间发送"服务器正在关闭..."
3. 等待处理中的消息排空
4. 干净地退出

```rust
use tokio::sync::watch;

let (shutdown_tx, shutdown_rx) = watch::channel(false);

// 在接受循环中：
loop {
    tokio::select! {
        result = listener.accept() => {
            let (socket, addr) = result?;
            // 用 shutdown_rx.clone() 生成客户端任务
        }
        _ = tokio::signal::ctrl_c() => {
            println!("Shutdown signal received");
            shutdown_tx.send(true)?;
            break;
        }
    }
}
```

**你的任务**：将 `shutdown_rx.changed()` 添加到每个客户端的 `select!` 循环中，以便在发出关闭信号时客户端退出。

## 步骤 5：错误处理和边缘情况

为服务器加强生产准备：

1. **落后的接收者**：如果慢客户端错过消息，`broadcast::recv()` 返回 `RecvError::Lagged(n)`。优雅处理它（记录 + 继续，不要崩溃）。
2. **昵称验证**：拒绝空或太长的昵称。
3. **背压**：广播通道缓冲区是有界的（100）。如果客户端跟不上，他们会收到 `Lagged` 错误。
4. **超时**：断开空闲 >5 分钟的客户端。

```rust
use tokio::time::{timeout, Duration};

// 在读取外包装超时：
match timeout(Duration::from_secs(300), reader.read_line(&mut line)).await {
    Ok(Ok(0)) | Ok(Err(_)) | Err(_) => break, // EOF、错误或超时
    Ok(Ok(_)) => { /* 处理行 */ }
}
```

## 步骤 6：集成测试

编写一个测试，启动服务器，连接两个客户端，并验证消息传递：

```rust
#[tokio::test]
async fn two_clients_can_chat() {
    // 在后台启动服务器
    let server = tokio::spawn(run_server("127.0.0.1:0")); // 端口 0 = OS 选择

    // 连接两个客户端
    let mut client1 = TcpStream::connect(addr).await.unwrap();
    let mut client2 = TcpStream::connect(addr).await.unwrap();

    // 客户端 1 发送消息
    client1.write_all(b"Hello from client 1\n").await.unwrap();

    // 客户端 2 应该收到它
    let mut buf = vec![0u8; 1024];
    let n = client2.read(&mut buf).await.unwrap();
    let msg = String::from_utf8_lossy(&buf[..n]);
    assert!(msg.contains("Hello from client 1"));
}
```

## 评估标准

| 标准 | 目标 |
|-----------|--------|
| 并发性 | 多房间中的多个客户端，无阻塞 |
| 正确性 | 消息只发送给同一房间的客户端 |
| 优雅关闭 | Ctrl+C 排空消息并干净退出 |
| 错误处理 | 落后接收者、断开连接、超时被处理 |
| 代码组织 | 清晰分离：接受循环、客户端任务、房间状态 |
| 测试 | 至少 2 个集成测试 |

## 扩展想法

一旦基本聊天服务器工作了，尝试这些增强：

1. **持久历史**：每个房间存储最后 N 条消息；向新加入者重放
2. **WebSocket 支持**：使用 `tokio-tungstenite` 同时接受 TCP 和 WebSocket 客户端
3. **速率限制**：使用 `tokio::time::Interval` 限制每个客户端每秒的消息数
4. **指标**：通过 `prometheus` crate 跟踪连接的客户端、消息/秒、房间数
5. **TLS**：添加 `tokio-rustls` 以获得加密连接

***
