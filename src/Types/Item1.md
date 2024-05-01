# 条目 1: 使用类型系统表达你的数据结构

```
"who called them programers and not type writers" – @thingskatedid
```

Rust类型系统的基础对于熟悉另一种静态类型编程语言（如C++、Go或Java）的人来说相当熟悉。这里有一系列具体大小的整数类型，包括有符号（[`i8`](https://doc.rust-lang.org/std/primitive.i8.html)、[`i16`](https://doc.rust-lang.org/std/primitive.i16.html)、[`i32`](https://doc.rust-lang.org/std/primitive.i32.html)、[`i64`](https://doc.rust-lang.org/std/primitive.i64.html)、[`i128`](https://doc.rust-lang.org/std/primitive.i64.html)）和无符号（[`u8`](https://doc.rust-lang.org/std/primitive.u8.html)、[`u16`](https://doc.rust-lang.org/std/primitive.u16.html)、[`u32`](https://doc.rust-lang.org/std/primitive.u32.html)、[`u64`](https://doc.rust-lang.org/std/primitive.u64.html)、[`u128`](https://doc.rust-lang.org/std/primitive.u128.html)）。

还有有符号（[`isize`](https://doc.rust-lang.org/std/primitive.isize.html)）和无符号（[`usize`](https://doc.rust-lang.org/std/primitive.usize.html)）整数，其大小与目标系统上的指针大小相匹配。Rust不是一种需要在指针和整数之间进行大量转换的语言，因此这种描述并不真正相关。然而，标准集合以`usize`（来自`.len()`）返回它们的大小，所以集合索引意味着usize值相当常见——从容量角度看这显然是可以的，因为系统上的内存地址数量多于内存中集合的项目数量。

整数类型确实给我们提供了第一个暗示，即Rust是一个比C++更严格的世界——试图将（`i32`）放入（`i16`）的容器中会生成一个编译时错误。

```rust
         let x: i32 = 42;
         let y: i16 = x;
```

这段代码会产生如下错误：

```shell
error[E0308]: mismatched types
  --> use-types/src/main.rs:14:22
   |
14 |         let y: i16 = x;
   |                ---   ^ expected `i16`, found `i32`
   |                |
   |                expected due to this
   |
help: you can convert an `i32` to an `i16` and panic if the converted value doesn't fit
   |
14 |         let y: i16 = x.try_into().unwrap();
   |                       ++++++++++++++++++++
```

这令人非常安心：Rust不会在程序员进行风险操作时坐视不管。Rust不但显示有更严格的规则，而且它也有有用的编译器提示，指出如何遵守这些规则。建议的解决方案引发了出一个问题，即如何处理由于转换而可能改变值的情况，我们稍后还会在错误处理（条目 4）和使用`panic!`（条目 18）上有更多讨论。

Rust还不允许一些可能看似 “安全” 操作的转换：

```rust
        let x = 42i32; // Integer literal with type suffix
        let y: i64 = x;
```

这段代码会产生如下错误：

```shell
error[E0308]: mismatched types
  --> use-types/src/main.rs:23:22
   |
23 |         let y: i64 = x;
   |                ---   ^ expected `i64`, found `i32`
   |                |
   |                expected due to this
   |
help: you can convert an `i32` to an `i64`
   |
23 |         let y: i64 = x.into();
   |                       +++++++

```

在这里，建议的解决方案并没有提出错误处理的问题，但转换仍然需要是显式的。我们稍后将更详细地讨论类型转换（条目 6）。

继续介绍基本的原始类型，Rust包括用于布尔值的[`bool`](https://doc.rust-lang.org/std/primitive.bool.html)类型，浮点类型（[`f32`](https://doc.rust-lang.org/std/primitive.f32.html)、[`f64`](https://doc.rust-lang.org/std/primitive.f64.html)）以及类似于C语言中的void类型[`()`](https://en.wikipedia.org/wiki/Unit_type)。

更值得注意的是，Rust具有[`char`](https://doc.rust-lang.org/std/primitive.char.html)类型，它存储一个[Unicode值](http://www.unicode.org/glossary/#unicode_scalar_value)，类似于Go的[`rune`](https://golang.org/doc/go1#rune)类型。虽然内部存储为4字节，但Rust不允许将其自由地与32位整型进行互相转换。

这种类型系统的严谨性强制你在代码中明确表达 —— 一个`u32`值与一个`char`不同，与一串UTF-8字节不同，与一串任意字节不同。定义你正在操作的内存取决于你。乔尔·斯波尔斯基（Joel Spolsky）关于Unicode的[著名博客文章](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)可以帮助理解这些差异。

虽然Rust提供了帮助方法来转换这些不同的类型，但它们的签名要求你管理或明确忽略转换错误的风险。例如，任何Unicode代码点都可以用32位表示，因此将'a'转换为u32是允许的。然而，从u32转换为char则更复杂，因为可能存在无效的Unicode代码点。

* `char::from_u32` 返回一个`Option<char>`，强制要求调用者来处理错误的情况
* `char::from_u32_unchecked` 假定输入是有效的Unicode代码点，这样就可以避免运行时检查，但是它的结果是需要标记位`unsafe`的，强制调用者是unsafe的使用(条目 16 相关)


# 聚合类型（Aggregate Types）

继续讨论聚合类型，Rust有：

* 数组（Arrays），它们包含多个单一类型的实例，其中实例的数量在编译时已知。例如，[u32; 4] 是连续的四个4字节整数。
* 元组（Tuples），它包含很多异构类型的实例，其中实例的数量和类型在编译时已知。例如，(WidgetOffset, WidgetSize, WidgetColour)。如果元组中的类型不是独特的 —— 例如(i32, i32, &'static str, bool) —— 最好给每个元素一个名字，并使用结构体
* 结构体（Structs），它们也包含在编译时已知的异质类型的实例，但允许通过名称引用整体类型和各个字段。

元组结构是结构体和元组的混合体：整体类型有一个名称，但个别字段没有名称——它们通过数字来引用，例如：s.0, s.1 等。

```rust
#![allow(unused)]
fn main() {
    struct TextMatch(usize, String);
    let m = TextMatch(12, "needle".to_owned());
    assert_eq!(m.0, 12);
}
```

这引出了Rust类型系统中的王冠上的宝石 —— `enum`。

在其基本形式中，很难看出有什么值得激动的地方。与其他语言类似，枚举允许你指定一组互斥的值，可能附带有数字或字符串值。

```rust
#![allow(unused)]
fn main() {
    enum HttpResultCode {
        Ok = 200,
        NotFound = 404,
        Teapot = 418,
    }
    let code = HttpResultCode::NotFound;
    assert_eq!(code as i32, 404);
}
```

因为每一个`enum`定义创建了不同的类型，这个可以提高如下函数的可读性以及维护性：

```rust
        print_page(/* both_sides= */ true, /* colour= */ false);
```

上述函数的使用`enum`的新版本：

```rust
#![allow(unused)]
fn main() {
    pub enum Sides {
        Both,
        Single,
    }

    pub enum Output {
        BlackAndWhite,
        Colour,
    }

    pub fn print_page(sides: Sides, colour: Output) {
        // ...
    }
}
```

这种方式类型安全更加有保障，而且给调用者来说也是更加直观的。

```rust
        print_page(Sides::Both, Output::BlackAndWhite);
```

不像`bool`的函数版本，如果库的使用者不小心将参数的顺序搞错了，那么编译器就会立即抱怨，让你知道你犯了错误。

```shell
error[E0308]: mismatched types
  --> use-types/src/main.rs:89:20
   |
89 |         print_page(Output::BlackAndWhite, Sides::Single);
   |                    ^^^^^^^^^^^^^^^^^^^^^ expected enum `enums::Sides`, found enum `enums::Output`
error[E0308]: mismatched types
  --> use-types/src/main.rs:89:43
   |
89 |         print_page(Output::BlackAndWhite, Sides::Single);
   |                                           ^^^^^^^^^^^^^ expected enum `enums::Output`, found enum `enums::Sides`

```

（使用新类型模式（条目 7）来包装布尔值也可以实现类型安全和可维护性；如果语义将始终是布尔值，通常最好使用该方法，并且如果将来可能出现新的替代项（例如 Sides::BothAlternateOrientation），则使用枚举。）

Rust的枚举的类型安全性在匹配表达式中继续体现：

```rust
        let msg = match code {
            HttpResultCode::Ok => "Ok",
            HttpResultCode::NotFound => "Not found",
            // forgot to deal with the all-important "I'm a teapot" code
        };
```

这段代码会产生如下错误：

```shell
error[E0004]: non-exhaustive patterns: `Teapot` not covered
  --> use-types/src/main.rs:65:25
   |
51 | /     enum HttpResultCode {
52 | |         Ok = 200,
53 | |         NotFound = 404,
54 | |         Teapot = 418,
   | |         ------ not covered
55 | |     }
   | |_____- `HttpResultCode` defined here
...
65 |           let msg = match code {
   |                           ^^^^ pattern `Teapot` not covered
   |
   = help: ensure that all possible cases are being handled, possibly by adding wildcards or more match arms
   = note: the matched value is of type `HttpResultCode`

```

编译器强制程序员考虑枚举表示的所有可能性，即使结果只是添加一个默认分支 _ => {}。（需要注意的是，现代C++编译器也可以警告缺少枚举`switch`分支。）

# 带字段的enums

Rust枚举功能的真正威力来自于每个变体都可以携带数据，使其成为代数数据类型（ADT）。对于主流语言的程序员来说，这可能不太熟悉；从C/C++的角度来看，它就像是枚举与联合的组合 —— 只不过是类型安全的。

这意味着程序数据结构的不变性可以编码到Rust的类型系统中；不符合这些不变性的状态甚至不会编译通过。一个设计良好的枚举能够使创作者的意图清晰地传达给人类和编译器：

```rust
pub enum SchedulerState {
    Inert,
    Pending(HashSet<Job>),
    Running(HashMap<CpuId, Vec<Job>>),
}
```

仅仅从类型定义，我们可以合理地猜测，任务会在挂起状态中排队，直到调度程序完全激活，然后它们会分配给每个CPU池。

这突显了这个项目的中心主题，即利用Rust的类型系统来表达与软件设计相关的概念。

一个明显的迹象表明这种情况并未发生是，当一些字段或参数的有效性需要解释时，会使用注释说明：

```rust
struct DisplayProps {
    x: u32,
    y: u32,
    monochrome: bool,
    // `fg_colour` must be (0, 0, 0) if `monochrome` is true.
    fg_colour: RgbColour,
}
```

这是一个非常适合用包含数据的枚举替换的候选对象：

```rust
#[derive(Debug)]
enum Colour {
    Monochrome,
    Foreground(RgbColour),
}

struct DisplayProperties {
    x: u32,
    y: u32,
    colour: Colour,
}
```

这个小例子说明了一个关键建议：使类型中的无效状态来表达无效。只支持有效值组合的类型意味着编译器会拒绝整类错误，从而产生更小、更安全的代码。

# Options 和 Errors

回到枚举的强大之处，有两个概念类型是如此常见，以至于Rust包含了内置的枚举类型来表达它们。

第一个概念是`Option`：要么有特定类型的值（`Some(T)`），要么没有（`None`）。对于可能不存在的值，始终使用`Option`；永远不要退回到使用标记值（-1、nullptr等）来尝试表示相同的概念。

然而，还有一个微妙需要考虑的地方。如果你处理的是一组集合数据，你需要考虑集合中是否存在数据与没有集合是相同的。对于大多数情况，并不需要明确区分这二者的区别，你可以直接使用Vec<Thing>：当里面不包含数据的时候就是代表没有这个集合数据。

当然，确实也存在其他少见的情况，需要使用Option<Vec<Thing>> 来区分这两种情况 —— 例如，加密系统可能需要区分“分开传输的有效载荷”和“提供了空有效载荷”。（这与SQL中的NULL标记列的争论有关。）（译者注释：相当于是编程中空是否赋予了含义）

一个常见的临界情况是可能不存在的字符串 - 使用 "" 还是 None 更合理来表示值的缺失？两种方式都可以，但 Option<String> 清楚地传达了这个值可能不存在的可能性。

第二个常见概念源自错误处理：如果一个函数失败了，该如何报告这个失败？在历史上，特殊的标记值（例如来自Linux系统调用的 -errno 返回值）或全局变量（POSIX系统的 errno）被使用。最近，支持从函数返回多个或元组返回值的语言（如Go）可能有一个约定，即在错误为非“零”时返回（结果，错误）对。

在Rust中，总是将可能失败的操作的结果编码为 Result<T, E>。T 类型保存成功的结果（在 Ok 变体中），E 类型在失败时保存错误详情（在 Err 变体中）。使用标准类型可以清晰地表达设计意图，并允许使用标准转换（条目 3）和错误处理（条目 4）；它还使得通过 ? 操作符来简化错误处理成为可能。