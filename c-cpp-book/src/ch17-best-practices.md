# Rust 最佳实践总结

> **你将学到什么：** 编写惯用 Rust 的实用指南——代码组织、命名约定、错误处理模式和文档。这是一个你会经常返回的快速参考章节。

## 代码组织
- **优先使用小函数**：易于测试和推理
- **使用描述性名称**：`calculate_total_price()` vs `calc()`
- **分组相关功能**：使用模块和单独文件
- **编写文档**：为公共 API 使用 `///`

## 错误处理
- **除非确信不会失败，否则避免 `unwrap()`**：只有当你 100% 确定它不会 panic 时才使用
```rust
// Bad: Can panic
let value = some_option.unwrap();

// Good: Handle the None case
let value = some_option.unwrap_or(default_value);
let value = some_option.unwrap_or_else(|| expensive_computation());
let value = some_option.unwrap_or_default(); // Uses Default trait

// For Result<T, E>
let value = some_result.unwrap_or(fallback_value);
let value = some_result.unwrap_or_else(|err| {
    eprintln!("Error occurred: {err}");
    default_value
});
```
- **使用带有描述性消息的 `expect()`**：当 unwrap 是合理的时，解释原因
```rust
let config = std::env::var("CONFIG_PATH")
    .expect("CONFIG_PATH environment variable must be set");
```
- **对可能失败的操作返回 `Result<T, E>`**：让调用者决定如何处理错误
- **使用 `thiserror` 定义自定义错误类型**：比手动实现更符合人体工程学
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Parse error: {message}")]
    Parse { message: String },
    
    #[error("Value {value} is out of range")]
    OutOfRange { value: i32 },
}
```
- **用 `?` 操作符链接错误**：将错误向上传播到调用栈
- **优先使用 `thiserror` 而不是 `anyhow`**：我们的团队惯例是用 `#[derive(thiserror::Error)]` 定义显式错误枚举，以便调用者可以匹配特定变体。`anyhow::Error` 对于快速原型制作很方便，但会擦除错误类型，使调用者更难处理特定失败。为库和生产代码使用 `thiserror`；为一次性脚本或只需要打印错误的最顶层二进制文件保留 `anyhow`。
- **何时 `unwrap()` 是可接受的**：
  - **单元测试**：`assert_eq!(result.unwrap(), expected)`
  - **原型制作**：你会替换的快速而粗糙的代码
  - **Infallible operations**: When you can prove it won't fail
```rust
let numbers = vec![1, 2, 3];
let first = numbers.get(0).unwrap(); // Safe: we just created the vec with elements

// Better: Use expect() with explanation
let first = numbers.get(0).expect("numbers vec is non-empty by construction");
```
- **Fail fast**: Check preconditions early and return errors immediately

## Memory Management
- **Prefer borrowing over cloning**: Use `&T` instead of cloning when possible
- **Use `Rc<T>` sparingly**: Only when you need shared ownership
- **Limit lifetimes**: Use scopes `{}` to control when values are dropped
- **Avoid `RefCell<T>` in public APIs**: Keep interior mutability internal

## Performance
- **Profile before optimizing**: Use `cargo bench` and profiling tools
- **Prefer iterators over loops**: More readable and often faster
- **Use `&str` over `String`**: When you don't need ownership
- **Consider `Box<T>` for large stack objects**: Move them to heap if needed

## Essential Traits to Implement

### Core Traits Every Type Should Consider

When creating custom types, consider implementing these fundamental traits to make your types feel native to Rust:

#### **Debug and Display**
```rust
use std::fmt;

#[derive(Debug)]  // Automatic implementation for debugging
struct Person {
    name: String,
    age: u32,
}

// Manual Display implementation for user-facing output
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{} (age {})", self.name, self.age)
    }
}

// Usage:
let person = Person { name: "Alice".to_string(), age: 30 };
println!("{:?}", person);  // Debug: Person { name: "Alice", age: 30 }
println!("{}", person);    // Display: Alice (age 30)
```

#### **Clone and Copy**
```rust
// Copy: Implicit duplication for small, simple types
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

// Clone: Explicit duplication for complex types
#[derive(Debug, Clone)]
struct Person {
    name: String,  // String doesn't implement Copy
    age: u32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;  // Copy (implicit)

let person1 = Person { name: "Bob".to_string(), age: 25 };
let person2 = person1.clone();  // Clone (explicit)
```

#### **PartialEq and Eq**
```rust
#[derive(Debug, PartialEq, Eq)]
struct UserId(u64);

#[derive(Debug, PartialEq)]
struct Temperature {
    celsius: f64,  // f64 doesn't implement Eq (due to NaN)
}

let id1 = UserId(123);
let id2 = UserId(123);
assert_eq!(id1, id2);  // Works because of PartialEq

let temp1 = Temperature { celsius: 20.0 };
let temp2 = Temperature { celsius: 20.0 };
assert_eq!(temp1, temp2);  // Works with PartialEq
```

#### **PartialOrd and Ord**
```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Priority(u8);

let high = Priority(1);
let low = Priority(10);
assert!(high < low);  // Lower numbers = higher priority

// Use in collections
let mut priorities = vec![Priority(5), Priority(1), Priority(8)];
priorities.sort();  // Works because Priority implements Ord
```

#### **Default**
```rust
#[derive(Debug, Default)]
struct Config {
    debug: bool,           // false (default)
    max_connections: u32,  // 0 (default)
    timeout: Option<u64>,  // None (default)
}

// Custom Default implementation
impl Default for Config {
    fn default() -> Self {
        Config {
            debug: false,
            max_connections: 100,  // Custom default
            timeout: Some(30),     // Custom default
        }
    }
}

let config = Config::default();
let config = Config { debug: true, ..Default::default() };  // Partial override
```

#### **From and Into**
```rust
struct UserId(u64);
struct UserName(String);

// Implement From, and Into comes for free
impl From<u64> for UserId {
    fn from(id: u64) -> Self {
        UserId(id)
    }
}

impl From<String> for UserName {
    fn from(name: String) -> Self {
        UserName(name)
    }
}

impl From<&str> for UserName {
    fn from(name: &str) -> Self {
        UserName(name.to_string())
    }
}

// Usage:
let user_id: UserId = 123u64.into();         // Using Into
let user_id = UserId::from(123u64);          // Using From
let username = UserName::from("alice");      // &str -> UserName
let username: UserName = "bob".into();       // Using Into
```

#### **TryFrom and TryInto**
```rust
use std::convert::TryFrom;

struct PositiveNumber(u32);

#[derive(Debug)]
struct NegativeNumberError;

impl TryFrom<i32> for PositiveNumber {
    type Error = NegativeNumberError;
    
    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value >= 0 {
            Ok(PositiveNumber(value as u32))
        } else {
            Err(NegativeNumberError)
        }
    }
}

// Usage:
let positive = PositiveNumber::try_from(42)?;     // Ok(PositiveNumber(42))
let error = PositiveNumber::try_from(-5);         // Err(NegativeNumberError)
```

#### **Serde (for serialization)**
```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

// Automatic JSON serialization/deserialization
let user = User {
    id: 1,
    name: "Alice".to_string(),
    email: "alice@example.com".to_string(),
};

let json = serde_json::to_string(&user)?;
let deserialized: User = serde_json::from_str(&json)?;
```

### Trait Implementation Checklist

For any new type, consider this checklist:

```rust
#[derive(
    Debug,          // [OK] Always implement for debugging
    Clone,          // [OK] If the type should be duplicatable
    PartialEq,      // [OK] If the type should be comparable
    Eq,             // [OK] If comparison is reflexive/transitive
    PartialOrd,     // [OK] If the type has ordering
    Ord,            // [OK] If ordering is total
    Hash,           // [OK] If type will be used as HashMap key
    Default,        // [OK] If there's a sensible default value
)]
struct MyType {
    // fields...
}

// Manual implementations to consider:
impl Display for MyType { /* user-facing representation */ }
impl From<OtherType> for MyType { /* convenient conversion */ }
impl TryFrom<FallibleType> for MyType { /* fallible conversion */ }
```

### When NOT to Implement Traits

- **Don't implement Copy for types with heap data**: `String`, `Vec`, `HashMap` etc.
- **Don't implement Eq if values can be NaN**: Types containing `f32`/`f64`
- **Don't implement Default if there's no sensible default**: File handles, network connections
- **Don't implement Clone if cloning is expensive**: Large data structures (consider `Rc<T>` instead)

### Summary: Trait Benefits

| Trait | Benefit | When to Use |
|-------|---------|-------------|
| `Debug` | `println!("{:?}", value)` | Always (except rare cases) |
| `Display` | `println!("{}", value)` | User-facing types |
| `Clone` | `value.clone()` | When explicit duplication makes sense |
| `Copy` | Implicit duplication | Small, simple types |
| `PartialEq` | `==` and `!=` operators | Most types |
| `Eq` | Reflexive equality | When equality is mathematically sound |
| `PartialOrd` | `<`, `>`, `<=`, `>=` | Types with natural ordering |
| `Ord` | `sort()`, `BinaryHeap` | When ordering is total |
| `Hash` | `HashMap` keys | Types used as map keys |
| `Default` | `Default::default()` | Types with obvious defaults |
| `From/Into` | Convenient conversions | Common type conversions |
| `TryFrom/TryInto` | Fallible conversions | Conversions that can fail |

----

----


