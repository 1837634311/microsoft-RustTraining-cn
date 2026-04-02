## C#开发者的Rust关键字速查

> **学习内容：** Rust关键字到C# equivalents的快速参考映射 — 可见性修饰符、所有权关键字、控制流、类型定义和模式匹配语法。
>
> **难度：** 🟢 初级

理解Rust的关键字及其用途有助于C#开发者更有效地驾驭这门语言。

### 可见性和访问控制关键字

#### C# 访问修饰符
```csharp
public class Example
{
    public int PublicField;           // 到处都可访问
    private int privateField;        // 仅此类内部
    protected int protectedField;    // 此类及子类
    internal int internalField;      // 程序集内部
    protected internal int protectedInternalField; // 组合
}
```

#### Rust 可见性关键字
```rust
// pub - 使项目公开（类似C#的public）
pub struct PublicStruct {
    pub public_field: i32,           // 公开字段
    private_field: i32,              // 默认私有（无关键字）
}

pub mod my_module {
    pub(crate) fn crate_public() {}     // 当前crate内公开（类似internal）
    pub(super) fn parent_public() {}    // 对父模块公开
    pub(self) fn self_public() {}       // 当前模块内公开（与私有相同）
    
    pub use super::PublicStruct;        // 重新导出（类似using别名）
}

// 没有直接等同于C# protected的方案 - 使用组合代替
```

### 内存和所有权关键字

#### C# 内存关键字
```csharp
// ref - 按引用传递
public void Method(ref int value) { value = 10; }

// out - 输出参数
public bool TryParse(string input, out int result) { /* */ }

// in - 只读引用（C# 7.2+）
public void ReadOnly(in LargeStruct data) { /* 无法修改data */ }
```

#### Rust 所有权关键字
```rust
// & - 不可变引用（类似C# in参数）
fn read_only(data: &Vec<i32>) {
    println!("Length: {}", data.len()); // 可以读取，无法修改
}

// &mut - 可变引用（类似C# ref参数）
fn modify(data: &mut Vec<i32>) {
    data.push(42); // 可以修改
}

// move - 强制移动捕获的闭包
let data = vec![1, 2, 3];
let closure = move || {
    println!("{:?}", data); // data被移入闭包
};
// data在这里不再可用

// Box - 堆分配（类似C# new创建引用类型）
let boxed_data = Box::new(42); // 在堆上分配
```

### 控制流关键字

#### C# 控制流
```csharp
// return - 带值退出函数
public int GetValue() { return 42; }

// yield return - 迭代器模式
public IEnumerable<int> GetNumbers()
{
    yield return 1;
    yield return 2;
}

// break/continue - 循环控制
foreach (var item in items)
{
    if (item == null) continue;
    if (item.Stop) break;
}
```

#### Rust 控制流关键字
```rust
// return - 显式返回（通常不需要）
fn get_value() -> i32 {
    return 42; // 显式返回
    // 或者: 42 （隐式返回）
}

// break/continue - 带可选值的循环控制
fn find_value() -> Option<i32> {
    loop {
        let value = get_next();
        if value < 0 { continue; }
        if value > 100 { break None; }      // 带值break
        if value == 42 { break Some(value); } // 带成功值break
    }
}

// loop - 无限循环（类似while(true)）
loop {
    if condition { break; }
}

// while - 条件循环
while condition {
    // code
}

// for - 迭代器循环
for item in collection {
    // code
}
```

### 类型定义关键字

#### C# 类型关键字
```csharp
// class - 引用类型
public class MyClass { }

// struct - 值类型
public struct MyStruct { }

// interface - 契约定义
public interface IMyInterface { }

// enum - 枚举
public enum MyEnum { Value1, Value2 }

// delegate - 函数指针
public delegate void MyDelegate(int value);
```

#### Rust 类型关键字
```rust
// struct - 数据结构（类似C# class/struct的结合）
struct MyStruct {
    field: i32,
}

// enum - 代数数据类型（比C# enum强大得多）
enum MyEnum {
    Variant1,
    Variant2(i32),              // 可以持有数据
    Variant3 { x: i32, y: i32 }, // 类似结构的变体
}

// trait - 接口定义（类似C# interface但更强大）
trait MyTrait {
    fn method(&self);
    
    // 默认实现（类似C# 8+默认接口方法）
    fn default_method(&self) {
        println!("Default implementation");
    }
}

// type - 类型别名（类似C# using别名）
type UserId = u32;
type Result<T> = std::result::Result<T, MyError>;

// impl - 实现块（无C#等价物 - 方法单独定义）
impl MyStruct {
    fn new() -> MyStruct {
        MyStruct { field: 0 }
    }
}

impl MyTrait for MyStruct {
    fn method(&self) {
        println!("Implementation");
    }
}
```

### 函数定义关键字

#### C# 函数关键字
```csharp
// static - 类方法
public static void StaticMethod() { }

// virtual - 可以被覆盖
public virtual void VirtualMethod() { }

// override - 覆盖基类方法
public override void VirtualMethod() { }

// abstract - 必须实现
public abstract void AbstractMethod();

// async - 异步方法
public async Task<int> AsyncMethod() { return await SomeTask(); }
```

#### Rust 函数关键字
```rust
// fn - 函数定义（类似C#方法但是独立的）
fn regular_function() {
    println!("Hello");
}

// const fn - 编译时函数（类似C# const但用于函数）
const fn compile_time_function() -> i32 {
    42 // 可以在编译时求值
}

// async fn - 异步函数（类似C# async）
async fn async_function() -> i32 {
    some_async_operation().await
}

// unsafe fn - 可能违反内存安全的函数
unsafe fn unsafe_function() {
    // 可以执行不安全操作
}

// extern fn - 外部函数接口
extern "C" fn c_compatible_function() {
    // 可以从C调用
}
```

### 变量声明关键字

#### C# 变量关键字
```csharp
// var - 类型推断
var name = "John"; // 推断为string

// const - 编译时常量
const int MaxSize = 100;

// readonly - 运行时常量
readonly DateTime createdAt = DateTime.Now;

// static - 类级变量
static int instanceCount = 0;
```

#### Rust 变量关键字
```rust
// let - 变量绑定（类似C# var）
let name = "John"; // 默认不可变

// let mut - 可变变量绑定
let mut count = 0; // 可以更改
count += 1;

// const - 编译时常量（类似C# const）
const MAX_SIZE: usize = 100;

// static - 全局变量（类似C# static）
static INSTANCE_COUNT: std::sync::atomic::AtomicUsize = 
    std::sync::atomic::AtomicUsize::new(0);
```

### 模式匹配关键字

#### C# 模式匹配（C# 8+）
```csharp
// switch表达式
string result = value switch
{
    1 => "One",
    2 => "Two",
    _ => "Other"
};

// is模式
if (obj is string str)
{
    Console.WriteLine(str.Length);
}
```

#### Rust 模式匹配关键字
```rust
// match - 模式匹配（类似C# switch但强大得多）
let result = match value {
    1 => "One",
    2 => "Two",
    3..=10 => "Between 3 and 10", // 范围模式
    _ => "Other", // 通配符（类似C# _）
};

// if let - 条件模式匹配
if let Some(value) = optional {
    println!("Got value: {}", value);
}

// while let - 带模式匹配的循环
while let Some(item) = iterator.next() {
    println!("Item: {}", item);
}

// let带模式 - 解构
let (x, y) = point; // 解构元组
let Some(value) = optional else {
    return; // 模式不匹配时提前返回
};
```

### 内存安全关键字

#### C# 内存关键字
```csharp
// unsafe - 禁用安全检查
unsafe
{
    int* ptr = &variable;
    *ptr = 42;
}

// fixed - 固定托管内存
unsafe
{
    fixed (byte* ptr = array)
    {
        // 使用ptr
    }
}
```

#### Rust 安全关键字
```rust
// unsafe - 禁用借用检查器（谨慎使用！）
unsafe {
    let ptr = &variable as *const i32;
    let value = *ptr; // 解引用原始指针
}

// 原始指针类型（无C#等价物 - 通常不需要）
let ptr: *const i32 = &42;  // 不可变原始指针
let ptr: *mut i32 = &mut 42; // 可变原始指针
```

### C#中没有的常见Rust关键字

```rust
// where - 泛型约束（比C# where更灵活）
fn generic_function<T>() 
where 
    T: Clone + Send + Sync,
{
    // T必须实现Clone、Send和Sync trait
}

// dyn - 动态trait对象（类似C# object但类型安全）
let drawable: Box<dyn Draw> = Box::new(Circle::new());

// Self - 引用实现类型（类似C# this但用于类型）
impl MyStruct {
    fn new() -> Self { // Self = MyStruct
        Self { field: 0 }
    }
}

// self - 方法接收器
impl MyStruct {
    fn method(&self) { }        // 不可变借用
    fn method_mut(&mut self) { } // 可变借用  
    fn consume(self) { }        // 获取所有权
}

// crate - 引用当前crate根目录
use crate::models::User; // 从crate根的绝对路径

// super - 引用父模块
use super::utils; // 从父模块导入
```

### C#开发者关键字汇总

| 用途 | C# | Rust | 关键差异 |
|---------|----|----|----------------|
| 可见性 | `public`, `private`, `internal` | `pub`，默认私有 | `pub(crate)`更细粒度 |
| 变量 | `var`, `readonly`, `const` | `let`, `let mut`, `const` | 默认不可变 |
| 函数 | `method()` | `fn` | 独立函数 |
| 类型 | `class`, `struct`, `interface` | `struct`, `enum`, `trait` | enum是代数类型 |
| 泛型 | `<T> where T : IFoo` | `<T> where T: Foo` | 更灵活的约束 |
| 引用 | `ref`, `out`, `in` | `&`, `&mut` | 编译时借用检查 |
| 模式 | `switch`, `is` | `match`, `if let` | 需要穷举匹配 |

***


