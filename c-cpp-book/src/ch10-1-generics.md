# Rust 泛型

> **你将学到什么：** 泛型类型参数、单态化（零成本泛型）、trait bounds，以及 Rust 泛型如何与 C++ 模板比较——具有更好的错误消息和没有 SFINAE。

- 泛型允许相同的算法或数据结构跨数据类型重用
    - 泛型参数作为标识符出现在 `<>` 中，例如：`<T>`。参数可以是任何合法标识符名称，但通常保持简短以简洁</    - 编译器在编译时执行单态化，即为遇到的每个 `T` 变体生成一个新类型
```rust
// Returns a tuple of type <T> composed of left and right of type <T>
fn pick<T>(x: u32, left: T, right: T) -> (T, T) {
   if x == 42 {
    (left, right) 
   } else {
    (right, left)
   }
}
fn main() {
    let a = pick(42, true, false);
    let b = pick(42, "hello", "world");
    println!("{a:?}, {b:?}");
}
```

# Rust 泛型
- 泛型也可以应用于数据类型和关联方法。可以为特定的 `<T>` 专门化实现（例如：`f32` vs. `u32`）
```rust
#[derive(Debug)] // We will discuss this later
struct Point<T> {
    x : T,
    y : T,
}
impl<T> Point<T> {
    fn new(x: T, y: T) -> Self {
        Point {x, y}
    }
    fn set_x(&mut self, x: T) {
         self.x = x;       
    }
    fn set_y(&mut self, y: T) {
         self.y = y;       
    }
}
impl Point<f32> {
    fn is_secret(&self) -> bool {
        self.x == 42.0
    }    
}
fn main() {
    let mut p = Point::new(2, 4); // i32
    let q = Point::new(2.0, 4.0); // f32
    p.set_x(42);
    p.set_y(43);
    println!("{p:?} {q:?} {}", q.is_secret());
}
```

# 练习：泛型

🟢 **入门级**
- 修改 `Point` 类型，为 x 和 y 使用两种不同的类型（`T` 和 `U`）

<details><summary>Solution (click to expand)</summary>

```rust
#[derive(Debug)]
struct Point<T, U> {
    x: T,
    y: U,
}

impl<T, U> Point<T, U> {
    fn new(x: T, y: U) -> Self {
        Point { x, y }
    }
}

fn main() {
    let p1 = Point::new(42, 3.14);        // Point<i32, f64>
    let p2 = Point::new("hello", true);   // Point<&str, bool>
    let p3 = Point::new(1u8, 1000u64);    // Point<u8, u64>
    println!("{p1:?}");
    println!("{p2:?}");
    println!("{p3:?}");
}
// Output:
// Point { x: 42, y: 3.14 }
// Point { x: "hello", y: true }
// Point { x: 1, y: 1000 }
```

</details>

### 结合 Rust traits 和泛型
- Traits 可用于对泛型类型施加限制（约束）
- 约束可以使用 `: ` 在泛型类型参数后指定，或使用 `where`。以下定义了一个泛型函数 `get_area`，它接受任何实现了 `ComputeArea` trait 的类型 `T`
```rust
    trait ComputeArea {
        fn area(&self) -> u64;
    }
    fn get_area<T: ComputeArea>(t: &T) -> u64 {
        t.area()
    }
```
- [▶ Try it in the Rust Playground](https://play.rust-lang.org/)

### Combining Rust traits and generics
- It is possible to have multiple trait constraints
```rust
trait Fish {}
trait Mammal {}
struct Shark;
struct Whale;
impl Fish for Shark {}
impl Fish for Whale {}
impl Mammal for Whale {}
fn only_fish_and_mammals<T: Fish + Mammal>(_t: &T) {}
fn main() {
    let w = Whale {};
    only_fish_and_mammals(&w);
    let _s = Shark {};
    // Won't compile
    only_fish_and_mammals(&_s);
}
```

### Rust traits constraints in data types
- Traits constrainsts can be combined with generics in data types
- In the following example, we define the ```PrintDescription``` ```trait``` and a generic ```struct``` ```Shape``` with a member constrained by the trait
```rust
trait PrintDescription {
    fn print_description(&self);
}
struct Shape<S: PrintDescription> {
    shape: S,
}
// Generic Shape implementation for any type that implements PrintDescription
impl<S: PrintDescription> Shape<S> {
    fn print(&self) {
        self.shape.print_description();
    }
}
```
- [▶ Try it in the Rust Playground](https://play.rust-lang.org/)

# Exercise: Traits constraints and generics

🟡 **Intermediate**
- Implement a ```struct``` with a generic member ```cipher``` that implements ```CipherText```
```rust
trait CipherText {
    fn encrypt(&self);
}
// TO DO
//struct Cipher<>

```
- Next, implement a method called ```encrypt``` on the ```struct``` ```impl``` that invokes ```encrypt``` on ```cipher```
```rust
// TO DO
impl for Cipher<> {}
```
- Next, implement ```CipherText``` on two structs called ```CipherOne``` and ```CipherTwo``` (just ```println()``` is fine). Create ```CipherOne``` and ```CipherTwo```, and use ```Cipher``` to invoke them

<details><summary>Solution (click to expand)</summary>

```rust
trait CipherText {
    fn encrypt(&self);
}

struct Cipher<T: CipherText> {
    cipher: T,
}

impl<T: CipherText> Cipher<T> {
    fn encrypt(&self) {
        self.cipher.encrypt();
    }
}

struct CipherOne;
struct CipherTwo;

impl CipherText for CipherOne {
    fn encrypt(&self) {
        println!("CipherOne encryption applied");
    }
}

impl CipherText for CipherTwo {
    fn encrypt(&self) {
        println!("CipherTwo encryption applied");
    }
}

fn main() {
    let c1 = Cipher { cipher: CipherOne };
    let c2 = Cipher { cipher: CipherTwo };
    c1.encrypt();
    c2.encrypt();
}
// Output:
// CipherOne encryption applied
// CipherTwo encryption applied
```

</details>

### Rust type state pattern and generics
- Rust types can be used to enforce state machine transitions at *compile* time
    - Consider a ```Drone``` with say two states: ```Idle``` and ```Flying```. In the ```Idle``` state, the only permitted method is ```takeoff()```. In the ```Flying``` state, we permit ```land()```
    
- One approach is to model the state machine using something like the following
```rust
enum DroneState {
    Idle,
    Flying
}
struct Drone {x: u64, y: u64, z: u64, state: DroneState}  // x, y, z are coordinates
```
- This requires a lot of runtime checks to enforce the state machine semantics — [▶ try it](https://play.rust-lang.org/) to see why

### Rust type state pattern generics
- Generics allows us to enforce the state machine at *compile time*. This requires using a special generic called ```PhantomData<T>```
- The ```PhantomData<T>``` is a ```zero-sized``` marker data type. In this case, we use it to represent the ```Idle``` and ```Flying``` states, but it has ```zero``` runtime size
- Notice that the ```takeoff``` and ```land``` methods take ```self``` as a parameter. This is referred to as ```consuming``` (contrast with ```&self``` which uses borrowing). Basically, once we call the ```takeoff()``` on ```Drone<Idle>```, we can only get back a ```Drone<Flying>``` and viceversa
```rust
struct Drone<T> {x: u64, y: u64, z: u64, state: PhantomData<T> }
impl Drone<Idle> {
    fn takeoff(self) -> Drone<Flying> {...}
}
impl Drone<Flying> {
    fn land(self) -> Drone<Idle> { ...}
}
```
    - [▶ Try it in the Rust Playground](https://play.rust-lang.org/)

### Rust type state pattern generics
- Key takeaways:
    - States can be represented using structs (zero-size)
    - We can combine the state ```T``` with ```PhantomData<T>``` (zero-size)
    - Implementing the methods for a particular stage of the state machine is now just a matter of ```impl State<T>```
    - Use a method that consumes ```self``` to transition from one state to another
    - This gives us ```zero cost``` abstractions. The compiler can enforce the state machine at compile time and it's impossible to call methods unless the state is right

### Rust builder pattern
- The consume ```self``` can be useful for builder patterns
- Consider a GPIO configuration with several dozen pins. The pins can be configured to high or low (default is low)
```rust
#[derive(default)]
enum PinState {
    #[default]
    Low,
    High,
} 
#[derive(default)]
struct GPIOConfig {
    pin0: PinState,
    pin1: PinState
    ... 
}
```
- The builder pattern can be used to construct a GPIO configuration by chaining — [▶ Try it](https://play.rust-lang.org/)


