## Crate级错误类型和Result别名

> **学习内容：** 使用`thiserror`定义per-crate错误枚举的生产模式、
> 创建`Result<T>`类型别名，以及何时选择`thiserror`（库）vs `anyhow`（应用程序）。
>
> **难度：** 🟡 中级

生产级Rust的关键模式：定义per-crate错误枚举和`Result`类型别名以消除样板代码。

### 模式
```rust
// src/error.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),

    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),

    #[error("Serialization error: {0}")]
    Serialization(#[from] serde_json::Error),

    #[error("Validation error: {message}")]
    Validation { message: String },

    #[error("Not found: {entity} with id {id}")]
    NotFound { entity: String, id: String },
}

/// Crate-wide Result alias — every function returns this
pub type Result<T> = std::result::Result<T, AppError>;
```

### 在整个Crate中使用
```rust
use crate::error::{AppError, Result};

pub async fn get_user(id: Uuid) -> Result<User> {
    let user = sqlx::query_as!(User, "SELECT * FROM users WHERE id = $1", id)
        .fetch_optional(&pool)
        .await?;  // sqlx::Error → AppError::Database via #[from]

    user.ok_or_else(|| AppError::NotFound {
        entity: "User".into(),
        id: id.to_string(),
    })
}

pub async fn create_user(req: CreateUserRequest) -> Result<User> {
    if req.name.trim().is_empty() {
        return Err(AppError::Validation {
            message: "Name cannot be empty".into(),
        });
    }
    // ...
}
```

### C# 对比
```csharp
// C# equivalent pattern
public class AppException : Exception
{
    public string ErrorCode { get; }
    public AppException(string code, string message) : base(message)
    {
        ErrorCode = code;
    }
}

// But in C#, callers don't know what exceptions to expect!
// In Rust, the error type is in the function signature.
```

### 为什么这很重要
- **`thiserror`** generates `Display` and `Error` impls automatically
- **`#[from]`** enables the `?` operator to convert library errors automatically
- The `Result<T>` alias means every function signature is clean: `fn foo() -> Result<Bar>`
- **Unlike C# exceptions**, callers see all possible error variants in the type


### thiserror vs anyhow：何时使用哪个

Two crates dominate Rust error handling. Choosing between them is the first decision you'll make:

| | `thiserror` | `anyhow` |
|---|---|---|
| **Purpose** | Define structured error types for **libraries** | Quick error handling for **applications** |
| **Output** | Custom enum you control | Opaque `anyhow::Error` wrapper |
| **Caller sees** | All error variants in the type | Just `anyhow::Error` — opaque |
| **Best for** | Library crates, APIs, any code with consumers | Binaries, scripts, prototypes, CLI tools |
| **Downcasting** | `match` on variants directly | `error.downcast_ref::<MyError>()` |

```rust
// thiserror — for LIBRARIES (callers need to match on error variants)
use thiserror::Error;

#[derive(Error, Debug)]
pub enum StorageError {
    #[error("File not found: {path}")]
    NotFound { path: String },

    #[error("Permission denied: {0}")]
    PermissionDenied(String),

    #[error(transparent)]
    Io(#[from] std::io::Error),
}

pub fn read_config(path: &str) -> Result<String, StorageError> {
    std::fs::read_to_string(path).map_err(|e| match e.kind() {
        std::io::ErrorKind::NotFound => StorageError::NotFound { path: path.into() },
        std::io::ErrorKind::PermissionDenied => StorageError::PermissionDenied(path.into()),
        _ => StorageError::Io(e),
    })
}
```

```rust
// anyhow — for APPLICATIONS (just propagate errors, don't define types)
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let config = std::fs::read_to_string("config.toml")
        .context("Failed to read config file")?;

    let port: u16 = config.parse()
        .context("Failed to parse port number")?;

    println!("Listening on port {port}");
    Ok(())
}
// anyhow::Result<T> = Result<T, anyhow::Error>
// .context() adds human-readable context to any error
```

```csharp
// C# comparison:
// thiserror ≈ defining custom exception classes with specific properties
// anyhow ≈ catching Exception and wrapping with message:
//   throw new InvalidOperationException("Failed to read config", ex);
```

**Guideline**: If your code is a **library** (other code calls it), use `thiserror`. If your code is an **application** (the final binary), use `anyhow`. Many projects use both — `thiserror` for the library crate's public API, `anyhow` in the `main()` binary.

### 错误恢复模式

C# developers are used to `try/catch` blocks that recover from specific exceptions. Rust uses combinators on `Result` for the same purpose:

```rust
use std::fs;

// Pattern 1: Recover with a fallback value
let config = fs::read_to_string("config.toml")
    .unwrap_or_else(|_| String::from("port = 8080"));  // default if missing

// Pattern 2: Recover from specific errors, propagate others
fn read_or_create(path: &str) -> Result<String, std::io::Error> {
    match fs::read_to_string(path) {
        Ok(content) => Ok(content),
        Err(e) if e.kind() == std::io::ErrorKind::NotFound => {
            let default = String::from("# new file");
            fs::write(path, &default)?;
            Ok(default)
        }
        Err(e) => Err(e),  // propagate permission errors, etc.
    }
}

// Pattern 3: Add context before propagating
use anyhow::Context;

fn load_config() -> anyhow::Result<Config> {
    let text = fs::read_to_string("config.toml")
        .context("Failed to read config.toml")?;
    let config: Config = toml::from_str(&text)
        .context("Failed to parse config.toml")?;
    Ok(config)
}

// Pattern 4: Map errors to your domain type
fn parse_port(s: &str) -> Result<u16, AppError> {
    s.parse::<u16>()
        .map_err(|_| AppError::Validation {
            message: format!("Invalid port: {s}"),
        })
}
```

```csharp
// C# equivalents:
try { config = File.ReadAllText("config.toml"); }
catch (FileNotFoundException) { config = "port = 8080"; }  // Pattern 1

try { /* ... */ }
catch (FileNotFoundException) { /* create file */ }        // Pattern 2
catch { throw; }                                            // re-throw others
```

**何时恢复 vs 传播：**
- **恢复**——当错误有合理的默认值或重试策略时
- **用`?`传播**——当*调用者*应该决定怎么做时
- **添加上下文**（`.context()`）——在模块边界构建错误追踪

---

## 练习

<details>
<summary><strong>🏋️ 练习：设计Crate错误类型</strong>（点击展开）</summary>

你正在构建一个用户注册服务。使用`thiserror`设计错误类型：

1. Define `RegistrationError` with variants: `DuplicateEmail(String)`, `WeakPassword(String)`, `DatabaseError(#[from] sqlx::Error)`, `RateLimited { retry_after_secs: u64 }`
2. Create a `type Result<T> = std::result::Result<T, RegistrationError>;` alias
3. Write a `register_user(email: &str, password: &str) -> Result<()>` that demonstrates `?` propagation and explicit error construction

<details>
<summary>🔑 Solution</summary>

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum RegistrationError {
    #[error("Email already registered: {0}")]
    DuplicateEmail(String),

    #[error("Password too weak: {0}")]
    WeakPassword(String),

    #[error("Database error")]
    Database(#[from] sqlx::Error),

    #[error("Rate limited — retry after {retry_after_secs}s")]
    RateLimited { retry_after_secs: u64 },
}

pub type Result<T> = std::result::Result<T, RegistrationError>;

pub fn register_user(email: &str, password: &str) -> Result<()> {
    if password.len() < 8 {
        return Err(RegistrationError::WeakPassword(
            "must be at least 8 characters".into(),
        ));
    }

    // This ? converts sqlx::Error → RegistrationError::Database automatically
    // db.check_email_unique(email).await?;

    // This is explicit construction for domain logic
    if email.contains("+spam") {
        return Err(RegistrationError::DuplicateEmail(email.to_string()));
    }

    Ok(())
}
```

**关键模式**：`#[from]`启用`?`用于库错误；显式`Err(...)`用于领域逻辑。Result别名保持每个签名整洁。

</details>
</details>

***


