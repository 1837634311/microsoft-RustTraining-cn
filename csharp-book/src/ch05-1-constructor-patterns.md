## 构造函数模式

> **学习内容：** 如何在没有传统构造函数的情况下创建Rust结构体——`new()`约定、
> `Default` trait、工厂方法，以及用于复杂初始化的构建器模式。
>
> **难度：** 🟢 初级

### C# 构造函数模式
```csharp
public class Configuration
{
    public string DatabaseUrl { get; set; }
    public int MaxConnections { get; set; }
    public bool EnableLogging { get; set; }
    
    // Default constructor
    public Configuration()
    {
        DatabaseUrl = "localhost";
        MaxConnections = 10;
        EnableLogging = false;
    }
    
    // Parameterized constructor
    public Configuration(string databaseUrl, int maxConnections)
    {
        DatabaseUrl = databaseUrl;
        MaxConnections = maxConnections;
        EnableLogging = false;
    }
    
    // Factory method
    public static Configuration ForProduction()
    {
        return new Configuration("prod.db.server", 100)
        {
            EnableLogging = true
        };
    }
}
```

### Rust 构造函数模式
```rust
#[derive(Debug)]
pub struct Configuration {
    pub database_url: String,
    pub max_connections: u32,
    pub enable_logging: bool,
}

impl Configuration {
    // Default constructor
    pub fn new() -> Configuration {
        Configuration {
            database_url: "localhost".to_string(),
            max_connections: 10,
            enable_logging: false,
        }
    }
    
    // Parameterized constructor
    pub fn with_database(database_url: String, max_connections: u32) -> Configuration {
        Configuration {
            database_url,
            max_connections,
            enable_logging: false,
        }
    }
    
    // Factory method
    pub fn for_production() -> Configuration {
        Configuration {
            database_url: "prod.db.server".to_string(),
            max_connections: 100,
            enable_logging: true,
        }
    }
    
    // Builder pattern method
    pub fn enable_logging(mut self) -> Configuration {
        self.enable_logging = true;
        self  // Return self for chaining
    }
    
    pub fn max_connections(mut self, count: u32) -> Configuration {
        self.max_connections = count;
        self
    }
}

// Default trait implementation
impl Default for Configuration {
    fn default() -> Self {
        Self::new()
    }
}

fn main() {
    // Different construction patterns
    let config1 = Configuration::new();
    let config2 = Configuration::with_database("localhost:5432".to_string(), 20);
    let config3 = Configuration::for_production();
    
    // Builder pattern
    let config4 = Configuration::new()
        .enable_logging()
        .max_connections(50);
    
    // Using Default trait
    let config5 = Configuration::default();
    
    println!("{:?}", config4);
}
```

### 构建器模式实现
```rust
// More complex builder pattern
#[derive(Debug)]
pub struct DatabaseConfig {
    host: String,
    port: u16,
    username: String,
    password: Option<String>,
    ssl_enabled: bool,
    timeout_seconds: u64,
}

pub struct DatabaseConfigBuilder {
    host: Option<String>,
    port: Option<u16>,
    username: Option<String>,
    password: Option<String>,
    ssl_enabled: bool,
    timeout_seconds: u64,
}

impl DatabaseConfigBuilder {
    pub fn new() -> Self {
        DatabaseConfigBuilder {
            host: None,
            port: None,
            username: None,
            password: None,
            ssl_enabled: false,
            timeout_seconds: 30,
        }
    }
    
    pub fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }
    
    pub fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }
    
    pub fn username(mut self, username: impl Into<String>) -> Self {
        self.username = Some(username.into());
        self
    }
    
    pub fn password(mut self, password: impl Into<String>) -> Self {
        self.password = Some(password.into());
        self
    }
    
    pub fn enable_ssl(mut self) -> Self {
        self.ssl_enabled = true;
        self
    }
    
    pub fn timeout(mut self, seconds: u64) -> Self {
        self.timeout_seconds = seconds;
        self
    }
    
    pub fn build(self) -> Result<DatabaseConfig, String> {
        let host = self.host.ok_or("Host is required")?;
        let port = self.port.ok_or("Port is required")?;
        let username = self.username.ok_or("Username is required")?;
        
        Ok(DatabaseConfig {
            host,
            port,
            username,
            password: self.password,
            ssl_enabled: self.ssl_enabled,
            timeout_seconds: self.timeout_seconds,
        })
    }
}

fn main() {
    let config = DatabaseConfigBuilder::new()
        .host("localhost")
        .port(5432)
        .username("admin")
        .password("secret123")
        .enable_ssl()
        .timeout(60)
        .build()
        .expect("Failed to build config");
    
    println!("{:?}", config);
}
```

---

## 练习

<details>
<summary><strong>🏋️ 练习：带验证的构建器</strong>（点击展开）</summary>

创建一个`EmailBuilder`，满足以下条件：
1. 要求`to`和`subject`（构建器没有它们无法编译——使用类型状态或在`build()`中验证）
2. 可选的`body`和`cc`（地址Vec）
3. `build()`返回`Result<Email, String>`——拒绝空的`to`或`subject`
4. 编写测试证明无效输入被拒绝

<details>
<summary>🔑 Solution</summary>

```rust
#[derive(Debug)]
struct Email {
    to: String,
    subject: String,
    body: Option<String>,
    cc: Vec<String>,
}

#[derive(Default)]
struct EmailBuilder {
    to: Option<String>,
    subject: Option<String>,
    body: Option<String>,
    cc: Vec<String>,
}

impl EmailBuilder {
    fn new() -> Self { Self::default() }

    fn to(mut self, to: impl Into<String>) -> Self {
        self.to = Some(to.into()); self
    }
    fn subject(mut self, subject: impl Into<String>) -> Self {
        self.subject = Some(subject.into()); self
    }
    fn body(mut self, body: impl Into<String>) -> Self {
        self.body = Some(body.into()); self
    }
    fn cc(mut self, addr: impl Into<String>) -> Self {
        self.cc.push(addr.into()); self
    }
    fn build(self) -> Result<Email, String> {
        let to = self.to.filter(|s| !s.is_empty())
            .ok_or("'to' is required")?;
        let subject = self.subject.filter(|s| !s.is_empty())
            .ok_or("'subject' is required")?;
        Ok(Email { to, subject, body: self.body, cc: self.cc })
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    #[test]
    fn valid_email() {
        let email = EmailBuilder::new()
            .to("alice@example.com")
            .subject("Hello")
            .build();
        assert!(email.is_ok());
    }
    #[test]
    fn missing_to_fails() {
        let email = EmailBuilder::new().subject("Hello").build();
        assert!(email.is_err());
    }
}
```

</details>
</details>

***


