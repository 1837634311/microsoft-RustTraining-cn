# C/C++程序员的Rust引导课程

## 课程概述
- 课程概述
    - Rust的优势（从C和C++角度）
    - 本地安装
    - 类型、函数、控制流、模式匹配
    - 模块、cargo
    - Trait、泛型
    - 集合、错误处理
    - 闭包、内存管理、生命周期、智能指针
    - 并发
    - Unsafe Rust，包括外部函数接口（FFI）
    - `no_std` 和面向固件团队的嵌入式Rust要点
    - 案例研究：实际的C++到Rust翻译模式
- 本课程不涵盖 `async` Rust——详见配套的 [Async Rust Training](../async-book/)，其中包含对futures、executors、`Pin`、tokio和生产级async模式的完整讲解


---

# 自学指南

本教程既适合教师授课，也适合自学。如果你独立学习，以下是最大化学习效果的方法。

**学习进度建议：**

| 章节 | 主题 | 建议时间 | 里程碑 |
|----------|-------|---------------|------------|
| 1–4 | 环境搭建、类型、控制流 | 1天 | 你能用Rust编写CLI温度转换器 |
| 5–7 | 数据结构、所有权 | 1–2天 | 你能解释 *为什么* `let s2 = s1` 会使 `s1` 失效 |
| 8–9 | 模块、错误处理 | 1天 | 你能创建一个使用 `?` 传播错误的多文件项目 |
| 10–12 | Trait、泛型、闭包 | 1–2天 | 你能编写带有trait约束的泛型函数 |
| 13–14 | 并发、unsafe/FFI | 1天 | 你能用 `Arc<Mutex<T>>` 编写线程安全计数器 |
| 15–16 | 深度探讨 | 按需 | 参考材料——相关时阅读 |
| 17–19 | 最佳实践与参考 | 按需 | 编写实际代码时参考 |

**如何使用练习：**
- 每章都有标记难度的动手练习：🟢 入门、🟡 中级、🔴 挑战
- **始终在展开解决方案之前尝试练习。** 与所有权检查器斗争是学习的一部分——编译器的错误消息就是你的老师
- 如果卡壳超过15分钟，展开解决方案，学习它，然后关闭它从头再试
- [Rust Playground](https://play.rust-lang.org/) 让你无需本地安装就能运行代码

**当你遇到困难时：**
- 仔细阅读编译器错误消息——Rust的错误信息非常有帮助
- 重读相关章节；像所有权（第7章）这样的概念往往在第二遍阅读时才会理解
- [Rust标准库文档](https://doc.rust-lang.org/std/) 非常优秀——可以搜索任何类型或方法
- 关于async模式，参见配套的 [Async Rust Training](../async-book/)

---

# 目录

## 第一部分 — 基础

### 1. 引言与动机
- [讲师介绍与通用方法](ch01-introduction-and-motivation.md#speaker-intro-and-general-approach)
- [Rust的优势](ch01-introduction-and-motivation.md#the-case-for-rust)
- [Rust如何解决这些问题？](ch01-introduction-and-motivation.md#how-does-rust-address-these-issues)
- [其他Rust独特卖点与特性](ch01-introduction-and-motivation.md#other-rust-usps-and-features)
- [快速参考：Rust vs C/C++](ch01-introduction-and-motivation.md#quick-reference-rust-vs-cc)
- [为什么C开发者需要Rust](ch01-1-why-c-developers-need-rust.md)
  - [常见C漏洞](ch01-1-why-c-developers-need-rust.md#a-brief-peek-at-some-common-c-vulnerabilities)
  - [C漏洞图解](ch01-1-why-c-developers-need-rust.md#illustration-of-c-vulnerabilities)
- [为什么C++开发者需要Rust](ch01-2-why-cpp-developers-need-rust.md)
  - [Rust解决的C++挑战](ch01-2-why-cpp-developers-need-rust.md#c-challenges-that-rust-addresses)
  - [C++内存安全问题（即使使用现代C++）](ch01-2-why-cpp-developers-need-rust.md#c-memory-safety-issues-even-with-modern-c)

### 2. 入门
- [废话少说：给我看代码](ch02-getting-started.md#enough-talk-already-show-me-some-code)
- [Rust本地安装](ch02-getting-started.md#rust-local-installation)
- [Rust包（crate）](ch02-getting-started.md#rust-packages-crates)
- [示例：cargo和crate](ch02-getting-started.md#example-cargo-and-crates)

### 3. 基本类型与变量
- [内置Rust类型](ch03-built-in-types.md#built-in-rust-types)
- [Rust类型说明与赋值](ch03-built-in-types.md#rust-type-specification-and-assignment)
- [Rust类型说明与推断](ch03-built-in-types.md#rust-type-specification-and-inference)
- [Rust变量与可变性](ch03-built-in-types.md#rust-variables-and-mutability)

### 4. 控制流
- [Rust if关键字](ch04-control-flow.md#rust-if-keyword)
- [Rust使用while和for的循环](ch04-control-flow.md#rust-loops-using-while-and-for)
- [Rust使用loop的循环](ch04-control-flow.md#rust-loops-using-loop)
- [Rust表达式块](ch04-control-flow.md#rust-expression-blocks)

### 5. 数据结构与集合
- [Rust数组类型](ch05-data-structures.md#rust-array-type)
- [Rust元组](ch05-data-structures.md#rust-tuples)
- [Rust引用](ch05-data-structures.md#rust-references)
- [C++引用 vs Rust引用 — 关键区别](ch05-data-structures.md#c-references-vs-rust-references--key-differences)
- [Rust切片](ch05-data-structures.md#rust-slices)
- [Rust常量与静态量](ch05-data-structures.md#rust-constants-and-statics)
- [Rust字符串：String vs &str](ch05-data-structures.md#rust-strings-string-vs-str)
- [Rust结构体](ch05-data-structures.md#rust-structs)
- [Rust Vec\<T\>](ch05-data-structures.md#rust-vec-type)
- [Rust HashMap](ch05-data-structures.md#rust-hashmap-type)
- [练习：Vec和HashMap](ch05-data-structures.md#exercise-vec-and-hashmap)

### 6. 模式匹配与枚举
- [Rust枚举类型](ch06-enums-and-pattern-matching.md#rust-enum-types)
- [Rust match语句](ch06-enums-and-pattern-matching.md#rust-match-statement)
- [练习：使用match和枚举实现加法和减法](ch06-enums-and-pattern-matching.md#exercise-implement-add-and-subtract-using-match-and-enum)

### 7. 所有权与内存管理
- [Rust内存管理](ch07-ownership-and-borrowing.md#rust-memory-management)
- [Rust所有权、借用与生命周期](ch07-ownership-and-borrowing.md#rust-ownership-borrowing-and-lifetimes)
- [Rust移动语义](ch07-ownership-and-borrowing.md#rust-move-semantics)
- [Rust Clone](ch07-ownership-and-borrowing.md#rust-clone)
- [Rust Copy trait](ch07-ownership-and-borrowing.md#rust-copy-trait)
- [Rust Drop trait](ch07-ownership-and-borrowing.md#rust-drop-trait)
- [练习：移动、复制与Drop](ch07-ownership-and-borrowing.md#exercise-move-copy-and-drop)
- [Rust生命周期与借用](ch07-1-lifetimes-and-borrowing-deep-dive.md#rust-lifetime-and-borrowing)
- [Rust生命周期标注](ch07-1-lifetimes-and-borrowing-deep-dive.md#rust-lifetime-annotations)
- [练习：带生命周期的切片存储](ch07-1-lifetimes-and-borrowing-deep-dive.md#exercise-slice-storage-with-lifetimes)
- [生命周期省略规则深度探讨](ch07-1-lifetimes-and-borrowing-deep-dive.md#lifetime-elision-rules-deep-dive)
- [Rust Box\<T\>](ch07-2-smart-pointers-and-interior-mutability.md#rust-boxt)
- [内部可变性：Cell\<T\>和RefCell\<T\>](ch07-2-smart-pointers-and-interior-mutability.md#interior-mutability-cellt-and-refcellt)
- [共享所有权：Rc\<T\>](ch07-2-smart-pointers-and-interior-mutability.md#shared-ownership-rct)
- [练习：共享所有权与内部可变性](ch07-2-smart-pointers-and-interior-mutability.md#exercise-shared-ownership-and-interior-mutability)

### 8. 模块与crate
- [Rust crate与模块](ch08-crates-and-modules.md#rust-crates-and-modules)
- [练习：模块与函数](ch08-crates-and-modules.md#exercise-modules-and-functions)
- [工作空间与crate（包）](ch08-crates-and-modules.md#workspaces-and-crates-packages)
- [练习：使用工作空间和包依赖](ch08-crates-and-modules.md#exercise-using-workspaces-and-package-dependencies)
- [使用crates.io上的社区crate](ch08-crates-and-modules.md#using-community-crates-from-cratesio)
- [crate依赖与SemVer](ch08-crates-and-modules.md#crates-dependencies-and-semver)
- [练习：使用rand crate](ch08-crates-and-modules.md#exercise-using-the-rand-crate)
- [Cargo.toml与Cargo.lock](ch08-crates-and-modules.md#cargotoml-and-cargolock)
- [Cargo test功能](ch08-crates-and-modules.md#cargo-test-feature)
- [其他Cargo功能](ch08-crates-and-modules.md#other-cargo-features)
- [测试模式](ch08-1-testing-patterns.md)

### 9. 错误处理
- [将枚举与Option和Result连接](ch09-error-handling.md#connecting-enums-to-option-and-result)
- [Rust Option类型](ch09-error-handling.md#rust-option-type)
- [Rust Result类型](ch09-error-handling.md#rust-result-type)
- [练习：使用Option实现log()函数](ch09-error-handling.md#exercise-log-function-implementation-with-option)
- [Rust错误处理](ch09-error-handling.md#rust-error-handling)
- [练习：错误处理](ch09-error-handling.md#exercise-error-handling)
- [错误处理最佳实践](ch09-1-error-handling-best-practices.md)

### 10. Trait与泛型
- [Rust trait](ch10-traits.md#rust-traits)
- [C++运算符重载 → Rust std::ops Trait](ch10-traits.md#c-operator-overloading--rust-stdops-traits)
- [练习：实现Logger trait](ch10-traits.md#exercise-logger-trait-implementation)
- [何时使用枚举 vs dyn Trait](ch10-traits.md#when-to-use-enum-vs-dyn-trait)
- [练习：翻译前先思考](ch10-traits.md#exercise-think-before-you-translate)
- [Rust泛型](ch10-1-generics.md#rust-generics)
- [练习：泛型](ch10-1-generics.md#exercise-generics)
- [组合Rust trait与泛型](ch10-1-generics.md#combining-rust-traits-and-generics)
- [Rust trait在数据类型中的约束](ch10-1-generics.md#rust-traits-constraints-in-data-types)
- [练习：trait约束与泛型](ch10-1-generics.md#exercise-traits-constraints-and-generics)
- [Rust类型状态模式与泛型](ch10-1-generics.md#rust-type-state-pattern-and-generics)
- [Rust builder模式](ch10-1-generics.md#rust-builder-pattern)

### 11. 类型系统高级特性
- [Rust From和Into trait](ch11-from-and-into-traits.md#rust-from-and-into-traits)
- [练习：From和Into](ch11-from-and-into-traits.md#exercise-from-and-into)
- [Rust Default trait](ch11-from-and-into-traits.md#rust-default-trait)
- [其他Rust类型转换](ch11-from-and-into-traits.md#other-rust-type-conversions)

### 12. 函数式编程
- [Rust闭包](ch12-closures.md#rust-closures)
- [练习：闭包与捕获](ch12-closures.md#exercise-closures-and-capturing)
- [Rust迭代器](ch12-closures.md#rust-iterators)
- [练习：Rust迭代器](ch12-closures.md#exercise-rust-iterators)
- [迭代器强力工具参考](ch12-1-iterator-power-tools.md#iterator-power-tools-reference)

### 13. 并发
- [Rust并发](ch13-concurrency.md#rust-concurrency)
- [为什么Rust防止数据竞争：Send和Sync](ch13-concurrency.md#why-rust-prevents-data-races-send-and-sync)
- [练习：多线程词频统计](ch13-concurrency.md#exercise-multi-threaded-word-count)

### 14. Unsafe Rust与FFI
- [Unsafe Rust](ch14-unsafe-rust-and-ffi.md#unsafe-rust)
- [简单FFI示例](ch14-unsafe-rust-and-ffi.md#simple-ffi-example-rust-library-function-consumed-by-c)
- [复杂FFI示例](ch14-unsafe-rust-and-ffi.md#complex-ffi-example)
- [确保unsafe代码的正确性](ch14-unsafe-rust-and-ffi.md#ensuring-correctness-of-unsafe-code)
- [练习：编写安全的FFI包装器](ch14-unsafe-rust-and-ffi.md#exercise-writing-a-safe-ffi-wrapper)

## 第二部分 — 深度探讨

### 15. no_std — 裸机Rust
- [什么是no_std？](ch15-no_std-rust-without-the-standard-library.md#what-is-no_std)
- [何时使用no_std vs std](ch15-no_std-rust-without-the-standard-library.md#when-to-use-no_std-vs-std)
- [练习：no_std环形缓冲区](ch15-no_std-rust-without-the-standard-library.md#exercise-no_std-ring-buffer)
- [嵌入式深度探讨](ch15-1-embedded-deep-dive.md)

### 16. 案例研究：实际C++到Rust的翻译
- [案例研究1：继承层次结构 → 枚举分发](ch16-case-studies.md#case-study-1-inheritance-hierarchy--enum-dispatch)
- [案例研究2：shared_ptr树 → 内存池/索引模式](ch16-case-studies.md#case-study-2-shared_ptr-tree--arenaindex-pattern)
- [案例研究3：框架通信 → 生命周期借用](ch16-1-case-study-lifetime-borrowing.md#case-study-3-framework-communication--lifetime-borrowing)
- [案例研究4：上帝对象 → 可组合状态](ch16-1-case-study-lifetime-borrowing.md#case-study-4-god-object--composable-state)
- [案例研究5：Trait对象 — 何时它们确实是正确的选择](ch16-1-case-study-lifetime-borrowing.md#case-study-5-trait-objects--when-they-are-right)

## 第三部分 — 最佳实践与参考

### 17. 最佳实践
- [Rust最佳实践总结](ch17-best-practices.md#rust-best-practices-summary)
- [避免过多的clone()](ch17-1-avoiding-excessive-clone.md#avoiding-excessive-clone)
- [避免未检查的索引](ch17-2-avoiding-unchecked-indexing.md#avoiding-unchecked-indexing)
- [扁平化赋值金字塔](ch17-3-collapsing-assignment-pyramids.md#collapsing-assignment-pyramids)
- [终极练习：诊断事件管道](ch17-3-collapsing-assignment-pyramids.md#capstone-exercise-diagnostic-event-pipeline)
- [日志与追踪生态系统](ch17-4-logging-and-tracing-ecosystem.md#logging-and-tracing-ecosystem)

### 18. C++ → Rust语义深度探讨
- [类型转换、预处理器、模块、volatile、static、constexpr、SFINAE等](ch18-cpp-rust-semantic-deep-dives.md)

### 19. Rust宏
- [声明式宏（`macro_rules!`）](ch19-macros.md#declarative-macros-with-macro_rules)
- [常见标准库宏](ch19-macros.md#common-standard-library-macros)
- [派生宏](ch19-macros.md#derive-macros)
- [属性宏](ch19-macros.md#attribute-macros)
- [过程宏](ch19-macros.md#procedural-macros-conceptual-overview)
- [何时使用什么：宏 vs 函数 vs 泛型](ch19-macros.md#when-to-use-what-macros-vs-functions-vs-generics)
- [练习](ch19-macros.md#exercises)
