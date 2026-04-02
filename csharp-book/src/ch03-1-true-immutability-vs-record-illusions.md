## 真正的不可变性 vs 虚假的Record

> **学习内容：** 为什么C#的`record`类型并非真正不可变（可变字段、反射绕过），
> Rust如何在编译时强制实现真正的不可变性，以及何时使用内部可变性模式。
>
> **难度：** 🟡 中级

### C# Records - 不可变性表演
```csharp
// C# records look immutable but have escape hatches
public record Person(string Name, int Age, List<string> Hobbies);

var person = new Person("John", 30, new List<string> { "reading" });

// These all "look" like they create new instances:
var older = person with { Age = 31 };  // New record
var renamed = person with { Name = "Jonathan" };  // New record

// But the reference types are still mutable!
person.Hobbies.Add("gaming");  // Mutates the original!
Console.WriteLine(older.Hobbies.Count);  // 2 - older person affected!
Console.WriteLine(renamed.Hobbies.Count); // 2 - renamed person also affected!

// Init-only properties can still be set via reflection
typeof(Person).GetProperty("Age")?.SetValue(person, 25);

// Collection expressions help but don't solve the fundamental issue
public record BetterPerson(string Name, int Age, IReadOnlyList<string> Hobbies);

var betterPerson = new BetterPerson("Jane", 25, new List<string> { "painting" });
// Still mutable via casting: 
((List<string>)betterPerson.Hobbies).Add("hacking the system");

// Even "immutable" collections aren't truly immutable
using System.Collections.Immutable;
public record SafePerson(string Name, int Age, ImmutableList<string> Hobbies);
// This is better, but requires discipline and has performance overhead
```

### Rust - 默认的真正不可变性
```rust
#[derive(Debug, Clone)]
struct Person {
    name: String,
    age: u32,
    hobbies: Vec<String>,
}

let person = Person {
    name: "John".to_string(),
    age: 30,
    hobbies: vec!["reading".to_string()],
};

// This simply won't compile:
// person.age = 31;  // ERROR: cannot assign to immutable field
// person.hobbies.push("gaming".to_string());  // ERROR: cannot borrow as mutable

// To modify, you must explicitly opt-in with 'mut':
let mut older_person = person.clone();
older_person.age = 31;  // Now it's clear this is mutation

// Or use functional update patterns:
let renamed = Person {
    name: "Jonathan".to_string(),
    ..person  // Copies other fields (move semantics apply)
};

// The original is guaranteed unchanged (until moved):
println!("{:?}", person.hobbies);  // Always ["reading"] - immutable

// Structural sharing with efficient immutable data structures
use std::rc::Rc;

#[derive(Debug, Clone)]
struct EfficientPerson {
    name: String,
    age: u32,
    hobbies: Rc<Vec<String>>,  // Shared, immutable reference
}

// Creating new versions shares data efficiently
let person1 = EfficientPerson {
    name: "Alice".to_string(),
    age: 30,
    hobbies: Rc::new(vec!["reading".to_string(), "cycling".to_string()]),
};

let person2 = EfficientPerson {
    name: "Bob".to_string(),
    age: 25,
    hobbies: Rc::clone(&person1.hobbies),  // Shared reference, no deep copy
};
```

```mermaid
graph TD
    subgraph "C# Records - Shallow Immutability"
        CS_RECORD["record Person(...)"]
        CS_WITH["with expressions"]
        CS_SHALLOW["⚠️ Only top-level immutable"]
        CS_REF_MUT["❌ Reference types still mutable"]
        CS_REFLECTION["❌ Reflection can bypass"]
        CS_RUNTIME["❌ Runtime surprises"]
        CS_DISCIPLINE["😓 Requires team discipline"]
        
        CS_RECORD --> CS_WITH
        CS_WITH --> CS_SHALLOW
        CS_SHALLOW --> CS_REF_MUT
        CS_RECORD --> CS_REFLECTION
        CS_REF_MUT --> CS_RUNTIME
        CS_RUNTIME --> CS_DISCIPLINE
    end
    
    subgraph "Rust - True Immutability"
        RUST_STRUCT["struct Person { ... }"]
        RUST_DEFAULT["✅ Immutable by default"]
        RUST_COMPILE["✅ Compile-time enforcement"]
        RUST_MUT["🔒 Explicit 'mut' required"]
        RUST_MOVE["🔄 Move semantics"]
        RUST_ZERO["⚡ Zero runtime overhead"]
        RUST_SAFE["🛡️ Memory safe"]
        
        RUST_STRUCT --> RUST_DEFAULT
        RUST_DEFAULT --> RUST_COMPILE
        RUST_COMPILE --> RUST_MUT
        RUST_MUT --> RUST_MOVE
        RUST_MOVE --> RUST_ZERO
        RUST_ZERO --> RUST_SAFE
    end
    
    style CS_REF_MUT fill:#ffcdd2,color:#000
    style CS_REFLECTION fill:#ffcdd2,color:#000
    style CS_RUNTIME fill:#ffcdd2,color:#000
    style RUST_COMPILE fill:#c8e6c9,color:#000
    style RUST_ZERO fill:#c8e6c9,color:#000
    style RUST_SAFE fill:#c8e6c9,color:#000
```

---

## 练习

<details>
<summary><strong>🏋️ 练习：证明不可变性</strong>（点击展开）</summary>

一位C#同事声称他们的`record`是不可变的。将这段C#代码翻译成Rust并解释为什么Rust版本是真正不可变的：

```csharp
public record Config(string Host, int Port, List<string> AllowedOrigins);

var config = new Config("localhost", 8080, new List<string> { "example.com" });
// "Immutable" record... but:
config.AllowedOrigins.Add("evil.com"); // Compiles! List is mutable.
```

1. 创建一个等价的、**真正不可变**的Rust结构体
2. 证明尝试修改`allowed_origins`会导致**编译错误**
3. 编写一个函数，在不修改原值的情况下创建修改后的副本（新host）

<details>
<summary>🔑 Solution</summary>

```rust
#[derive(Debug, Clone)]
struct Config {
    host: String,
    port: u16,
    allowed_origins: Vec<String>,
}

impl Config {
    fn with_host(&self, host: impl Into<String>) -> Self {
        Config {
            host: host.into(),
            ..self.clone()
        }
    }
}

fn main() {
    let config = Config {
        host: "localhost".into(),
        port: 8080,
        allowed_origins: vec!["example.com".into()],
    };

    // config.allowed_origins.push("evil.com".into());
    // ❌ ERROR: cannot borrow `config.allowed_origins` as mutable

    let production = config.with_host("prod.example.com");
    println!("Dev: {:?}", config);       // original unchanged
    println!("Prod: {:?}", production);  // new copy with different host
}
```

**关键洞察**：在Rust中，`let config = ...`（没有`mut`）使*整个值树*不可变——包括嵌套的`Vec`。C# records只让*引用*不可变，而不是内容。

</details>
</details>

***


