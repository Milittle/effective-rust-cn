# 条目 5：了解类型转换

Rust类型转换分为三类：

* 手动：用户通过`From`和`Into` trait来实现类型转换。
* 半自动：适用`as`关键字在值类型之间进行转换。
* 自动：隐式强制转换为新类型。

这个条目主要介绍第一点因为后两者的转换不适用于用户自定义类型的转换，对此有一些例外，因为本条目末尾的部分会讨论强制转换和，包括它们如何应用与用户自定义类型。

需要注意的是，在Rust中和其他语言不通，默认不会进行数字类型之间的自动转换。哪怕是整数类型的安全转换也不允许自动转换。

```rust
let x: u32 = 2;
let y: u64 = x;
```

```text
error[E0308]: mismatched types
  --> src/main.rs:70:18
   |
70 |     let y: u64 = x;
   |            ---   ^ expected `u64`, found `u32`
   |            |
   |            expected due to this
   |
help: you can convert a `u32` to a `u64`
   |
70 |     let y: u64 = x.into();
   |                   +++++++
```

## 用户定义的类型转换

与该语言的其他功能一直（条目 10），在不同用户自定义类型之间进行转换的能力被封装为标准`trait`,或者更加确切的讲，封装为一组相关的通用`trait`。

表达转换类型值的能力的四个相关的`trait`:

* [`Frome<T>`](https://doc.rust-lang.org/std/convert/trait.From.html): 从类型`T`到此类型的转换。而且转换总是会成功。
* [`TryFrom<T>`](https://doc.rust-lang.org/std/convert/trait.TryFrom.html): 从类型`T`到此类型的转换。转换可能会失败。
* [`Into<T>`](https://doc.rust-lang.org/std/convert/trait.Into.html): 从此类型到类型`T`的转换。而且转换总是会成功。
* [`TryInto<T>`](https://doc.rust-lang.org/std/convert/trait.TryInto.html): 从此类型到类型`T`的转换。转换可能会失败。

在[条目 1](./Item1.md)中我们讨论过适用类型系统来表达数据结构。不难发现`Try...`变体的区别对于`trait`唯一的方法来说返回的是一个`Result`而不是保障一定有一个新的转换类型。`Try..` trait定义还包含一个关联类型，需要给出当失败时候应该返回的错误类型E。

因此，在这里有第一条建议，如果我们的转换可能失败，则实现`Try...` trait,与[条目 4](./Item4.md)一致。另一种方式是直接使用`.unwarp()`方法来忽略可能的错误，但是这样做必须经过深思熟虑，而且最好大多数情况把这个决定权交给调用者。

类型转换trait具有明显的对称性：如果类型`T`可以转换为`U`(通过`Into<U>`),那么`U`可以转换为`T`(通过`From<T>`)。

情况确实如此，这就引出第二条建议：实现`From` trait。Rust标准库必须让实现者选择两个中一个trait来实现，以防止迷人眼，所以标准库根据`From` trait的实现自动提供了`Into` trait的实现。

如果你在自己的泛型类型上使用这两个特质之一作为特征约束（trait bounds），那么建议你反过来：使用 Into 特征作为特征约束（trait bounds）。这样，既可以满足直接实现 Into 的对象，也可以满足仅直接实现 From 的对象。

这种自动由 From 和 Into的转换在文档中有说明，但也非常值得阅读标准库代码中的相关部分，那里包含了一个全覆盖的特征（trait）实现：

```rust
impl<T, U> Into<U> for T 
where 
    U: From<T> 
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

将特征规范翻译成大白话有利于帮助我们理解复杂的特征约束，在这个案例中，可以简单的描述为：当类型`U`实现了`From<T>`，那么就为类型`T`实现`Into<U>`。

标准库类型包括了这些转换特征的各种实现。正如你所期望的，对于目标类型包含源类型所有可能值的整数转换有 From 实现（例如，`From<u32>` for `u64`），当源可能不适合目标时有 TryFrom 实现（例如，`TryFrom<u64>` for `u32`）。

除了之前展示的 `Into` 特征的实现之外，还有其他各种全覆盖的特征实现；这些主要是针对智能指针类型，允许智能指针从其持有的类型实例自动构建。这意味着接受智能指针参数的泛型方法也可以用普通旧项目调用；关于这一点更多讨论，会在条目 8中详述。

`TryFrom` 特征也为任何自动实现（如前所示）了 `From` 的任何类型已经在相反方向上实现了 `Into` 特征的类型提供了全覆盖实现 -- 换句话说，如果你可以无误地将 `T` 转换为 `U`，你也可以可靠地从 `T` 获得 `U`；由于这种转换总是会成功，相关的错误类型被命名为 [Infallible](https://doc.rust-lang.org/std/convert/enum.Infallible.html)。

还有一个非常特殊的 `From` 泛型实现，那就是 _反射实现_：

```rust
impl<T> From<T> for T {
    fn from(t: T) -> T {
        t
    }
}
```

用大白话表达，这在讲“给定一个 `T`，我可以得到一个 `T`。”这是显而易见的“嗯，当然”，值得我们停下来思考为什么这个实现会有用。

考虑一个简单的新类型`struct`(条目 6)，并且有一个函数会操作它（忽略这个函数可以表达为一个类方法）：

```rust
/// Integer value from an IANA-controlled range.
#[derive(Clone, Copy, Debug)]
pub struct IanaAllocated(pub u64);

/// Indicate whether value is reserved.
pub fn is_iana_reserved(s: IanaAllocated) -> bool {
    s.0 == 0 || s.0 == 65535
}
```

这个函数可以适用这个类的实例进行调用:

```rust
let s = IanaAllocated(1);
println!("{:?} reserved? {}", s, is_iana_reserved(s));
// output: "IanaAllocated(1) reserved? false"
```
但即使为新类型封装实现了 `From<u64>`：

```rust
impl From<u64> for IanaAllocated {
    fn from(v: u64) -> Self {
        Self(v)
    }
}
```

这个函数不能通过 `u64` 直接调用：

```shell
if is_iana_reserved(42) {
    // ...
}
```

```text
error[E0308]: mismatched types
  --> src/main.rs:77:25
   |
77 |     if is_iana_reserved(42) {
   |        ---------------- ^^ expected `IanaAllocated`, found integer
   |        |
   |        arguments to this function are incorrect
   |
note: function defined here
  --> src/main.rs:7:8
   |
7  | pub fn is_iana_reserved(s: IanaAllocated) -> bool {
   |        ^^^^^^^^^^^^^^^^ ----------------
help: try wrapping the expression in `IanaAllocated`
   |
77 |     if is_iana_reserved(IanaAllocated(42)) {
   |                         ++++++++++++++  +
```

然而，一个泛型函数版本接受（并明确转换）任何满足         `Into<IanaAllocated>` 的类型：

```rust
pub fn is_iana_reserved<T>(s: T) -> bool
where
    T: Into<IanaAllocated>,
{
    let s = s.into();
    s.0 == 0 || s.0 == 65535
}
```

才允许这样使用：

```rust
if is_iana_reserved(42) {
    // ...
}
```

有了这个特征约束，`From<T>` 的反射特征实现更有意义了：它意味着泛型函数可以处理已经是 `IanaAllocated` 实例的项目，无需转换。(译者注：目前还未吃透该概念，需要进一步的学习)

这种模式还解释了为什么（以及如何）Rust 代码有时似乎在类型之间进行隐式转换：`From<T>` 的实现和 `Into<T>` 的特征约束的组合导致代码看起来在调用点神奇地转换（但仍然在底层进行安全、明确的转换）。当这种模式与引用类型及其相关的转换特征结合时，它变得更加强大；更多内容见条目 8。

## 显示强制转换（Casts）

Rust还有一个`as`关键字去执行类型之间显示的[转换](https://doc.rust-lang.org/reference/expressions/operator-expr.html#type-cast-expressions)。

这些类型之间的类型构成了一个相当有限的集合，包括标准库基础的整数类型(也提供了可选的`into`方法转换)，以及用户自定义的简单`enum`类型(仅仅是关联了整数值的类型)。

```rust
let x: u32 = 9;
let y = x as u64;
let z: u64 = x.into();
```

`as`转换还允许有损失的转换：

```rust
let x: u32 = 9;
let y = x as u16;
```

这种转换会被`from`和`into`方法拒绝：

```text
error[E0277]: the trait bound `u16: From<u32>` is not satisfied
   --> src/main.rs:136:20
    |
136 |     let y: u16 = x.into();
    |                    ^^^^ the trait `From<u32>` is not implemented for `u16`
    |
    = help: the following other types implement trait `From<T>`:
              <u16 as From<NonZeroU16>>
              <u16 as From<bool>>
              <u16 as From<u8>>
    = note: required for `u32` to implement `Into<u16>`
```

为了一致性和安全性，你应该优先适用`from`和`into`方法，而不是`as`转换，除非你理解更加精确的[转换语意](https://doc.rust-lang.org/reference/expressions/operator-expr.html#semantics)（例如，为了和C之间的互相操作性）。这个建议可以通过`Clippy`工具进一步强化（条目 29），其中包括一些关于[`as`转换](https://rust-lang.github.io/rust-clippy/stable/index.html#/as_conversions)的lints;但是在默认情况下这些lint是关闭的。


## 隐式强制转换（Coercion）

前一节描述的显式 `as` 转换是编译器将默默执行的隐式强制转换的超集：任何隐式强制转换都可以通过显式的 `as` 来显示强制转换，但反之则不成立。特别是，前一节进行的整数转换不是隐式强制转换，因此总是需要使用 `as`。

大多数隐式强制转换涉及指针和引用类型的无声转换，这些转换对程序员来说是合理且方便的，类似的转换有：

* 从一个可变引用到一个不可变引用。（一个接受`&T`类型的函数可以使用一个`&mut T`的类型来作为的参数）
* 一个引用到一个原生指针（这不是`unsafe`的 -- 其实不安全的行为是发生在你无知的对一个原生指针进行解引用的时候）
* 一个没有捕获任何变量的闭包到一个函数指针（[条目 2](./Item2.md)）
* 一个[数组](https://doc.rust-lang.org/std/primitive.array.html)到一个[切片](https://doc.rust-lang.org/std/primitive.slice.html)
* 一个实现了trait的具体类型到该[trait对象](https://doc.rust-lang.org/reference/types/trait-object.html)
* 一个从厂生命周期到短生命周期（条目 14）

只有两种隐式强制转换的行为会受到用户定义类型的影响。第一种情况发生在用户定义的类型实现了 [`Deref`](https://doc.rust-lang.org/std/ops/trait.Deref.html) 或 [`DerefMut`](https://doc.rust-lang.org/std/ops/trait.DerefMut.html) 特征时。这些特征表明用户定义的类型在某种程度上充当智能指针（见条目 8），在这种情况下，编译器将强制将对智能指针项的引用转换为对智能指针所包含的类型项的引用（由其特征的 [`Target`](https://doc.rust-lang.org/std/ops/trait.Deref.html#associatedtype.Target)关联类型指定）。

第二种用户定义类型的隐式强制转换发生在将具体对象转换为特征对象时。这个操作构建了一个具体对象的胖指针；之所以称为胖指针，是因为它包括了具体对象在内存中位置的指针和指向具体类型实现特征的虚表的指针 -- 见条目 8。
