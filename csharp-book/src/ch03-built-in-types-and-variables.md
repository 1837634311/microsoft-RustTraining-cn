## 变量和可变性

> **学习内容：** Rust的变量声明和可变模型 vs C#的 `var`/`const`、
> 基本类型映射、`String` vs `&str` 的关键区别、类型推断，
> 以及Rust如何不同地处理类型转换和强制类型转换。
>
> **难度：** 🟢 初级

### C# 变量声明
```csharp
// C# - 变量默认可变
int count = 0;           // 可变
count = 5;               // ✅ 可以

readonly int maxSize = 100;  // 初始化后不可变
// maxSize = 200;        // ❌ 编译错误

const int BUFFER_SIZE = 1024; // 编译时常量
```

### Rust 变量声明
```rust
// Rust - 变量默认不可变
let count = 0;           // 默认不可变
// count = 5;            // ❌ 编译错误：无法对不可变变量赋值两次

let mut count = 0;       // 显式可变
count = 5;               // ✅ 可以

const BUFFER_SIZE: usize = 1024; // 编译时常量
```

### C#开发者关键思维转变
```rust
// 把'let'当作默认的'readonly'
let name = "John";       // 类似: readonly string name = "John";
let mut age = 30;        // 类似: int age = 30;

// 变量遮蔽（Rust独有）
let spaces = "   ";      // 字符串
let spaces = spaces.len(); // 现在是数字（usize）
// 这与可变不同 - 我们创建了一个新变量
```

### 实践示例：计数器
```csharp
// C#版本
public class Counter
{
    private int value = 0;
    
    public void Increment()
    {
        value++;  // 可变
    }
    
    public int GetValue() => value;
}
```

```rust
// Rust版本
pub struct Counter {
    value: i32,  // 默认私有
}

impl Counter {
    pub fn new() -> Counter {
        Counter { value: 0 }
    }
    
    pub fn increment(&mut self) {  // 需要&mut来可变
        self.value += 1;
    }
    
    pub fn get_value(&self) -> i32 {
        self.value
    }
}
```

***

## 数据类型比较

### 基本类型

| C# 类型 | Rust 类型 | 大小 | 范围 |
|---------|-----------|------|-------|
| `byte` | `u8` | 8 bits | 0 到 255 |
| `sbyte` | `i8` | 8 bits | -128 到 127 |
| `short` | `i16` | 16 bits | -32,768 到 32,767 |
| `ushort` | `u16` | 16 bits | 0 到 65,535 |
| `int` | `i32` | 32 bits | -2³¹ 到 2³¹-1 |
| `uint` | `u32` | 32 bits | 0 到 2³²-1 |
| `long` | `i64` | 64 bits | -2⁶³ 到 2⁶³-1 |
| `ulong` | `u64` | 64 bits | 0 到 2⁶⁴-1 |
| `float` | `f32` | 32 bits | IEEE 754 |
| `double` | `f64` | 64 bits | IEEE 754 |
| `bool` | `bool` | 1 bit | true/false |
| `char` | `char` | 32 bits | Unicode标量 |

### 大小类型（重要！）
```csharp
// C# - int总是32位
int arrayIndex = 0;
long fileSize = file.Length;
```

```rust
// Rust - 大小类型匹配指针大小（32位或64位）
let array_index: usize = 0;    // 类似C中的size_t
let file_size: u64 = file.len(); // 显式64位
```

### 类型推断
```csharp
// C# - var关键字
var name = "John";        // string
var count = 42;           // int
var price = 29.99;        // double
```

```rust
// Rust - 自动类型推断
let name = "John";        // &str（字符串切片）
let count = 42;           // i32（默认整数类型）
let price = 29.99;        // f64（默认浮点类型）

// 显式类型注解
let count: u32 = 42;
let price: f32 = 29.99;
```

### 数组和集合概述
```csharp
// C# - 引用类型，堆分配
int[] numbers = new int[5];        // 固定大小
List<int> list = new List<int>();  // 动态大小
```

```rust
// Rust - 多种选项
let numbers: [i32; 5] = [1, 2, 3, 4, 5];  // 栈数组，固定大小
let mut list: Vec<i32> = Vec::new();       // 堆向量，动态大小
```

***

## 字符串类型：String vs &str

这是C#开发者最容易混淆的概念之一，让我们仔细分解。

### C# 字符串处理
```csharp
// C# - 简单字符串模型
string name = "John";           // 字符串字面量
string greeting = "Hello, " + name;  // 字符串拼接
string upper = name.ToUpper();  // 方法调用
```

### Rust 字符串类型
```rust
// Rust - 两个主要字符串类型

// 1. &str（字符串切片）- 类似C#的ReadOnlySpan<char>
let name: &str = "John";        // 字符串字面量（不可变，借用）

// 2. String - 类似StringBuilder或可变字符串
let mut greeting = String::new();       // 空字符串
greeting.push_str("Hello, ");          // 追加
greeting.push_str(name);               // 追加

// 或者直接创建
let greeting = String::from("Hello, John");
let greeting = "Hello, John".to_string();  // 将&str转换为String
```

### 何时使用哪种？

| 场景 | 使用 | C#等价物 |
|----------|-----|---------------|
| 字符串字面量 | `&str` | `string`字面量 |
| 函数参数（只读） | `&str` | `string`或`ReadOnlySpan<char>` |
| 拥有的可变字符串 | `String` | `StringBuilder` |
| 返回拥有的字符串 | `String` | `string` |

### 实践示例
```rust
// 接受任何字符串类型的函数
fn greet(name: &str) {  // 同时接受String和&str
    println!("Hello, {}!", name);
}

fn main() {
    let literal = "John";                    // &str
    let owned = String::from("Jane");        // String
    
    greet(literal);                          // 可以
    greet(&owned);                           // 可以（将String借用为&str）
    greet("Bob");                            // 可以
}

// 返回拥有的字符串的函数
fn create_greeting(name: &str) -> String {
    format!("Hello, {}!", name)  // format!宏返回String
}
```

### C#开发者：这样理解
```rust
// &str类似ReadOnlySpan<char> - 指向字符串数据的视图
// String类似char[] - 你拥有的可以修改的字符串

let borrowed: &str = "我不拥有这个数据";
let owned: String = String::from("我拥有这个数据");

// 相互转换
let owned_copy: String = borrowed.to_string();  // 复制到拥有的字符串
let borrowed_view: &str = &owned;               // 从拥有的字符串借用
```

***

## 打印和字符串格式化

C#开发者 heavily依赖 `Console.WriteLine` 和字符串插值（`$""`）。Rust的格式化系统同样强大，但使用宏和格式说明符。

### 基本输出
```csharp
// C#输出
Console.Write("no newline");
Console.WriteLine("with newline");
Console.Error.WriteLine("to stderr");

// 字符串插值（C# 6+）
string name = "Alice";
int age = 30;
Console.WriteLine($"{name} is {age} years old");
```

```rust
// Rust输出 — 全部是宏（注意那个！）
print!("no newline");              // → stdout，无换行
println!("with newline");           // → stdout + 换行
eprint!("to stderr");              // → stderr，无换行  
eprintln!("to stderr with newline"); // → stderr + 换行

// 字符串格式化（类似$""插值）
let name = "Alice";
let age = 30;
println!("{name} is {age} years old");     // 内联变量捕获（Rust 1.58+）
println!("{} is {} years old", name, age); // 位置参数

// format!返回String而不是打印
let msg = format!("{name} is {age} years old");
```

### 格式说明符
```csharp
// C#格式说明符
Console.WriteLine($"{price:F2}");         // 固定小数：  29.99
Console.WriteLine($"{count:D5}");         // 填充整数： 00042
Console.WriteLine($"{value,10}");         // 右对齐，宽度10
Console.WriteLine($"{value,-10}");        // 左对齐，宽度10
Console.WriteLine($"{hex:X}");            // 十六进制：    FF
Console.WriteLine($"{ratio:P1}");         // 百分比：     85.0%
```

```rust
// Rust格式说明符
println!("{price:.2}");          // 2位小数：  29.99
println!("{count:05}");          // 零填充，宽度5：00042
println!("{value:>10}");         // 右对齐，宽度10
println!("{value:<10}");         // 左对齐，宽度10
println!("{value:^10}");         // 居中对齐，宽度10
println!("{hex:#X}");            // 带前缀的十六进制：0xFF
println!("{hex:08X}");           // 零填充十六进制：000000FF
println!("{bits:#010b}");        // 带前缀的二进制：0b00001010
println!("{big}", big = 1_000_000); // 命名参数
```

### Debug vs Display打印
```rust
// {:?}  — Debug trait（面向开发者，自动派生）
// {:#?} — 格式化Debug（缩进，多行）
// {}    — Display trait（面向用户，必须手动实现）

#[derive(Debug)] // 自动生成Debug输出
struct Point { x: f64, y: f64 }

let p = Point { x: 1.5, y: 2.7 };

println!("{:?}", p);   // Point { x: 1.5, y: 2.7 }   — 紧凑debug
println!("{:#?}", p);  // Point {                     — 格式化debug
                        //     x: 1.5,
                        //     y: 2.7,
                        // }
// println!("{}", p);  // ❌ 错误：Point没有实现Display

// 为用户输出实现Display：
use std::fmt;

impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
println!("{}", p);    // (1.5, 2.7)  — 用户友好
```

```csharp
// C#等价物：
// {:?}  ≈ object.GetType().ToString()或反射输出
// {}    ≈ object.ToString()
// 在C#中重写ToString()；在Rust中实现Display
```

### 快速参考

| C# | Rust | 输出 |
|----|------|--------|
| `Console.WriteLine(x)` | `println!("{x}")` | Display格式化 |
| `$"{x}"`（插值） | `format!("{x}")` | 返回`String` |
| `x.ToString()` | `x.to_string()` | 需要`Display` trait |
| 重写`ToString()` | `impl Display` | 用户面向输出 |
| 调试器视图 | `{:?}` 或 `dbg!(x)` | 开发者输出 |
| `String.Format("{0:F2}", x)` | `format!("{x:.2}")` | 格式化的`String` |
| `Console.Error.WriteLine` | `eprintln!()` | 写入stderr |

***

## 类型转换和强制转换

C#有隐式转换、显式强制转换 `(int)x` 和 `Convert.To*()`。Rust更严格 — 没有隐式数值转换。

### 数值转换
```csharp
// C# — 隐式和显式转换
int small = 42;
long big = small;              // 隐式加宽：可以
double d = small;              // 隐式加宽：可以
int truncated = (int)3.14;     // 显式收窄：3
byte b = (byte)300;            // 静默溢出：44

// 安全转换
if (int.TryParse("42", out int parsed)) { /* ... */ }
```

```rust
// Rust — 所有数值转换都是显式的
let small: i32 = 42;
let big: i64 = small as i64;       // 加宽：使用'as'
let d: f64 = small as f64;         // 整数转浮点：显式
let truncated: i32 = 3.14_f64 as i32; // 收窄：3（截断）
let b: u8 = 300_u16 as u8;        // 溢出：回绕到44（类似C# unchecked）

// 使用TryFrom安全转换
use std::convert::TryFrom;
let safe: Result<u8, _> = u8::try_from(300_u16); // 错误 — 超出范围
let ok: Result<u8, _>   = u8::try_from(42_u16);  // Ok(42)

// 字符串解析 — 返回Result，不是bool + out参数
let parsed: Result<i32, _> = "42".parse::<i32>();   // Ok(42)
let bad: Result<i32, _>    = "abc".parse::<i32>();  // Err(ParseIntError)

// 使用turbofish语法：
let n = "42".parse::<f64>().unwrap(); // 42.0
```

### 字符串转换
```csharp
// C#
int n = 42;
string s = n.ToString();          // "42"
string formatted = $"{n:X}";
int back = int.Parse(s);          // 42或抛出异常
bool ok = int.TryParse(s, out int result);
```

```rust
// Rust — 通过Display的to_string()，通过FromStr的parse()
let n: i32 = 42;
let s: String = n.to_string();            // "42"（使用Display trait）
let formatted = format!("{n:X}");         // "2A"
let back: i32 = s.parse().unwrap();       // 42或panic
let result: Result<i32, _> = s.parse();   // Ok(42) — 安全版本

// &str ↔ String转换（Rust中最常见的转换）
let owned: String = "hello".to_string();    // &str → String
let owned2: String = String::from("hello"); // &str → String（等价）
let borrowed: &str = &owned;               // String → &str（免费，只是借用）
```

### 引用转换（无继承强制转换！）
```csharp
// C# — 向上转换和向下转换
Animal a = new Dog();              // 向上转换（隐式）
Dog d = (Dog)a;                    // 向下转换（显式，可能抛出）
if (a is Dog dog) { /* ... */ }    // 带模式匹配的安全向下转换
```

```rust
// Rust — 没有继承，没有向上/向下转换
// 使用trait对象实现多态：
let animal: Box<dyn Animal> = Box::new(Dog);

// "向下转换"需要Any trait（很少需要）：
use std::any::Any;
if let Some(dog) = animal_any.downcast_ref::<Dog>() {
    // 使用dog
}
// 实际上，使用枚举代替向下转换：
enum Animal {
    Dog(Dog),
    Cat(Cat),
}
match animal {
    Animal::Dog(d) => { /* 使用d */ }
    Animal::Cat(c) => { /* 使用c */ }
}
```

### 快速参考

| C# | Rust | 备注 |
|----|------|-------|
| `(int)x` | `x as i32` | 截断/回绕强制转换 |
| 隐式加宽 | 必须使用`as` | 无隐式数值转换 |
| `Convert.ToInt32(x)` | `i32::try_from(x)` | 安全，返回`Result` |
| `int.Parse(s)` | `s.parse::<i32>().unwrap()` | 失败时panic |
| `int.TryParse(s, out n)` | `s.parse::<i32>()` | 返回`Result<i32, _>` |
| `(Dog)animal` | 不可用 | 使用枚举或`Any` |
| `as Dog` / `is Dog` | `downcast_ref::<Dog>()` | 通过`Any` trait；优先使用枚举 |

***

## 注释和文档

### 常规注释
```csharp
// C#注释
// 单行注释
/* 多行
   注释 */

/// <summary>
/// XML文档注释
/// </summary>
/// <param name="name">用户的名字</param>
/// <returns>问候字符串</returns>
public string Greet(string name)
{
    return $"Hello, {name}!";
}
```

```rust
// Rust注释
// 单行注释
/* 多行
   注释 */

/// 文档注释（类似C#的///）
/// 此函数按名字问候用户。
/// 
/// # 参数
/// 
/// * `name` - 作为字符串切片的用户名
/// 
/// # 返回
/// 
/// 包含问候语的`String`
/// 
/// # 示例
/// 
/// ```
/// let greeting = greet("Alice");
/// assert_eq!(greeting, "Hello, Alice!");
/// ```
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

### 文档生成
```bash
# 生成文档（类似C#的XML文档）
cargo doc --open

# 运行文档测试
cargo test --doc
```

---

## 练习

<details>
<summary><strong>🏋️ 练习：类型安全的温度</strong>（点击展开）</summary>

创建一个Rust程序：
1. 声明一个绝对零度的摄氏温度`const`（`-273.15`）
2. 声明一个`static`计数器来记录已执行的转换次数（使用`AtomicU32`）
3. 编写函数`celsius_to_fahrenheit(c: f64) -> f64`，对于低于绝对零度的温度返回`f64::NAN`
4. 演示通过将字符串`"98.6"`解析为`f64`然后转换来使用变量遮蔽

<details>
<summary>🔑 解决方案</summary>

```rust
use std::sync::atomic::{AtomicU32, Ordering};

const ABSOLUTE_ZERO_C: f64 = -273.15;
static CONVERSION_COUNT: AtomicU32 = AtomicU32::new(0);

fn celsius_to_fahrenheit(c: f64) -> f64 {
    if c < ABSOLUTE_ZERO_C {
        return f64::NAN;
    }
    CONVERSION_COUNT.fetch_add(1, Ordering::Relaxed);
    c * 9.0 / 5.0 + 32.0
}

fn main() {
    let temp = "98.6";           // &str
    let temp: f64 = temp.parse().unwrap(); // 遮蔽为f64
    let temp = celsius_to_fahrenheit(temp); // 遮蔽为华氏温度
    println!("{temp:.1}°F");
    println!("Conversions: {}", CONVERSION_COUNT.load(Ordering::Relaxed));
}
```

</details>
</details>

***

