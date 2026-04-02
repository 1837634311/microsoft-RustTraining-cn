## 安装和设置

> **学习内容：** 如何安装Rust并设置IDE、Cargo构建系统vs MSBuild/NuGet、
> 你的第一个Rust程序与C#的对比，以及如何读取命令行输入。
>
> **难度：** 🟢 初级

### 安装Rust
```bash
# 安装Rust（适用于Windows、macOS、Linux）
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 在Windows上，你也可以从以下地址下载：https://rustup.rs/
```

### Rust工具 vs C#工具
| C# 工具 | Rust 等价物 | 用途 |
|---------|----------------|---------|
| `dotnet new` | `cargo new` | 创建新项目 |
| `dotnet build` | `cargo build` | 编译项目 |
| `dotnet run` | `cargo run` | 运行项目 |
| `dotnet test` | `cargo test` | 运行测试 |
| NuGet | Crates.io | 包仓库 |
| MSBuild | Cargo | 构建系统 |
| Visual Studio | VS Code + rust-analyzer | IDE |

### IDE 设置
1. **VS Code**（推荐给初学者）
   - 安装 "rust-analyzer" 扩展
   - 安装 "CodeLLDB" 用于调试

2. **Visual Studio**（Windows）
   - 安装Rust支持扩展

3. **JetBrains RustRover**（完整IDE）
   - 类似C#的Rider

***

## 你的第一个Rust程序

### C# Hello World
```csharp
// Program.cs
using System;

namespace HelloWorld
{
    class Program
    {
        static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

### Rust Hello World
```rust
// main.rs
fn main() {
    println!("Hello, World!");
}
```

### C#开发者的关键差异
1. **不需要类** - 函数可以在顶层存在
2. **没有命名空间** - 使用模块系统代替
3. **`println!` 是一个宏** - 注意那个 `!`
4. **println!后面没有分号** - 表达式vs语句
5. **没有显式返回类型** - `main` 返回 `()`（单元类型）

### 创建你的第一个项目
```bash
# 创建新项目（类似 'dotnet new console'）
cargo new hello_rust
cd hello_rust

# 创建的项目结构：
# hello_rust/
# ├── Cargo.toml      （类似 .csproj 文件）
# └── src/
#     └── main.rs     （类似 Program.cs）

# 运行项目（类似 'dotnet run'）
cargo run
```

***

## Cargo vs NuGet/MSBuild

### 项目配置

**C# (.csproj)**
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  <PackageReference Include="Serilog" Version="3.0.1" />
</Project>
```

**Rust (Cargo.toml)**
```toml
[package]
name = "hello_rust"
version = "0.1.0"
edition = "2021"

[dependencies]
serde_json = "1.0"    # 类似 Newtonsoft.Json
log = "0.4"           # 类似 Serilog
```

### 常用Cargo命令
```bash
# 创建新项目
cargo new my_project
cargo new my_project --lib  # 创建库项目

# 构建和运行
cargo build          # 类似 'dotnet build'
cargo run            # 类似 'dotnet run'
cargo test           # 类似 'dotnet test'

# 包管理
cargo add serde      # 添加依赖（类似 'dotnet add package'）
cargo update         # 更新依赖

# Release构建
cargo build --release  # 优化构建
cargo run --release    # 运行优化版本

# 文档
cargo doc --open     # 生成并打开文档
```

### Workspace vs 解决方案

**C# 解决方案 (.sln)**
```text
MySolution/
├── MySolution.sln
├── WebApi/
│   └── WebApi.csproj
├── Business/
│   └── Business.csproj
└── Tests/
    └── Tests.csproj
```

**Rust Workspace (Cargo.toml)**
```toml
[workspace]
members = [
    "web_api",
    "business", 
    "tests"
]
```

***

## 读取输入和CLI参数

每个C#开发者都知道 `Console.ReadLine()`。以下是处理用户输入、环境变量和命令行参数的方法。

### 控制台输入
```csharp
// C# — 读取用户输入
Console.Write("Enter your name: ");
string name = Console.ReadLine();
Console.WriteLine($"Hello, {name}!");

// 解析输入
Console.Write("Enter a number: ");
if (int.TryParse(Console.ReadLine(), out int number))
{
    Console.WriteLine($"You entered: {number}");
}
else
{
    Console.WriteLine("That's not a valid number.");
}
```

```rust
use std::io::{self, Write};

fn main() {
    // 读取一行输入
    print!("Enter your name: ");
    io::stdout().flush().unwrap(); // flush因为print!不会自动flush

    let mut name = String::new();
    io::stdin().read_line(&mut name).expect("Failed to read line");
    let name = name.trim(); // 移除尾部换行符
    println!("Hello, {name}!");

    // 解析输入
    print!("Enter a number: ");
    io::stdout().flush().unwrap();

    let mut input = String::new();
    io::stdin().read_line(&mut input).expect("Failed to read");
    match input.trim().parse::<i32>() {
        Ok(number) => println!("You entered: {number}"),
        Err(_)     => println!("That's not a valid number."),
    }
}
```

### 命令行参数
```csharp
// C# — 读取CLI参数
static void Main(string[] args)
{
    if (args.Length < 1)
    {
        Console.WriteLine("Usage: program <filename>");
        return;
    }
    string filename = args[0];
    Console.WriteLine($"Processing {filename}");
}
```

```rust
use std::env;

fn main() {
    let args: Vec<String> = env::args().collect();
    //  args[0] = 程序名（类似C#的Assembly名称）
    //  args[1..] = 实际参数

    if args.len() < 2 {
        eprintln!("Usage: {} <filename>", args[0]); // eprintln! → stderr
        std::process::exit(1);
    }
    let filename = &args[1];
    println!("Processing {filename}");
}
```

### 环境变量
```csharp
// C#
string dbUrl = Environment.GetEnvironmentVariable("DATABASE_URL") ?? "localhost";
```

```rust
use std::env;

let db_url = env::var("DATABASE_URL").unwrap_or_else(|_| "localhost".to_string());
// env::var返回Result<String, VarError> — 没有null！
```

### 使用`clap`构建生产级CLI应用

对于任何超出简单参数解析的场景，使用 **`clap`** crate — 它是 `System.CommandLine` 或类似 `CommandLineParser` 库的Rust等价物。

```toml
# Cargo.toml
[dependencies]
clap = { version = "4", features = ["derive"] }
```

```rust
use clap::Parser;

/// 一个简单的文件处理器 — 此文档注释成为帮助文本
#[derive(Parser, Debug)]
#[command(name = "processor", version, about)]
struct Args {
    /// 要处理的输入文件
    #[arg(short, long)]
    input: String,

    /// 输出文件（默认为stdout）
    #[arg(short, long)]
    output: Option<String>,

    /// 启用详细日志
    #[arg(short, long, default_value_t = false)]
    verbose: bool,

    /// 工作线程数
    #[arg(short = 'j', long, default_value_t = 4)]
    threads: usize,
}

fn main() {
    let args = Args::parse(); // 自动解析、验证、生成--help

    if args.verbose {
        println!("Input:   {}", args.input);
        println!("Output:  {:?}", args.output);
        println!("Threads: {}", args.threads);
    }

    // 使用 args.input, args.output 等
}
```

```bash
# 自动生成的帮助：
$ processor --help
A simple file processor

Usage: processor [OPTIONS] --input <INPUT>

Options:
  -i, --input <INPUT>      Input file to process
  -o, --output <OUTPUT>    Output file (defaults to stdout)
  -v, --verbose            Enable verbose logging
  -j, --threads <THREADS>  Number of worker threads [default: 4]
  -h, --help               Print help
  -V, --version            Print version
```

```csharp
// C#等价的System.CommandLine（更多样板代码）：
var inputOption = new Option<string>("--input", "Input file") { IsRequired = true };
var verboseOption = new Option<bool>("--verbose", "Enable verbose logging");
var rootCommand = new RootCommand("A simple file processor");
rootCommand.AddOption(inputOption);
rootCommand.AddOption(verboseOption);
rootCommand.SetHandler((input, verbose) => { /* ... */ }, inputOption, verboseOption);
await rootCommand.InvokeAsync(args);
// clap的derive宏方法更简洁且类型安全
```

| C# | Rust | 备注 |
|----|------|-------|
| `Console.ReadLine()` | `io::stdin().read_line(&mut buf)` | 必须提供缓冲区，返回 `Result` |
| `int.TryParse(s, out n)` | `s.parse::<i32>()` | 返回 `Result<i32, ParseIntError>` |
| `args[0]` | `env::args().nth(1)` | Rust args[0] = 程序名 |
| `Environment.GetEnvironmentVariable` | `env::var("KEY")` | 返回 `Result`，不是nullable |
| `System.CommandLine` | `clap` | 基于derive，自动生成帮助 |

***

