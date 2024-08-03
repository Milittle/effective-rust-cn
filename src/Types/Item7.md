# 条目 7: 使用Builders来构建复杂的类型

这个条目描述了构建者模式，其中复杂的数据结构有一个相关的构建者类型，使用户更容易创建数据结构的实例。

Rust 强调在创建结构体的新实例时，结构体中的所有字段都必须被填充。这通过确保没有未初始化的值来保持代码的安全性，但也导致了比理想情况更多的冗长样板代码。

例如，任何可选字段都必须明确标记为缺失，使用 None：

```rust
/// Phone number in E164 format.
#[derive(Debug, Clone)]
pub struct PhoneNumberE164(pub String);

#[derive(Debug, Default)]
pub struct Details {
    pub given_name: String,
    pub preferred_name: Option<String>,
    pub middle_name: Option<String>,
    pub family_name: String,
    pub mobile_phone: Option<PhoneNumberE164>,
}

// ...

let dizzy = Details {
    given_name: "Dizzy".to_owned(),
    preferred_name: None,
    middle_name: None,
    family_name: "Mixer".to_owned(),
    mobile_phone: None,
};
```

这种样例代码也很脆弱，因为将来如果在结构体中添加一个新字段，就需要更新每一个构建该结构体的地方。

这个样例中可以通过实现[`Default`](https://doc.rust-lang.org/std/default/trait.Default.html)特征来减少代码的冗余度，就像[条目10](../Traits/Item10.md)描述的一样。

```rust
let dizzy = Details {
    given_name: "Dizzy".to_owned(),
    family_name: "Mixer".to_owned(),
    ..Default::default()
};
```

使用 `Default` 也有助于减少在添加新字段时所需的更改，前提是新字段本身的类型实现了 `Default`。

这是一个更普遍的问题：自动派生的 `Default` 实现只有在所有字段类型都实现了 `Default` trait 时才有效。如果有一个字段不符合要求，派生步骤就不起作用：

下面的例子是无法编译的：

```rust
#[derive(Debug, Default)]
pub struct Details {
    pub given_name: String,
    pub preferred_name: Option<String>,
    pub middle_name: Option<String>,
    pub family_name: String,
    pub mobile_phone: Option<PhoneNumberE164>,
    pub date_of_birth: time::Date,
    pub last_seen: Option<time::OffsetDateTime>,
}
```

编译器错误：

```shell
error[E0277]: the trait bound `Date: Default` is not satisfied
  --> src/main.rs:48:9
   |
41 |     #[derive(Debug, Default)]
   |                     ------- in this derive macro expansion
...
48 |         pub date_of_birth: time::Date,
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ the trait `Default` is not
   |                                       implemented for `Date`
   |
   = note: this error originates in the derive macro `Default`
```

由于孤儿规则，代码不能为 `chrono::Utc` 实现 `Default`；但即便可以，这也不会有帮助——使用默认的出生日期值几乎总是错误的。

缺少 `Default` 意味着所有字段都必须手动填写：

```rust
let bob = Details {
    given_name: "Robert".to_owned(),
    preferred_name: Some("Bob".to_owned()),
    middle_name: Some("the".to_owned()),
    family_name: "Builder".to_owned(),
    mobile_phone: None,
    date_of_birth: time::Date::from_calendar_date(
        1998,
        time::Month::November,
        28,
    )
    .unwrap(),
    last_seen: None,
};
```

如果为复杂的数据结构实现构建者模式，这些易用性可以得到改善。

构建者模式的最简单变体是一个单独的结构体，它保存构建项目所需的信息。为简单起见，示例将保存项目本身的一个实例：

```rust
pub struct DetailsBuilder(Details);

impl DetailsBuilder {
    /// Start building a new [`Details`] object.
    pub fn new(
        given_name: &str,
        family_name: &str,
        date_of_birth: time::Date,
    ) -> Self {
        DetailsBuilder(Details {
            given_name: given_name.to_owned(),
            preferred_name: None,
            middle_name: None,
            family_name: family_name.to_owned(),
            mobile_phone: None,
            date_of_birth,
            last_seen: None,
        })
    }
}
```

然后，构建者类型可以配备辅助方法来填充新项目的字段。每个这样的方法都会消耗 `self`，但会生成一个新的 `Self`，从而允许不同的构建方法进行链式调用：

```rust
/// Set the preferred name.
pub fn preferred_name(mut self, preferred_name: &str) -> Self {
    self.0.preferred_name = Some(preferred_name.to_owned());
    self
}

/// Set the middle name.
pub fn middle_name(mut self, middle_name: &str) -> Self {
    self.0.middle_name = Some(middle_name.to_owned());
    self
}
```

这些辅助方法可以比简单的设值器更加有用：

```rust
/// Update the `last_seen` field to the current date/time.
pub fn just_seen(mut self) -> Self {
    self.0.last_seen = Some(time::OffsetDateTime::now_utc());
    self
}
```

构建者的最后一个方法会消耗构建者并生成构建好的项目：

```rust
/// Consume the builder object and return a fully built [`Details`]
/// object.
pub fn build(self) -> Details {
    self.0
}
```

总的来说，这让构建者的客户端拥有更符合人体工学的构建体验：

```rust
let also_bob = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
)
.middle_name("the")
.preferred_name("Bob")
.just_seen()
.build();
```

这种类型的构建者的全消耗特性导致了一些小问题。首先是无法单独分离出构建过程的各个阶段：

下面的例子是错误的：
```rust
let builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
);
if informal {
    builder.preferred_name("Bob");
}
let bob = builder.build();
```

这段代码会产生如下错误：

```rust
error[E0382]: use of moved value: `builder`
   --> src/main.rs:256:15
    |
247 |     let builder = DetailsBuilder::new(
    |         ------- move occurs because `builder` has type `DetailsBuilder`,
    |                 which does not implement the `Copy` trait
...
254 |         builder.preferred_name("Bob");
    |                 --------------------- `builder` moved due to this method
    |                                       call
255 |     }
256 |     let bob = builder.build();
    |               ^^^^^^^ value used here after move
    |
note: `DetailsBuilder::preferred_name` takes ownership of the receiver `self`,
      which moves `builder`
   --> src/main.rs:60:35
    |
27  |     pub fn preferred_name(mut self, preferred_name: &str) -> Self {
    |                               ^^^^
```

可以通过将被消耗的构建者重新分配给同一个变量来解决这个问题：

```rust
let mut builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
);
if informal {
    builder = builder.preferred_name("Bob");
}
let bob = builder.build();
```

这种构建者的全消耗特性的另一个缺点是只能构建一个项目；尝试通过在同一个构建者上重复调用 `build()` 来创建多个实例，会如预期那样导致编译器报错：

下面的代码是有编译错误的：

```rust
let smithy = DetailsBuilder::new(
    "Agent",
    "Smith",
    time::Date::from_calendar_date(1999, time::Month::June, 11).unwrap(),
);
let clones = vec![smithy.build(), smithy.build(), smithy.build()];
```

```shell
error[E0382]: use of moved value: `smithy`
   --> src/main.rs:159:39
    |
154 |   let smithy = DetailsBuilder::new(
    |       ------ move occurs because `smithy` has type `base::DetailsBuilder`,
    |              which does not implement the `Copy` trait
...
159 |   let clones = vec![smithy.build(), smithy.build(), smithy.build()];
    |                            -------  ^^^^^^ value used here after move
    |                            |
    |                            `smithy` moved due to this method call
```

一种可选的方法是让构造器的方法接受`&mut self`并输出一个`&mut Self`:

```rust
/// Update the `last_seen` field to the current date/time.
pub fn just_seen(&mut self) -> &mut Self {
    self.0.last_seen = Some(time::OffsetDateTime::now_utc());
    self
}
```

这样就不需要在各个构建阶段进行自我赋值了：

```rust
let mut builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
);
if informal {
    builder.preferred_name("Bob"); // no `builder = ...`
}
let bob = builder.build();
```

然而，这个版本使得无法将构建者的构建与调用其设值方法链式结合在一起：

下面的代码是错误的：

```rust
let builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
)
.middle_name("the")
.just_seen();
let bob = builder.build();
```

这段代码会产生如下错误：

```shell
error[E0716]: temporary value dropped while borrowed
   --> src/main.rs:265:19
    |
265 |       let builder = DetailsBuilder::new(
    |  ___________________^
266 | |         "Robert",
267 | |         "Builder",
268 | |         time::Date::from_calendar_date(1998, time::Month::November, 28)
269 | |             .unwrap(),
270 | |     )
    | |_____^ creates a temporary value which is freed while still in use
271 |       .middle_name("the")
272 |       .just_seen();
    |                   - temporary value is freed at the end of this statement
273 |       let bob = builder.build();
    |                 --------------- borrow later used here
    |
    = note: consider using a `let` binding to create a longer lived value
```

如编译器错误所示，可以通过给构建者项目命名来解决这个问题：

```rust
let mut builder = DetailsBuilder::new(
    "Robert",
    "Builder",
    time::Date::from_calendar_date(1998, time::Month::November, 28)
        .unwrap(),
);
builder.middle_name("the").just_seen(); // 注意这行，是单独分离开来解决上面的编译错误的
if informal {
    builder.preferred_name("Bob");
}
let bob = builder.build();
```

这种可变的构建者变体还允许构建多个项目。`build()` 方法的签名必须不消耗 `self`，因此必须如下所示：

```rust
/// Construct a fully built [`Details`] object.
pub fn build(&self) -> Details {
    // ...
}
```

这个可重复的 `build()` 方法的实现必须在每次调用时构建一个新的项目。如果底层项目实现了 `Clone`，这很容易——构建者可以持有一个模板，并在每次构建时 `clone()` 它。如果底层项目没有实现 `Clone`，那么构建者需要有足够的状态，以便在每次调用 `build()` 时手动构建一个底层项目的实例。

无论使用哪种构建者模式，样例代码现在都集中在一个地方——构建者——而不是在每个使用底层类型的地方都需要。

剩余的样板代码可以通过使用宏（项目28）进一步减少，但如果你选择这种方法，你还应该检查是否有现成的 crate（特别是 `derive_builder` crate）可以提供所需的功能——前提是你愿意依赖它（项目25）。

译者注：构造器模式是一个非常常用的设计模式，在Rust中可以让一个对象的构建变得收敛且简单，也不用在对象内部字段变化的时候散弹式进行修改。