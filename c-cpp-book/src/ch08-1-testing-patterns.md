## C++ 程序员的测试模式

> **你将学到什么：** Rust 内置的测试框架——`#[test]`、`#[should_panic]`、返回 `Result` 的测试、用于测试数据的 builder 模式、基于 trait 的模拟、使用 `proptest` 的属性测试、使用 `insta` 的快照测试，以及集成测试组织。替代 Google Test + CMake 的零配置测试。

C++ 测试通常依赖外部框架（Google Test、Catch2、Boost.Test）以及复杂的构建集成。Rust 的测试框架**内置于语言和工具链中**——无需依赖、无需 CMake 集成、无需测试运行器配置。

### `#[test]` 之外的测试属性

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn basic_pass() {
        assert_eq!(2 + 2, 4);
    }

    // Expect a panic — equivalent to GTest's EXPECT_DEATH
    #[test]
    #[should_panic]
    fn out_of_bounds_panics() {
        let v = vec![1, 2, 3];
        let _ = v[10]; // Panics — test passes
    }

    // Expect a panic with a specific message substring
    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn specific_panic_message() {
        let v = vec![1, 2, 3];
        let _ = v[10];
    }

    // Tests that return Result<(), E> — use ? instead of unwrap()
    #[test]
    fn test_with_result() -> Result<(), String> {
        let value: u32 = "42".parse().map_err(|e| format!("{e}"))?;
        assert_eq!(value, 42);
        Ok(())
    }

    // Ignore slow tests by default — run with `cargo test -- --ignored`
    #[test]
    #[ignore]
    fn slow_integration_test() {
        std::thread::sleep(std::time::Duration::from_secs(10));
    }
}
```

```bash
cargo test                          # Run all non-ignored tests
cargo test -- --ignored             # Run only ignored tests
cargo test -- --include-ignored     # Run ALL tests including ignored
cargo test test_name                # Run tests matching a name pattern
cargo test -- --nocapture           # Show println! output during tests
cargo test -- --test-threads=1      # Run tests serially (for shared state)
```

### 测试辅助函数：用于测试数据的 builder 模式

在 C++ 中你会使用 Google Test fixtures（`class MyTest : public ::testing::Test`）。
在 Rust 中，使用 builder 函数或 `Default` trait：

```rust
#[cfg(test)]
mod tests {
    use super::*;

    // Builder function — creates test data with sensible defaults
    fn make_gpu_event(severity: Severity, fault_code: u32) -> DiagEvent {
        DiagEvent {
            source: "accel_diag".to_string(),
            severity,
            message: format!("Test event FC:{fault_code}"),
            fault_code,
        }
    }

    // Reusable test fixture — a set of pre-built events
    fn sample_events() -> Vec<DiagEvent> {
        vec![
            make_gpu_event(Severity::Critical, 67956),
            make_gpu_event(Severity::Warning, 32709),
            make_gpu_event(Severity::Info, 10001),
        ]
    }

    #[test]
    fn filter_critical_events() {
        let events = sample_events();
        let critical: Vec<_> = events.iter()
            .filter(|e| e.severity == Severity::Critical)
            .collect();
        assert_eq!(critical.len(), 1);
        assert_eq!(critical[0].fault_code, 67956);
    }
}
```

### 使用 traits 进行模拟

在 C++ 中，模拟需要 Google Mock 等框架或手动虚拟覆盖。
在 Rust 中，为依赖定义一个 trait，并在测试中交换实现：

```rust
// Production trait
trait SensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String>;
}

// Production implementation
struct HwSensorReader;
impl SensorReader for HwSensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String> {
        // Real hardware call...
        Ok(72.5)
    }
}

// Test mock — returns predictable values
#[cfg(test)]
struct MockSensorReader {
    temperatures: std::collections::HashMap<u32, f64>,
}

#[cfg(test)]
impl SensorReader for MockSensorReader {
    fn read_temperature(&self, sensor_id: u32) -> Result<f64, String> {
        self.temperatures.get(&sensor_id)
            .copied()
            .ok_or_else(|| format!("Unknown sensor {sensor_id}"))
    }
}

// Function under test — generic over the reader
fn check_overtemp(reader: &impl SensorReader, ids: &[u32], threshold: f64) -> Vec<u32> {
    ids.iter()
        .filter(|&&id| reader.read_temperature(id).unwrap_or(0.0) > threshold)
        .copied()
        .collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn detect_overtemp_sensors() {
        let mut mock = MockSensorReader { temperatures: Default::default() };
        mock.temperatures.insert(0, 72.5);
        mock.temperatures.insert(1, 91.0);  // Over threshold
        mock.temperatures.insert(2, 65.0);

        let hot = check_overtemp(&mock, &[0, 1, 2], 80.0);
        assert_eq!(hot, vec![1]);
    }
}
```

### 测试中的临时文件和目录

C++ 测试经常使用平台特定的临时目录。Rust 有 `tempfile`：

```rust
// Cargo.toml: [dev-dependencies]
// tempfile = "3"

#[cfg(test)]
mod tests {
    use super::*;
    use tempfile::NamedTempFile;
    use std::io::Write;

    #[test]
    fn parse_config_from_file() -> Result<(), Box<dyn std::error::Error>> {
        // Create a temp file that's auto-deleted when dropped
        let mut file = NamedTempFile::new()?;
        writeln!(file, r#"{{"sku": "ServerNode", "level": "Quick"}}"#)?;

        let config = load_config(file.path().to_str().unwrap())?;
        assert_eq!(config.sku, "ServerNode");
        Ok(())
        // file is deleted here — no cleanup code needed
    }
}
```

### 使用 `proptest` 的基于属性的测试

不要编写特定的测试用例，而是描述应该对所有输入成立的**属性**。`proptest` 生成随机输入并找到最小失败案例：

```rust
// Cargo.toml: [dev-dependencies]
// proptest = "1"

#[cfg(test)]
mod tests {
    use proptest::prelude::*;

    fn parse_and_format(n: u32) -> String {
        format!("{n}")
    }

    proptest! {
        #[test]
        fn roundtrip_u32(n: u32) {
            let formatted = parse_and_format(n);
            let parsed: u32 = formatted.parse().unwrap();
            prop_assert_eq!(n, parsed);
        }

        #[test]
        fn string_contains_no_null(s in "[a-zA-Z0-9 ]{0,100}") {
            prop_assert!(!s.contains('\0'));
        }
    }
}
```

### 使用 `insta` 进行快照测试

对于产生复杂输出（JSON、格式化字符串）的测试，`insta` 自动生成并管理参考快照：

```rust
// Cargo.toml: [dev-dependencies]
// insta = { version = "1", features = ["json"] }

#[cfg(test)]
mod tests {
    use insta::assert_json_snapshot;

    #[test]
    fn der_entry_format() {
        let entry = DerEntry {
            fault_code: 67956,
            component: "GPU".to_string(),
            message: "ECC error detected".to_string(),
        };
        // First run: creates a snapshot file in tests/snapshots/
        // Subsequent runs: compares against the saved snapshot
        assert_json_snapshot!(entry);
    }
}
```

```bash
cargo insta test              # Run tests and review new/changed snapshots
cargo insta review            # Interactive review of snapshot changes
```

### C++ vs Rust 测试比较

| **C++（Google Test）** | **Rust** | **备注** |
|----------------------|---------|----------|
| `TEST(Suite, Name) { }` | `#[test] fn name() { }` | 不需要 suite/类层次结构 |
| `ASSERT_EQ(a, b)` | `assert_eq!(a, b)` | 内置宏，无需框架 |
| `ASSERT_NEAR(a, b, eps)` | `assert!((a - b).abs() < eps)` | 或使用 `approx` crate |
| `EXPECT_THROW(expr, type)` | `#[should_panic(expected = "...")]` | 或使用 `catch_unwind` 进行精细控制 |
| `EXPECT_DEATH(expr, "msg")` | `#[should_panic(expected = "msg")]` | |
| `class Fixture : public ::testing::Test` | Builder 函数 + `Default` | 不需要继承 |
| Google Mock `MOCK_METHOD` | Trait + 测试实现 | 更显式，无需宏魔法 |
| `INSTANTIATE_TEST_SUITE_P`（参数化） | `proptest!` 或宏生成的测试 | |
| `SetUp()` / `TearDown()` | 通过 `Drop` 的 RAII——自动清理 | 变量在测试结束时被 drop |
| 单独的测试二进制文件 + CMake | `cargo test`——零配置 | |
| `ctest --output-on-failure` | `cargo test -- --nocapture` | |

----

### 集成测试：`tests/` 目录

单元测试位于代码旁边的 `#[cfg(test)]` 模块中。**集成测试**位于 crate 根目录的单独 `tests/` 目录中，像外部消费者一样测试你库的公共 API：

```
my_crate/
├── src/
│   └── lib.rs          # Your library code
├── tests/
│   ├── smoke.rs        # Each .rs file is a separate test binary
│   ├── regression.rs
│   └── common/
│       └── mod.rs      # Shared test helpers (NOT a test itself)
└── Cargo.toml
```

```rust
// tests/smoke.rs — tests your crate as an external user would
use my_crate::DiagEngine;  // Only public API is accessible

#[test]
fn engine_starts_successfully() {
    let engine = DiagEngine::new("test_config.json");
    assert!(engine.is_ok());
}

#[test]
fn engine_rejects_invalid_config() {
    let engine = DiagEngine::new("nonexistent.json");
    assert!(engine.is_err());
}
```

```rust
// tests/common/mod.rs — shared helpers, NOT compiled as a test binary
pub fn setup_test_environment() -> tempfile::TempDir {
    let dir = tempfile::tempdir().unwrap();
    std::fs::write(dir.path().join("config.json"), r#"{"log_level": "debug"}"#).unwrap();
    dir
}
```

```rust
// tests/regression.rs — can use shared helpers
mod common;

#[test]
fn regression_issue_42() {
    let env = common::setup_test_environment();
    let engine = my_crate::DiagEngine::new(
        env.path().join("config.json").to_str().unwrap()
    );
    assert!(engine.is_ok());
}
```

**运行集成测试：**
```bash
cargo test                          # 运行单元和集成测试
cargo test --test smoke             # 仅运行 tests/smoke.rs
cargo test --test regression        # 仅运行 tests/regression.rs
cargo test --lib                    # 仅运行单元测试（跳过集成）
```

> **与单元测试的关键差异**：集成测试无法访问私有函数或 `pub(crate)` 项。这迫使你验证你的公共 API 是否足够——一个有价值的设计信号。在 C++ 术语中，这相当于仅针对公共头文件进行测试，没有 `friend` 访问权限。

----


