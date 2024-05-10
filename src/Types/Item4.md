# 条目 4: 优先惯用的Error类型

[条目 3](./Item3.md)描述了如何使用标准库为`Option`和`Result`类型提供的转换方法，允许使用`?`运算符对结果类型进行简洁、方便的处理。但是并没有详细讨论如何进行类型`Result<T, E>`的第二个不同的错误类型之间的相互转换；这个条目将讨论这个问题。

这个条目只有当我们具有很多个不同的错误类型时才有意义。如果函数遇到的所有不同的错误都是同一种类型，那么直接返回该类型即可。当存在不同类型的错误时，需要决定是否应该保留子错误类型的信息。

## 错误特征 （Error Trait）

了解标准的Trait（条目 10）是非常有必要的一件事情，这里面有一个`trait`就是[`std::error::Error`](https://doc.rust-lang.org/std/error/trait.Error.html)。`Result`类型的参数`E`不是必须实现`Error trait`，但是最好让你的错误类型实现`Error trait`，这样就可以允许错误类型包装器可以表达合适的`trait bound`.

首先需要注意的时`Error`类型的唯一硬性要求的`trait bound`:任何实现了`Error`类型还需要实现以下特征：

* `Display trait`: 意味着它可以使用`{}`进行`format!`编辑
* `Debug trait`: 意味着它可以使用`{:?}`进行`format!`编辑

换句话说，应该使用上述格式化来向用户和程序员显示`Error`类型

`Error trait`有唯一的方法[`source()`](https://doc.rust-lang.org/std/error/trait.Error.html#method.source)可以让`Error`类型暴露内部嵌套的内部错误。这个方法时可选的，默认的实现（条目 13）返回`None`，表示它内部的错误类型是不可用的。

最后需要注意的是：如果你的代码是工作在`no_std`环境中的，那么你无法实现`Error trait`，因为`std`库中的`Error trait`是`std`的一部分而不是`core`的一部分。

## 最小的错误 (Minimal Errors)

如果不需要嵌套的错误信息，那么实现`Error`类型可能就是一个字符串的效果 —— 那么这样直接使用`String`类型会是比较合适的方式。也不需要比`String`实现更多的能力，所以使用`String`作为`E`类型的参数。

```shell
pub fn find_user(username: &str) -> Resutl<UserId, String> {
    let f = std::fs::File::open("/etc/passwd")
        .map_err(|e| format!("Failed to open password file: {:?}", e))?;
    // ...
}
```

`String`没有实现`Error`，我们更加倾向于选择`String`让上层的代码更好的处理`Error`。也不可能对`String`类型实现`Error`，因为这两个类型不满足孤儿原则。

```shell
impl std::error::Error for String {}
```

```shell
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
  --> src/main.rs:18:5
   |
18 |     impl std::error::Error for String {}
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^------
   |     |                          |
   |     |                          `String` is not defined in the current crate
   |     impl doesn't use only types from inside the current crate
   |
   = note: define and implement a trait or new type instead
```

类型别名也不能解决这种问题，因为类型别名不会创建新类型，因为编译错误也是一样的：

```shell
pub type MyError = String;

impl std::error::Error for MyError {}
```

```shell
error[E0117]: only traits defined in the current crate can be implemented for
              types defined outside of the crate
  --> src/main.rs:41:5
   |
41 |     impl std::error::Error for MyError {}
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^-------
   |     |                          |
   |     |                          `String` is not defined in the current crate
   |     impl doesn't use only types from inside the current crate
   |
   = note: define and implement a trait or new type instead
```

像上面的报错一样，编译器错误信息给出了解决问题的提示，定义一个包装器类型 `MyError`（新类型模式，条目 6）允许该类型可以实现`Error trait`。(通常我们需要实现特定的trait的时候，但是两个类型不满足孤儿原则，则需要通过此手段进行trait的实现)

```rust
#[derive(Debug)]
pub struct MyError(String);

impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl std::error::Error for MyError {}

pub fn find_user(username: &str) -> Result<UserId, MyError> {
    let f = std::fs::File::open("/etc/passwd").map_err(|e| {
        MyError(format!("Failed to open password file: {:?}", e))
    })?;
    // ...
}
```

为了方便起见，给`MyError`类型实现`From<String>`特征是非常有意义的，这样便于我们从字符串转换成`MyError`类型（条目 5）

```rust 
impl From<String> for MyError {
    fn from(s: String) -> Self {
        Self(s)
    }
}
```

当遇到`?`运算符的时候，编译器将自动转换实现了相关`From`特征的类型到目标错误类型，这将非常的方便。

```rust
pub fn find_user(username: &str) -> Result<UserId, MyError> {
    let f = std::fs::File::open("/etc/passwd")
        .map_err(|e| format!("Failed to open password file: {:?}", e))?;
    // ...
}
```

上述的示例的错误路径如下：
* `File::open`返回一个错误类型`std::io::Error`。
* `format!`实现了`Debug`特征的`std::io::Error`类型转换为一个`String`。
* `?`运算符和使得编译器可以从`String`类型转换为实现了`From<String>`特征的`MyError`类型。

## 嵌套错误 (Nested Errors)

另一种时嵌套错误内容很重要，需要保留它并提供给调用者。

考虑一个库函数，它尝试以字符串(String)的方式返回一个文件的第一行不太长的内容，稍微思考一下，也有可能会发生如下三种类型的故障：

* 该文件可能不存在或者无法读取
* 该文件可能包含无效的UTF-8数据，因为无法转换为`String`
* 该文件的第一行可能太长

根据[条目 1](./Item1.md)，你可以使用类型系统将所有可能性表达为`enum`:

```rust
#[derive(Debug)]
pub enum MyError {
    Io(std::io::Error),
    Utf8(std::string::FromUtf8Error),
    General(String),
}
```

此 enum 定义包含 derive(Debug) ，但为了满足 Error 特征，还需要 Display 实现：

```shell
impl std::fmt::Display for MyError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            MyError::Io(e) => write!(f, "I/O error: {}", e),
            MyError::Utf8(e) => write!(f, "UTF-8 error: {}", e),
            MyError::General(e) => write!(f, "General error: {}", e),
        }
    }
}
```

覆写默认的`source()`实现也很有意义，以便轻松访问嵌套类型的错误：

```rust
use std::error::Error;

impl Error for MyError {
    fn source(&self) -> Option<&(dyn Error + 'static)> {
        match self {
            MyError::Io(e) => Some(e),
            MyError::Utf8(e) => Some(e),
            MyError::General(_) => None,
        }
    }
}
```

使用`enum`可以使错误处理变得简洁同时保留不同错误类型的信息：

```rust
use std::io::BufRead; // for `.read_until()`

/// Maximum supported line length.
const MAX_LEN: usize = 1024;

/// Return the first line of the given file.
pub fn first_line(filename: &str) -> Result<String, MyError> {
    let file = std::fs::File::open(filename).map_err(MyError::Io)?;
    let mut reader = std::io::BufReader::new(file);

    // (A real implementation could just use `reader.read_line()`)
    let mut buf = vec![];
    let len = reader.read_until(b'\n', &mut buf).map_err(MyError::Io)?;
    let result = String::from_utf8(buf).map_err(MyError::Utf8)?;
    if result.len() > MAX_LEN {
        return Err(MyError::General(format!("Line too long: {}", len)));
    }
    Ok(result)
}
```

为所有子错误类型实现`From`特征是非常有意义的，这样可以使得错误处理更加简洁：

```rust
impl From<std::io::Error> for MyError {
    fn from(e: std::io::Error) -> Self {
        Self::Io(e)
    }
}
impl From<std::string::FromUtf8Error> for MyError {
    fn from(e: std::string::FromUtf8Error) -> Self {
        Self::Utf8(e)
    }
}
```

这个可以防止库使用者受到孤儿规则的限制，不允许他们在`MyError`上实现 `From` trait。因为对于 `From` trait和`MyError`对于库使用者来说都是外部的类型。

最好的是实现`From` trait可以让这个过程更加的简洁，因为问号运算符可以自动将实现了`From` trait的错误类型转换为`MyError`类型。从而不需要`map_err`来处理错误。

```rust
use std::io::BufRead; // for `.read_until()`

/// Maximum supported line length.
pub const MAX_LEN: usize = 1024;

/// Return the first line of the given file.
pub fn first_line(filename: &str) -> Result<String, MyError> {
    let file = std::fs::File::open(filename)?; // `From<std::io::Error>`
    let mut reader = std::io::BufReader::new(file);
    let mut buf = vec![];
    let len = reader.read_until(b'\n', &mut buf)?; // `From<std::io::Error>`
    let result = String::from_utf8(buf)?; // `From<string::FromUtf8Error>`
    if result.len() > MAX_LEN {
        return Err(MyError::General(format!("Line too long: {}", len)));
    }
    Ok(result)
}
```

编写完整的错误类型可能涉及很多的样板代码，这使得通过`derive`宏来自动化是实现是非常好的实践（条目 28）。然而，没有必要自己编写这样的宏，考虑使用David Tolnay的`thiserror`库，它提供了这个类高质量以及广泛的使用的宏。`thiserror`生成的代码还小心地避免了`thiserror`类型在生成式API中可见性，反过来讲，这样将对于（条目 24）的问题会不太适用。

## trait对象 (Trait Objects)

第一种处理嵌套错误的方法丢弃了所有子错误的详细信息，只保留了一些字符串输出（即format!("{:?}", err)）。第二种方法保留了所有可能子错误的完整类型信息，但需要完全枚举所有可能的子错误类型。

这里有一个问题，是否有一种折中的方法，既能保留子错误信息，又不需要手动包括每一种可能的错误类型？

将子错误信息编码为[trait对象](https://doc.rust-lang.org/reference/types/trait-object.html)避免了为每种错误创建枚举变体的需要，抹去了特定底层错误类型的细节。接收这种对象的人将能够访问`Error trait`及其`trait bounds`的方法 -- `source()`、`Display::fmt()` 和 `Debug::fmt()`，但不需要知道子错误的原始静态类型。

```rust
#[derive(Debug)]
pub enum WrappedError {
    Wrapped(Box<dyn Error>),
    General(String),
}

impl std::fmt::Display for WrappedError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            Self::Wrapped(e) => write!(f, "Inner error: {}", e),
            Self::General(s) => write!(f, "{}", s),
        }
    }
}
```

事实证明这是可能的，但这种方式出奇的微妙。部分难度来自于`trait object`的对象安全性约束（条目 12），这里Rust的一致性规则起到了作用，这些规则大致说的是一个类型最多可以实现一次`trait`。

假设有一个错误包装类型`WrappedError`通常会实现如下两个功能：

* 实现`Error trait`, 因为它本身也是一个错误类型。
* 实现`From` trait，以便将任何实现了`Error trait`的子类型转换为`WrappedError`。

因为`WrappedError`实现了`Error trait`，这个意味着`WrappedError`类型可以通过`From`实现自我转换，这样就与`From trait`的全面自反实现冲突。

```shell
impl Error for WrappedError {}

impl<E: 'static + Error> From<E> for WrappedError {
    fn from(e: E) -> Self {
        Self::Wrapped(Box::new(e))
    }
}
```

```shell
error[E0119]: conflicting implementations of trait `From<WrappedError>` for
              type `WrappedError`
   --> src/main.rs:279:5
    |
279 |     impl<E: 'static + Error> From<E> for WrappedError {
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    |
    = note: conflicting implementation in crate `core`:
            - impl<T> From<T> for T;
```

David Tolnay 的 [anyhow](https://docs.rs/anyhow) 是一个已经解决了这些问题的 crate（通过通过 [`Box` 添加一个额外的间接层](https://github.com/dtolnay/anyhow/issues/63#issuecomment-582079114)），并且还添加了其他有用的功能（如堆栈跟踪）。因此，它正迅速成为错误处理的标准推荐——这里也再次推荐：考虑在应用程序中使用 anyhow crate 进行错误处理。

## 库与应用（Libraries Versus Applications）

前一部分的最终建议中包括了一个限定词"...用于应用程序中的错误处理"。这是因为通常区分为库重用编写的代码和构成顶层应用程序的代码之间存在明显的差异。

为库编写的代码无法预测代码使用的环境，因此最好发出具体、详细的错误信息，并让调用者自行决定如何使用这些信息。这倾向于之前描述的枚举式嵌套错误（同时也避免了在库的公共 API 中依赖 anyhow，见条目 24）。

然而，应用程序代码通常需要更多地关注如何向用户展示错误。它还可能需要处理其依赖关系图中所有库发出的所有不同错误类型（见条目 25）。因此，一个更动态的错误类型（如 [anyhow::Error](https://docs.rs/anyhow/latest/anyhow/struct.Error.html)）使得错误处理在整个应用程序中更简单、更一致。

## 牢记于心的事情（Things to Remember）

* 标准的`Error trait`对你来说不常用，建议你为你的错误类型实现`Error trait`。
* 在处理异构的底层错误类型时，需要决定是否有必要保留这些类型。
    * 如果不需要保留，则考虑适用`anyhow`去封装你应用的子错误类型。
    * 如果需要保留，则考虑适用`enum`去封装你应用的子错误类型。考虑适用`thiserror`去更好的帮助你这么做。
* 考虑适用`anyhow`库在你的应用程序代码中进行错误处理。
* 如果你想自己决定，无论怎么决定，都将错误类型编码到类型系统中([条目 1](./Item1.md))。