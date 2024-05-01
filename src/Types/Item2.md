# 条目2：使用类型系统来表达常见行为

[条目 1](./Item1.md) 讨论了如何使用在类型系统中表达数据结构；本项目将讨论如何使用类型系统来表达常见行为。

# 方法（Methods）

首先，Rust类型系统的第一个行为就是向数据结构中添加方法：这些方法作用于某个类型的成员，由 self 标识。这种方式将相关数据和代码以面向对象的方式封装在一起，与其他语言类似；然而，在 Rust 中，方法不仅可以添加到结构类型，还可以添加到枚举类型，这与 Rust 枚举的普遍性质保持一致[条目 1](./Item1.md) 。

```rust
enum Shape {
    Rectangle { width: f64, height: f64 },
    Circle { radius: f64 },
}

impl Shape {
    pub fn area(&self) -> f64 {
        match self {
            Shape::Rectangle { width, height } => width * height,
            Shape::Circle { radius } => std::f64::consts::PI * radius * radius,
        }
    }
}
```

方法的名称为它所表示的行为提供了一个标签，而方法签名为其输入和输出提供了类型信息。方法的第一个输入将是 self 的某个变体，指示该方法可能对数据结构执行的操作：
* &self: 该参数表示方法可能读取数据结构的数据，但是并不会修改数据结构的数据内容
* &mut self: 该参数表示方法可能修改数据结构的数据内容
* self: 该参数表示方法消耗数据结构的数据，这意味着调用者将无法再次使用数据结构（移动语意）

# 抽象行为 （Abstracting Behaviour）

调用一个方法总是导致执行相同的代码；从一次调用到另一次调用，变化的只是方法操作的数据。这涵盖了许多可能的场景，但如果代码需要在运行时变化怎么办？

Rust 在其类型系统中包含了几个特性以适应这种情况，本节将探讨这些特性。

## （函数指针）Function Pointers

最简单的行为抽象就是[函数指针}(https://doc.rust-lang.org/std/primitive.fn.html): 指向（仅仅是）一些代码的指针，其类型反映了函数签名。该类型在编译时进行检查，因此到程序运行时，该值仅为一个指针的大小。

```rust
    fn sum(x: i32, y: i32) -> i32 {
        x + y
    }
    // Explicit coercion to `fn` type is required...
    let op: fn(i32, i32) -> i32 = sum;
```

函数指针没有与之相关联的其他数据,因此它们可以通过各种方式被当作值来处理:

```rust
    fn sum(x: i32, y: i32) -> i32 {
        x + y
    }
    // `fn` types implement `Copy`
    let op1 = op;
    let op2 = op;
    // `fn` types implement `Eq`
    assert!(op1 == op2);
    // `fn` implements `std::fmt::Pointer`, used by the {:p} format specifier.
    println!("op = {:p}", op);
    // Example output: "op = 0x101e9aeb0"
```

需要注意的一个技术细节：需要使用显示强制类型转换为 `fn` 类型，因为仅仅使用函数名称你不会给你提供 `fn` 类型。

```rust
    let op1 = sum;
    let op2 = sum;
    // Both op1 and op2 are of a type that cannot be named in user code,
    // and this internal type does not implement `Eq`.
    assert!(op1 == op2);
```

```shell
   Compiling playground v0.0.1 (/playground)
error[E0369]: binary operation `==` cannot be applied to type `fn(i32, i32) -> i32 {main::sum}`
  --> src/main.rs:19:18
   |
19 |     assert!(op11 == op22);
   |             ---- ^^ ---- fn(i32, i32) -> i32 {main::sum}
   |             |
   |             fn(i32, i32) -> i32 {main::sum}
   |
help: use parentheses to call these
   |
19 |     assert!(op11(/* i32 */, /* i32 */) == op22(/* i32 */, /* i32 */));
   |                 ++++++++++++++++++++++        ++++++++++++++++++++++

For more information about this error, try `rustc --explain E0369`.
error: could not compile `playground` (bin "playground") due to 1 previous error
```

相反,编译器错误表明该类型是类似于`fn(i32, i32) -> i32 {main::sum}`的东西,这是一种完全内部的编译器类型(即无法在用户代码中编写),它不仅标识了具体的函数,还标识了它的签名。换句话说,`sum`的类型同时编码了函数的签名和它的位置([出于优化原因](https://doc.rust-lang.org/std/primitive.fn.html#creating-function-pointers));但是这种类型可以自动强制转换为`fn`类型(条目 6)。

## 闭包（Closures）

裸函数指针是有一些限制的，因为裸函数指针唯一可用的输入就是作为参数值显示传递的输入。

例如，考虑到使用一个函数指针来修改切片的元素：

```rust
    // In real code, an `Iterator` method would be more appropriate.
    pub fn modify_all(data: &mut [u32], mutator: fn(u32) -> u32) {
        for value in data {
            *value = mutator(*value);
        }
    }
```

这个试用一些对切片简单的修改：

```rust
    pub fn modify_all(data: &mut [u32], mutator: fn(u32) -> u32) {
        for value in data {
            *value = mutator(*value);
        }
    }
    fn add2(v: u32) -> u32 {
        v + 2
    }
    let mut data = vec![1, 2, 3];
    modify_all(&mut data, add2);
    assert_eq!(data, vec![3, 4, 5,]);
```

然而，如果我们的修改需要依赖外部的数据状态修改，这个就不太可能使用函数指针来做到这个事情。

```rust
    pub fn modify_all(data: &mut [u32], mutator: fn(u32) -> u32) {
        for value in data {
            *value = mutator(*value);
        }
    }
    let amount_to_add = 3;
    fn add_n(v: u32) -> u32 {
        v + amount_to_add
    }
    let mut data = vec![1, 2, 3];
    modify_all(&mut data, add_n);
    assert_eq!(data, vec![4, 5, 6,]);
```

```shell
   Compiling playground v0.0.1 (/playground)
error[E0434]: can't capture dynamic environment in a fn item
 --> src/main.rs:9:13
  |
9 |         v + amount_to_add
  |             ^^^^^^^^^^^^^
  |
  = help: use the `|| { ... }` closure form instead

For more information about this error, try `rustc --explain E0434`.
error: could not compile `playground` (bin "playground") due to 1 previous error
```

这个错误信息表明，我们正确的选择是使用闭包来代替函数指针：闭包是一段看起来像函数定义主体的代码(一个lambda表达式),只不过:
* 它可以作为表达式的一部分，因为可以不需要一个单独的名称来引用它
* 输入的参数使用`||`包裹，类似`|param1, param2|`(这里参数的关联类型通常可以由编译器自动推导)
* 闭包还可以捕获上下文的变量

```rust
    let amount_to_add = 3;
    let add_n = |y| {
        // a closure capturing `amount_to_add`
        y + amount_to_add
    };
    let z = add_n(5);
    assert_eq!(z, 8);
```

为了（粗略地）理解捕获是如何进行工作的，假设编译器创建了一个一次性的内部类型，它包含lambda表达式中提到的环境上下文的所有部分。创建闭包是，会创建临时的类型来保存相关值，并且当调用闭包时，该实例将其附加在调用上下文：

```rust
    let amount_to_add = 3;
    // *Rough* equivalent to a capturing closure.
    struct InternalContext<'a> {
        // references to captured variables
        amount_to_add: &'a u32,
    }
    impl<'a> InternalContext<'a> {
        fn internal_op(&self, y: u32) -> u32 {
            // body of the lambda expression
            y + *self.amount_to_add
        }
    }
    let add_n = InternalContext {
        amount_to_add: &amount_to_add,
    };
    let z = add_n.internal_op(5);
    assert_eq!(z, 8);
```

这些值在上下文保存的通常是不可变引用，但是也可以作为可变引用，或者是所有权的转移。这些值的生命周期是由闭包的生命周期来决定的。（通过使用 `move` 关键字来进行所有权的移动）

回到我们`modify_all`的示例中，在使用函数指针的地方抱怨不能使用闭包：

```rust
    pub fn modify_all(data: &mut [u32], mutator: fn(u32) -> u32) {
        for value in data {
            *value = mutator(*value);
        }
    }
    let amount_to_add = 3;
    let mut data = vec![1, 2, 3];
    modify_all(&mut data, |y| y + amount_to_add);
    assert_eq!(data, vec![4, 5, 6,]);
```

```shell
   Compiling playground v0.0.1 (/playground)
error[E0308]: mismatched types
 --> src/main.rs:9:27
  |
9 |     modify_all(&mut data, |y| y + amount_to_add);
  |     ----------            ^^^^^^^^^^^^^^^^^^^^^ expected fn pointer, found closure
  |     |
  |     arguments to this function are incorrect
  |
  = note: expected fn pointer `fn(u32) -> u32`
                found closure `{closure@src/main.rs:9:27: 9:30}`
note: closures can only be coerced to `fn` types if they do not capture any variables
 --> src/main.rs:9:35
  |
9 |     modify_all(&mut data, |y| y + amount_to_add);
  |                                   ^^^^^^^^^^^^^ `amount_to_add` captured here
note: function defined here
 --> src/main.rs:2:12
  |
2 |     pub fn modify_all(data: &mut [u32], mutator: fn(u32) -> u32) {
  |            ^^^^^^^^^^                   -----------------------

For more information about this error, try `rustc --explain E0308`.
error: could not compile `playground` (bin "playground") due to 1 previous error
```

相反,接收闭包的代码必须接受某个 `Fn*` trait 的实例。

```rust
    pub fn modify_all<F>(data: &mut [u32], mut mutator: F)
    where 
        F: FnMut(u32) -> u32
    {
        for value in data {
            *value = mutator(*value);
        }
    }
    let amount_to_add = 3;
    let mut data = vec![1, 2, 3];
    modify_all(&mut data, |y| y + amount_to_add);
    assert_eq!(data, vec![4, 5, 6,]);
```

Rust有三种不同的 `Fn*` trait，他们之间表达了该环境变量捕获的一些区别：

* `FnOnce`：描述了只能调用一次的闭包，如果其环境变量的某些部分 `move` 进入闭包，那么`move`只能发生一次 - 对于原始的环境变量来说，这是一个所有权转移。所以该闭包智能被调用一次。
* `FnMut`：描述了可以多次调用的闭包，因为它是可变借用的，所以闭包可以修改其环境变量的部分。（但是不能被多个线程同时调用）
* `Fn`：描述了一个可以重复调用的闭包，并且它仅从环境中不可变地借用值。（可以被多个线程同时调用）

编译器会自动为代码中的任何lambda表达式实现这些`Fn* trait`中的适当子集；不可能手动实现这些trait的任何一个（和C++的operator()重载不同）。

回到上面闭包的粗略的心理模型，编译器自动实现的那些trait大致对应捕获的环境上下文是否具有：

* `FnOnce`：任何可移动的值
* `FnMut`：任何可变的引用（&mut T）
* `Fn`：任何的不可变引用（&T）

上面列表中的后两个特征各自具有前面一个特征的特征界限，当你考虑使用闭包的时候，这是需要了解的。

* 如果某个闭包期望调用一次（通过`FnOnce`来接收）,那么传递一个可以重复调用很多次的`FnMut`闭包也是完全可行的。
* 如果某个闭包期望调用多次（通过`FnMut`来接收）,那么传递一个不需要改变其环境变量的`Fn`闭包也是完全可行的。

函数裸指针类型`fn`理论上也属于该列表的最后一个`Fn`，任何（非`unsafe`）`fn`类型会自动实现所有`Fn*` trait，因为它不从环境中借用任何内容，是符合所有`Fn*` trait的要求的。

因此，在编写接受闭包的代码时，请使用最通用的有效`Fn*` trait，以便于给调用者提供最大的灵活性 - 例如，接受`FnOnce`对于只调用一次的闭包。同样的建议，尽可能优选`Fn*` trait的函数bound，而不是使用裸指针函数的方式。

# 特征（Traits）

Fn* 特征比裸函数指针更灵活，但它们仍然只能描述单个函数的行为，即使这样也只能根据函数的签名来描述。

然而，trait在Rust类型系统中是描述这种行为的另一种机制。trait定义了一些公开可用的一组相关方法。特征中的每个方法也有一个名称，提供一个标签，允许编译器消除具有相同签名的方法的歧义，更重要的是，它允许程序员推断出该方法的意图。

Rust trait大致类似于 Go 和 Java 中的`接口`，或者 C++ 中的`抽象类`（所有虚拟方法，没有数据成员）。trait的实现必须提供所有方法（但请注意，trait定义可以包括默认实现，条目13），并且在这些实现的使用中可以有关联数据。这意味着代码和数据以某种面向对象的方式封装在一个公共抽象中。

接受 `struct` 并作为参数调用其方法的代码被限制为仅适用于该特定类型。如果有多种类型实现常见行为（实现共同的trait），那么定义封装该常见行为的`trait`并让代码使用该`trait`的方法而不是特定 `struct` 上的方法会更灵活。

与其他面向对象的语言出现的相同类型的建议：如果预期未来的灵活性，则更喜欢接受trait类型而不是具体类型。

有时，您想要在类型系统中区分某些行为，但无法将其表示为trait定义中的某些特定方法签名。例如，考虑对集合进行排序的trait；实现可能是稳定的（比较相同的元素将在排序前后以相同的顺序出现），但无法在 sort 方法参数中表达这一点。

在这种情况下，仍然值得使用类型系统来跟踪此需求，并使用`marker trait`。

```rust
pub trait Sort {
    /// Re-arrange contents into sorted order.
    fn sort(&mut self);
}

/// Marker trait to indicate that a [`Sortable`] sorts stably.
pub trait StableSort: Sort {}
```

`marker trait`没有方法，但实现仍然必须声明它正在实现该特征 - 这充当实现者的承诺：“我郑重宣誓，我的排序实现是稳定的”。然后，依赖于稳定排序的代码可以指定 `StableSort` trait bound，依靠系统来保留其不变性。使用marker trait来区分特征方法签名中无法表达的行为。

一旦行为作为trait被封装到 Rust 的类型系统中，就可以通过两种方式使用它：

* 作为`trait bound`，它限制了编译时通用数据类型或方法可接受的类型，或者
* 作为`trait object`。它限制了运行时可以存储或传递给方法的类型。

条目 12 更详细地讨论了`trait bound`和`trait object`之间的权衡。

`trait bound`指示由某种类型 T 参数化的通用代码只能在该类型 T 实现某些特定`trait`时使用。`trait bound`的存在意味着泛型的实现可以使用该特征中的方法，因为编译器将确保编译的任何 T 确实具有这些方法。这种检查发生在编译时，此时泛型是单态的（对于 C++ 的术语称为“模板实例化”）。

对目标类型 T 的这种限制是显式的，编码在`trait bound`中：`trait`只能由满足`trait bound`的类型实现。这与 C++ 中的等效情况形成对比，其中对 template<typename T> 中使用的类型 T 的约束是隐式的：C++ 模板代码仍然只能编译如果所有引用的方法在编译时都可用，但检查纯粹基于方法和签名。 （这种“鸭子类型”可能会导致混淆；使用 t.pop() 的 C++ 模板可能会编译为 Stack 或 Balloon – 这可能是不太理想的行为。）

对显式`trait bound`的需求也意味着很大一部分泛型使用`trait bound`。要了解其中的原因，观察并考虑在 T 上没有`trait bound`的情况下可以使用 struct Thing<T> 做什么。如果没有`trait bound`， Thing 只能执行适用于任何类型 T 的操作；这允许容器、集合和智能指针，但仅此而已。任何使用 T 类型的东西都需要一个`trait bound`。

```rust
pub fn dump_sorted<T>(mut collection: T)
where
    T: Sort + IntoIterator,
    T::Item: Debug,
{
    // Next line requires `T: Sort` trait bound.
    collection.sort();
    // Next line requires `T: IntoIterator` trait bound.
    for item in collection {
        // Next line requires `T::Item : Debug` trait bound
        println!("{:?}", item);
    }
}
```

因此，这里建议使用`trait bound`来表达对泛型中使用的类型要求，这是很容易遵循的建议 - 无论如何，编译器都会强制你遵守它。

`trait object`是利用特征定义封装的另一种方式，但这里`trait`的不同可能实现是在运行时而不是编译时选择的。这种`动态分派`类似于 C++ 中虚拟函数的使用，并且 Rust 的底层具有与 C++ 中的大致类似的`vtable`对象。

`trait object`的这种动态方面还意味着它们始终必须通过引用（ `&dyn Trait` ）或指针（ `Box<dyn Trait>` ）间接处理。这是因为实现该特征的对象的大小在编译时是未知的 - 它可能是一个巨大的 `struct` 或一个很小的 ​​ `enum` - 所以没有办法正确分配的裸特征对象的内存空间。

`trait object`的`trait`不能具有返回 `Self` 类型的方法，因为使用`trait object`的预先编译的代码不知道 `Self` 可能有多大。

可能存在所有不同类型的`T`，具有通用方法 fn method<T>(t:T) 的`trait`可能允许实现无数的方法。这对于用作`trait object`的`trait`来说很好，因为可能调用的泛型方法的无限集在编译时变成了实际调用的泛型方法的有限集。对于`trait object`来说情况并非如此：在编译时可用的代码必须应对在运行时出现所有可能的 T 类型。

这两个限制 -- 不返回 `Self` 和没有泛型方法 -- 被结合到对象安全的概念中。只有对象安全特征才能用`trait object`。