## 元组和解构

> **学习内容：** Rust元组 vs C#的`ValueTuple`、数组和切片、结构体 vs 类、
> 用于零成本类型安全领域建模的newtype模式，以及解构语法。
>
> **难度：** 🟢 初级

C#有`ValueTuple`（自C# 7起）。Rust元组类似，但与语言集成得更深入。

### C# 元组
```csharp
// C# ValueTuple (C# 7+)
var point = (10, 20);                         // (int, int)
var named = (X: 10, Y: 20);                   // Named elements
Console.WriteLine($"{named.X}, {named.Y}");

// Tuple as return type
public (int Quotient, int Remainder) Divide(int a, int b)
{
    return (a / b, a % b);
}

var (q, r) = Divide(10, 3);    // Deconstruction
Console.WriteLine($"{q} remainder {r}");

// Discards
var (_, remainder) = Divide(10, 3);  // Ignore quotient
```

### Rust 元组
```rust
// Rust tuples — immutable by default, no named elements
let point = (10, 20);                // (i32, i32)
let point3d: (f64, f64, f64) = (1.0, 2.0, 3.0);

// Access by index (0-based)
println!("x={}, y={}", point.0, point.1);

// Tuple as return type
fn divide(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

let (q, r) = divide(10, 3);       // Destructuring
println!("{q} remainder {r}");

// Discards with _
let (_, remainder) = divide(10, 3);

// Unit type () — the "empty tuple" (like C# void)
fn greet() {          // implicit return type is ()
    println!("hi");
}
```

### 关键差异

| Feature | C# `ValueTuple` | Rust Tuple |
|---------|-----------------|------------|
| Named elements | `(int X, int Y)` | Not supported — use structs |
| Max arity | ~8 (nesting for more) | Unlimited (practical limit ~12) |
| Comparisons | Automatic | Automatic for tuples ≤ 12 elements |
| Used as dict key | Yes | Yes (if elements implement `Hash`) |
| Return from functions | Common | Common |
| Mutable elements | Always mutable | Only with `let mut` |

### 元组结构体（Newtypes）
```rust
// When a plain tuple isn't descriptive enough, use a tuple struct:
struct Meters(f64);     // Single-field "newtype" wrapper
struct Celsius(f64);
struct Fahrenheit(f64);

// The compiler treats these as DIFFERENT types:
let distance = Meters(100.0);
let temp = Celsius(36.6);
// distance == temp;  // ❌ ERROR: can't compare Meters with Celsius

// Newtype pattern prevents unit-confusion bugs at compile time!
// In C# you'd need a full class/struct for the same safety.
```

```csharp
// C# equivalent requires more ceremony:
public readonly record struct Meters(double Value);
public readonly record struct Celsius(double Value);
// Not interchangeable, but records add overhead vs Rust's zero-cost newtypes
```

### Newtype模式深入：零成本领域建模

Newtypes远不止防止单位混淆。它们是Rust将**业务规则编码到类型系统**的主要工具——取代了C#中常见的"守卫子句"和"验证类"模式。

#### C# 验证方法：运行时守卫
```csharp
// C# — validation happens at runtime, every time
public class UserService
{
    public User CreateUser(string email, int age)
    {
        if (string.IsNullOrWhiteSpace(email) || !email.Contains('@'))
            throw new ArgumentException("Invalid email");
        if (age < 0 || age > 150)
            throw new ArgumentException("Invalid age");

        return new User { Email = email, Age = age };
    }

    public void SendEmail(string email)
    {
        // Must re-validate — or trust the caller?
        if (!email.Contains('@')) throw new ArgumentException("Invalid email");
        // ...
    }
}
```

#### Rust Newtype方法：编译时证明
```rust
/// A validated email address — the type itself IS the proof of validity.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Email(String);

impl Email {
    /// The ONLY way to create an Email — validation happens once at construction.
    pub fn new(raw: &str) -> Result<Self, &'static str> {
        if raw.contains('@') && raw.len() > 3 {
            Ok(Email(raw.to_lowercase()))
        } else {
            Err("invalid email format")
        }
    }

    /// Safe access to the inner value
    pub fn as_str(&self) -> &str { &self.0 }
}

/// A validated age — impossible to create an invalid one.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct Age(u8);

impl Age {
    pub fn new(raw: u8) -> Result<Self, &'static str> {
        if raw <= 150 { Ok(Age(raw)) } else { Err("age out of range") }
    }
    pub fn value(&self) -> u8 { self.0 }
}

// Now functions take PROVEN types — no re-validation needed!
fn create_user(email: Email, age: Age) -> User {
    // email is GUARANTEED valid — it's a type invariant
    User { email, age }
}

fn send_email(to: &Email) {
    // No validation needed — Email type proves validity
    println!("Sending to: {}", to.as_str());
}
```

#### C#开发者的常见Newtype用途

| C# Pattern | Rust Newtype | What It Prevents |
|------------|-------------|------------------|
| `string` for UserId, Email, etc. | `struct UserId(Uuid)` | Passing wrong string to wrong parameter |
| `int` for Port, Count, Index | `struct Port(u16)` | Port and Count are not interchangeable |
| Guard clauses everywhere | Constructor validation once | Re-validation, missed validation |
| `decimal` for USD, EUR | `struct Usd(Decimal)` | Adding USD to EUR by accident |
| `TimeSpan` for different semantics | `struct Timeout(Duration)` | Passing connection timeout as request timeout |

```rust
// 零成本：newtypes编译后与内部类型生成相同的汇编代码。
// 这段Rust代码：
struct UserId(u64);
fn lookup(id: UserId) -> Option<User> { /* ... */ }

// 生成与以下代码相同的机器码：
fn lookup(id: u64) -> Option<User> { /* ... */ }
// 但具有编译时的完整类型安全！
```

***

## 数组和切片

理解数组、切片和向量之间的区别至关重要。

### C# 数组
```csharp
// C# arrays
int[] numbers = new int[5];         // Fixed size, heap allocated
int[] initialized = { 1, 2, 3, 4, 5 }; // Array literal

// Access
numbers[0] = 10;
int first = numbers[0];

// Length
int length = numbers.Length;

// Array as parameter (reference type)
void ProcessArray(int[] array)
{
    array[0] = 99;  // Modifies original
}
```

### Rust 数组、切片和向量
```rust
// 1. Arrays - Fixed size, stack allocated
let numbers: [i32; 5] = [1, 2, 3, 4, 5];  // Type: [i32; 5]
let zeros = [0; 10];                       // 10 zeros

// Access
let first = numbers[0];
// numbers[0] = 10;  // ❌ Error: arrays are immutable by default

let mut mut_array = [1, 2, 3, 4, 5];
mut_array[0] = 10;  // ✅ Works with mut

// 2. Slices - Views into arrays or vectors
let slice: &[i32] = &numbers[1..4];  // Elements 1, 2, 3
let all_slice: &[i32] = &numbers;    // Entire array as slice

// 3. Vectors - Dynamic size, heap allocated (covered earlier)
let mut vec = vec![1, 2, 3, 4, 5];
vec.push(6);  // Can grow
```

### 切片作为函数参数
```csharp
// C# - Method that works with arrays
public void ProcessNumbers(int[] numbers)
{
    for (int i = 0; i < numbers.Length; i++)
    {
        Console.WriteLine(numbers[i]);
    }
}

// Works with arrays only
ProcessNumbers(new int[] { 1, 2, 3 });
```

```rust
// Rust - Function that works with any sequence
fn process_numbers(numbers: &[i32]) {  // Slice parameter
    for (i, num) in numbers.iter().enumerate() {
        println!("Index {}: {}", i, num);
    }
}

fn main() {
    let array = [1, 2, 3, 4, 5];
    let vec = vec![1, 2, 3, 4, 5];
    
    // Same function works with both!
    process_numbers(&array);      // Array as slice
    process_numbers(&vec);        // Vector as slice
    process_numbers(&vec[1..4]);  // Partial slice
}
```

### 字符串切片（&str）再探
```rust
// String and &str relationship
fn string_slice_example() {
    let owned = String::from("Hello, World!");
    let slice: &str = &owned[0..5];      // "Hello"
    let slice2: &str = &owned[7..];      // "World!"
    
    println!("{}", slice);   // "Hello"
    println!("{}", slice2);  // "World!"
    
    // Function that accepts any string type
    print_string("String literal");      // &str
    print_string(&owned);               // String as &str
    print_string(slice);                // &str slice
}

fn print_string(s: &str) {
    println!("{}", s);
}
```

***

## 结构体 vs 类

Rust中的结构体与C#中的类类似，但在所有权和方法方面有一些关键差异。

```mermaid
graph TD
    subgraph "C# Class (Heap)"
        CObj["Object Header\n+ vtable ptr"] --> CFields["Name: string ref\nAge: int\nHobbies: List ref"]
        CFields --> CHeap1["\"Alice\" on heap"]
        CFields --> CHeap2["List&lt;string&gt; on heap"]
    end
    subgraph "Rust Struct (Stack)"
        RFields["name: String\n  ptr | len | cap\nage: i32\nhobbies: Vec\n  ptr | len | cap"]
        RFields --> RHeap1["\"Alice\" heap buffer"]
        RFields --> RHeap2["Vec heap buffer"]
    end

    style CObj fill:#bbdefb,color:#000
    style RFields fill:#c8e6c9,color:#000
```

> **关键洞察**：C#类始终在引用背后存在于堆上。Rust结构体默认位于栈上——只有动态大小的数据（如`String`内容）会进入堆。这消除了小型、频繁创建对象的GC开销。

### C# 类定义
```csharp
// C# class with properties and methods
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
    public List<string> Hobbies { get; set; }
    
    public Person(string name, int age)
    {
        Name = name;
        Age = age;
        Hobbies = new List<string>();
    }
    
    public void AddHobby(string hobby)
    {
        Hobbies.Add(hobby);
    }
    
    public string GetInfo()
    {
        return $"{Name} is {Age} years old";
    }
}
```

### Rust 结构体定义
```rust
// Rust struct with associated functions and methods
#[derive(Debug)]  // Automatically implement Debug trait
pub struct Person {
    pub name: String,    // Public field
    pub age: u32,        // Public field
    hobbies: Vec<String>, // Private field (no pub)
}

impl Person {
    // Associated function (like static method)
    pub fn new(name: String, age: u32) -> Person {
        Person {
            name,
            age,
            hobbies: Vec::new(),
        }
    }
    
    // Method (takes &self, &mut self, or self)
    pub fn add_hobby(&mut self, hobby: String) {
        self.hobbies.push(hobby);
    }
    
    // Method that borrows immutably
    pub fn get_info(&self) -> String {
        format!("{} is {} years old", self.name, self.age)
    }
    
    // Getter for private field
    pub fn hobbies(&self) -> &Vec<String> {
        &self.hobbies
    }
}
```

### 创建和使用实例
```csharp
// C# object creation and usage
var person = new Person("Alice", 30);
person.AddHobby("Reading");
person.AddHobby("Swimming");

Console.WriteLine(person.GetInfo());
Console.WriteLine($"Hobbies: {string.Join(", ", person.Hobbies)}");

// Modify properties directly
person.Age = 31;
```

```rust
// Rust struct creation and usage
let mut person = Person::new("Alice".to_string(), 30);
person.add_hobby("Reading".to_string());
person.add_hobby("Swimming".to_string());

println!("{}", person.get_info());
println!("Hobbies: {:?}", person.hobbies());

// Modify public fields directly
person.age = 31;

// Debug print the entire struct
println!("{:?}", person);
```

### 结构体初始化模式
```csharp
// C# object initialization
var person = new Person("Bob", 25)
{
    Hobbies = new List<string> { "Gaming", "Coding" }
};

// Anonymous types
var anonymous = new { Name = "Charlie", Age = 35 };
```

```rust
// Rust struct initialization
let person = Person {
    name: "Bob".to_string(),
    age: 25,
    hobbies: vec!["Gaming".to_string(), "Coding".to_string()],
};

// Struct update syntax (like object spread)
let older_person = Person {
    age: 26,
    ..person  // Use remaining fields from person (moves person!)
};

// Tuple structs (like anonymous types)
#[derive(Debug)]
struct Point(i32, i32);

let point = Point(10, 20);
println!("Point: ({}, {})", point.0, point.1);
```

***

## 方法和关联函数

理解方法和关联函数之间的区别是关键。

### C# 方法类型
```csharp
public class Calculator
{
    private int memory = 0;
    
    // Instance method
    public int Add(int a, int b)
    {
        return a + b;
    }
    
    // Instance method that uses state
    public void StoreInMemory(int value)
    {
        memory = value;
    }
    
    // Static method
    public static int Multiply(int a, int b)
    {
        return a * b;
    }
    
    // Static factory method
    public static Calculator CreateWithMemory(int initialMemory)
    {
        var calc = new Calculator();
        calc.memory = initialMemory;
        return calc;
    }
}
```

### Rust 方法类型
```rust
#[derive(Debug)]
pub struct Calculator {
    memory: i32,
}

impl Calculator {
    // Associated function (like static method) - no self parameter
    pub fn new() -> Calculator {
        Calculator { memory: 0 }
    }
    
    // Associated function with parameters
    pub fn with_memory(initial_memory: i32) -> Calculator {
        Calculator { memory: initial_memory }
    }
    
    // Method that borrows immutably (&self)
    pub fn add(&self, a: i32, b: i32) -> i32 {
        a + b
    }
    
    // Method that borrows mutably (&mut self)
    pub fn store_in_memory(&mut self, value: i32) {
        self.memory = value;
    }
    
    // Method that takes ownership (self)
    pub fn into_memory(self) -> i32 {
        self.memory  // Calculator is consumed
    }
    
    // Getter method
    pub fn memory(&self) -> i32 {
        self.memory
    }
}

fn main() {
    // Associated functions called with ::
    let mut calc = Calculator::new();
    let calc2 = Calculator::with_memory(42);
    
    // Methods called with .
    let result = calc.add(5, 3);
    calc.store_in_memory(result);
    
    println!("Memory: {}", calc.memory());
    
    // Consuming method
    let memory_value = calc.into_memory();  // calc is no longer usable
    println!("Final memory: {}", memory_value);
}
```

### 方法接收器类型详解
```rust
impl Person {
    // &self - Immutable borrow (most common)
    // Use when you only need to read the data
    pub fn get_name(&self) -> &str {
        &self.name
    }
    
    // &mut self - Mutable borrow
    // Use when you need to modify the data
    pub fn set_name(&mut self, name: String) {
        self.name = name;
    }
    
    // self - Take ownership (less common)
    // Use when you want to consume the struct
    pub fn consume(self) -> String {
        self.name  // Person is moved, no longer accessible
    }
}

fn method_examples() {
    let mut person = Person::new("Alice".to_string(), 30);
    
    // Immutable borrow
    let name = person.get_name();  // person can still be used
    println!("Name: {}", name);
    
    // Mutable borrow
    person.set_name("Alice Smith".to_string());  // person can still be used
    
    // Taking ownership
    let final_name = person.consume();  // person is no longer usable
    println!("Final name: {}", final_name);
}
```

---

## 练习

<details>
<summary><strong>🏋️ 练习：滑动窗口平均值</strong>（点击展开）</summary>

**挑战**：编写一个函数，接受一个`f64`值的切片和一个窗口大小，返回一个滚动平均值的`Vec<f64>`。例如，`[1.0, 2.0, 3.0, 4.0, 5.0]`窗口为3时 → `[2.0, 3.0, 4.0]`。

```rust
fn rolling_average(data: &[f64], window: usize) -> Vec<f64> {
    // Your implementation here
    todo!()
}

fn main() {
    let data = vec![1.0, 2.0, 3.0, 4.0, 5.0];
    let avgs = rolling_average(&data, 3);
    println!("{avgs:?}"); // [2.0, 3.0, 4.0]
}
```

<details>
<summary>🔑 Solution</summary>

```rust
fn rolling_average(data: &[f64], window: usize) -> Vec<f64> {
    data.windows(window)
        .map(|w| w.iter().sum::<f64>() / w.len() as f64)
        .collect()
}

fn main() {
    let data = vec![1.0, 2.0, 3.0, 4.0, 5.0];
    let avgs = rolling_average(&data, 3);
    assert_eq!(avgs, vec![2.0, 3.0, 4.0]);
    println!("{avgs:?}");
}
```

**关键要点**：切片有强大的内置方法，如`.windows()`、`.chunks()`和`.split()`，它们取代了手动索引运算。在C#中，你需要使用`Enumerable.Range`或LINQ的`.Skip().Take()`。

</details>
</details>

<details>
<summary><strong>🏋️ 练习：迷你通讯录</strong>（点击展开）</summary>

使用结构体、枚举和方法构建一个小型通讯录：

1. Define an enum `PhoneType { Mobile, Home, Work }`
2. Define a struct `Contact` with `name: String` and `phones: Vec<(PhoneType, String)>`
3. Implement `Contact::new(name: impl Into<String>) -> Self`
4. Implement `Contact::add_phone(&mut self, kind: PhoneType, number: impl Into<String>)`
5. Implement `Contact::mobile_numbers(&self) -> Vec<&str>` that returns only mobile numbers
6. In `main`, create a contact, add two phones, and print the mobile numbers

<details>
<summary>🔑 Solution</summary>

```rust
#[derive(Debug, PartialEq)]
enum PhoneType { Mobile, Home, Work }

#[derive(Debug)]
struct Contact {
    name: String,
    phones: Vec<(PhoneType, String)>,
}

impl Contact {
    fn new(name: impl Into<String>) -> Self {
        Contact { name: name.into(), phones: Vec::new() }
    }

    fn add_phone(&mut self, kind: PhoneType, number: impl Into<String>) {
        self.phones.push((kind, number.into()));
    }

    fn mobile_numbers(&self) -> Vec<&str> {
        self.phones
            .iter()
            .filter(|(kind, _)| *kind == PhoneType::Mobile)
            .map(|(_, num)| num.as_str())
            .collect()
    }
}

fn main() {
    let mut alice = Contact::new("Alice");
    alice.add_phone(PhoneType::Mobile, "+1-555-0100");
    alice.add_phone(PhoneType::Work, "+1-555-0200");
    alice.add_phone(PhoneType::Mobile, "+1-555-0101");

    println!("{}'s mobile numbers: {:?}", alice.name, alice.mobile_numbers());
}
```

</details>
</details>

***


