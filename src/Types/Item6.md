# 条目 6: 拥抱新类型

[条目 1](./Item1.md)描述了元组结构体（tuple structs），元组结构体中没有命名字段而是通过数字来进行引用访问(self.0)。这个条目着重于有已经存在的类型单个类型的元组结构体，从而创建一个新类型，该类型可以保存于封装完全相同的值范围。这种模式在Rust中非常普遍，它非常值得我们在单一的条目中进行讨论，而且可以有一个自己的名字：newtype模式。

newtype模式最简单的用途是为类型指示超出其正常行为的[附加语意](https://doc.rust-lang.org/book/ch19-04-advanced-types.html#using-the-newtype-pattern-for-type-safety-and-abstraction).为了说明这一点，想象一个将一个卫星发送到火星的项目。这是一个非常大的项目，因此不同的团队构建了该项目的不同部分，一组处理火箭发动机的代码：

```rust
/// Fire the thrusters. Returns generated impulse in pound-force seconds.
pub fn thruster_impulse(direction: Direction) -> f64 {
    // ...
    return 42.0;
}
```

而另一组负责惯性导航系统：

```rust
/// Update trajectory model for impulse, provided in Newton seconds.
pub fn update_trajectory(force: f64) {
    // ...
}
```

最终这些不同的部分需要连接在一起：

```rust
let thruster_force: f64 = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force);
```

糟糕 [Mars Climate Orbiter](https://en.wikipedia.org/wiki/Mars_Climate_Orbiter#Cause_of_failure)

Rust还包含类型别名特性，它允许不同的组可以清晰地表达自己的意图：

```rust
/// Units for force.
pub type PoundForceSeconds = f64;

/// Fire the thrusters. Returns generated impulse.
pub fn thruster_impulse(direction: Direction) -> PoundForceSeconds {
    // ...
    return 42.0;
}
```

```rust
/// Units for force.
pub type NewtonSeconds = f64;

/// Update trajectory model for impulse.
pub fn update_trajectory(force: NewtonSeconds) {
    // ...
}
```

然而，类型别名只是字面上的约束，它们是比以前版本的文档主食更强的提示，但是并没有直接阻止在需要使用`NewtonSeconds`值的地方传递`PoundForceSeconds`值。

```rust
let thruster_force: PoundForceSeconds = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force);
```

哦不（不知道如何翻译）

从这里引出就是我们newtype模式的用武之地了：

```rust
/// Units for force.
pub struct PoundForceSeconds(pub f64);

/// Fire the thrusters. Returns generated impulse.
pub fn thruster_impulse(direction: Direction) -> PoundForceSeconds {
    // ...
    return PoundForceSeconds(42.0);
}
```

```rust
/// Units for force.
pub struct NewtonSeconds(pub f64);

/// Update trajectory model for impulse.
pub fn update_trajectory(force: NewtonSeconds) {
    // ...
}
```

顾名思义，newtype是一种新类型，因此当类型不匹配时候，编译器直接会进行拦截，这里尝试将`PoundForceSeconds`传递给需要 `NewtonSeconds`的地方会导致编译错误：

```rust
let thruster_force: PoundForceSeconds = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force);
```

```shell
error[E0308]: mismatched types
  --> src/main.rs:76:43
   |
76 |     let new_direction = update_trajectory(thruster_force);
   |                         ----------------- ^^^^^^^^^^^^^^ expected
   |                         |        `NewtonSeconds`, found `PoundForceSeconds`
   |                         |
   |                         arguments to this function are incorrect
   |
note: function defined here
  --> src/main.rs:66:8
   |
66 | pub fn update_trajectory(force: NewtonSeconds) {
   |        ^^^^^^^^^^^^^^^^^ --------------------
help: call `Into::into` on this expression to convert `PoundForceSeconds` into
      `NewtonSeconds`
   |
76 |     let new_direction = update_trajectory(thruster_force.into());
   |                                                         +++++++
```

根据[条目 5](./Item5.md)的描述，我们给`NewtonSeconds`类型添加标准的`From`特征的实现

```rust
impl From<PoundForceSeconds> for NewtonSeconds {
    fn from(val: PoundForceSeconds) -> NewtonSeconds {
        NewtonSeconds(4.448222 * val.0)
    }
}
```

这样我们就允许使用`.into()`方法进行类型的转换：

```rust
let thruster_force: PoundForceSeconds = thruster_impulse(direction);
let new_direction = update_trajectory(thruster_force.into());
```

使用新类型来标记类型的附加单位语意可以帮助使布尔参数不那么容易混淆。重新审视[条目 1](./Item1.md)中的示例，使用newtype模式使参数的意图更加明确：

```rust
struct DoubleSided(pub bool);

struct ColorOutput(pub bool);

fn print_page(sides: DoubleSided, color: ColorOutput) {
    // ...
}
```

```rust
print_page(DoubleSided(true), ColorOutput(false));
```

如果需要考虑大小效率以及二进制兼容性，则可以使用`repr(transparent)`属性可确保新类型在内存中具有与内部类型相同的表示形式。

这就是newtype模式的简单用法，也是[条目 1](./Item1.md)的具体示例 -- 将语意编码到类型系统中，方便编译器负责管理这些语意，避免程序员犯错。

## 绕过特征的孤儿规则

另一个常见但是更加微妙的需要newtype模式的场景是围绕着Rust的孤儿规则的。粗略地讲，这表示只有满足如下条件之一，crate才能实现一个trait：

* crate已经定义了这个trait
* crate有需要实现trait的这个类型

尝试为外来类型实现外来特征：

```rust
use std::fmt;

impl fmt::Display for rand::rngs::StdRng {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> Result<(), fmt::Error> {
        write!(f, "<StdRng instance>")
    }
}
```

会导致编译器报错（这反过来又需要newtype模式来解决）

```shell
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
   --> src/main.rs:146:1
    |
146 | impl fmt::Display for rand::rngs::StdRng {
    | ^^^^^^^^^^^^^^^^^^^^^^------------------
    | |                     |
    | |                     `StdRng` is not defined in the current crate
    | impl doesn't use only types from inside the current crate
    |
    = note: define and implement a trait or new type instead
```

这种限制的原因是由于存在有歧义的风险：如果依赖图中的两个不同的crate(条目 25)都有`impl std::fmt::Display for rand::rngs::StdRng`，那么编译器/链接器就无法在两者之间进行选择。

这样经常导致编译的冲突和挫败感，例如，如果你尝试给来自另外一个crate的类型实现`serialize data`，孤儿元组则会阻止你编写`impl serde::Serialize for somecrate::SomeType`

但是newtype模式意味着你正在定义一个新类型，它就是当前crate的一部分，因此就可以满足孤儿原则的第二个条件，现在就可以实现外部的trait：

```rust
struct MyRng(rand::rngs::StdRng);

impl fmt::Display for MyRng {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> Result<(), fmt::Error> {
        write!(f, "<MyRng instance>")
    }
}
```

## newtype的限制

newtype模式解决了上述两类问题 -- 防止了单位相同类型的转换和绕过孤儿规则 -- 但是它也带来了一些尴尬：涉及newtype的每个操作都需要转发到其内部类型。

简单的考虑就意味着代码必须使用`thing.0`,而不是`thing`，这个很简单，编译器会告诉你哪里需要这么做。

更尴尬的是是内部类型上的任何特征实现都丢失了，因为newtype是一个新类型，所以现有的内部实现都不得适用。

对于可派生特征，这仅意味着newtype需要声明`derive`属性：

```rust
#[derive(Debug, Copy, Clone, Eq, PartialEq, Ord, PartialOrd)]
pub struct NewType(InnerType);
```

然而对于更加复杂的特征来说，需要一些转发样板来恢复内部类型的实现，例如：

```rust
use std::fmt;
impl fmt::Display for NewType {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> Result<(), fmt::Error> {
        self.0.fmt(f)
    }
}
```