---
title: Rust宏
date: 2022-04-24 23:43:53
tags: Rust
cover: images/Rust.png
top_img: images/Rocket.png
categories: Rust
---

# 前言

宏是Rust比较重要而且强大的特性之一。宏可以减少重复代码，自动生成一些代码，让代码看起来更优雅。例如[Rocket web](https://rocket.rs/)框架的宏:
```rust
#[macro_use] extern crate rocket;

#[get("/")]
fn index() -> &'static str {
    "Hello, world!"
}

#[launch]
fn rocket() -> _ {
    rocket::build().mount("/", routes![index])
}
```
熟悉Python Flask框架的同学肯定会直呼: 这个不就是Rust版本的Flask嘛。对，这个就是Rust宏的强大之处，通过宏让代码写起来特别简单优雅。

那么了解Rust宏是很有必要的，一方面能让我们的代码更加简洁，另一方面在阅读和学习开源代码的时候也能更加得心应手（很多开源代码都使用了大量的宏）。所以本文就是和大家一起去探索Rust宏，目的是让读者能够写出自己需要的宏。本文首先会讲解一些关于Rust宏一些基础概念和知识，并对相关的知识点给出示例代码进行分析。

# 宏

提到宏很多同学应该都会想到C/C++的宏。在C/C++中宏主要是文本替换，所以如果要实现一个multiply(x, y)宏需要这样实现:
```C++
// demo mutliply(2 + 3, 4 + 5)
#define multiply(x, y) x * y // 错误，宏展开: 2 + 3 * 4 + 5，结果19
#define multiply(x, y) ((x) * (y)) // 正确，红展开: ((2 + 3) * (4 + 5))，结果45
```
我们来看看Rust版本的宏
```rust
macro_rules! multiply {
    ($x:expr, $y:expr) => {
        $x * $y
    };
}

fn main() {
    let a = multiply!(2 + 3, 4 + 5);
}
```

通过```cargo expand```可以查看宏展开之后的代码

```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
fn main() {
    let a = (2 + 3) * (4 + 5);
}
```
如果不了解Rust的声明宏也没有关系，我们先来直观的看看Rust和C/C++宏的区别。比较大的区别是Rust宏并没有像C/C++那样使用很多括号来保护，可以看出Rust宏并不是简单的文本替换。其实Rust宏是有专门的宏解析器，是在语法解析层面进行的宏展开。

Rust宏可以分为两大类:

- 声明宏（Declarative Macro）
- 过程宏（Procedural Macro）

声明宏是指通过```macro_rules!```声明定义的宏，它是Rust中比较常见的宏，如上述的```multiply```宏。这种宏类似C/C++的宏，主要做替换展开，但是比C/C++的文本替换方式要强大并且安全。该类宏的调用方式和函数调用类似，只是名字后面有感叹号(!)```宏名字!```，如```println!```、```assert_eq!```、```multiply!```等。

过程宏是编译器语法扩展的方式之一。Rust允许通过特定的语法编写编译插件，但是该编写的插件语法还不稳定，所以提供了过程宏来让开发者实现自定义派生属性的功能。比如Serde库中实现的```#[derive(Serialize, Deserialize)]```就是基于过程宏实现的。———— 《Rust编程之道》

# 声明宏

声明宏定义格式如下:

```rust
macro_rules! $name {
    $pattern0 => ($expansion);
    $pattern1 => ($expansion);
}
```

其中```$name```表示宏的名字，内部一般由1个或者多个模式匹配组成。匹配上规则之后就用(```$expansion```)代替。

举个栗子(例子来源《Rust编程之道》):

```rust
macro_rules! unless {
    ($arg: expr, $branch: expr) => (if !$arg { $branch; };);
}

fn main() {
    let (a, b) = (1, 2);
    unless!(a > b, {
        b - a
    });
}
```

上述```unless```宏的匹配模式是```($arg: expr, $branch: expr)```，表示匹配两个表达式参数，参数之间的分隔符是逗号(,)。其中```$arg```和```$branch```为捕获变量，可以自由命名，但是必须以```$```开头。冒号(:)后面的是捕获类型，expr表示表达式。

用```cargo expand```看看宏展开之后的代码:
```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
fn main() {
    let (a, b) = (1, 2);
    if !(a > b) {
        {
            b - a
        };
    };
}
```

关于声明宏中可以捕获的类型：——《Rust编程之道》

- item: 代表语言项，就是组成一个Rust包的基本单位，比如模块、声明、函数定义、结构体定义、impl实现等。
- block: 代表代码块，由花括号限定的代码。
- stmt: 代码语句，一般是指以分号结尾的代码。
- expr: 表达式，会生成具体的值
- pat: 模式。
- ty: 类型。
- ident: 标识。
- path: 路径，比如foo、std::iter等
- meta: 元信息，表示包含在#[]或者#![...]属性内的信息
- tt: TokenTree的缩写，词条树
- vis: 指代可见性，比如pub
- lifetime: 生命周期参数

## 匹配不定长参数

Rust自带的宏```vec![]```就是一个不定长参数宏，我们先看看官方是怎么实现的:
```rust
macro_rules! __rust_force_expr {
    ($e:expr) => {
        $e
    };
}

macro_rules! vec {
    () => (
        $crate::__rust_force_expr!($crate::vec::Vec::new())
    );
    ($elem:expr; $n:expr) => (
        $crate::__rust_force_expr!($crate::vec::from_elem($elem, $n))
    );
    ($($x:expr),+ $(,)?) => (
        $crate::__rust_force_expr!(<[_]>::into_vec(box [$($x),+]))
    );
}
```
我们再来看看如何使用这个宏,
```rust
fn main() {
    let a:Vec<i32> = vec![]; // 空数组
    let b = vec![1; 10]; // [1, 1, 1, 1, 1, 1, 1, 1, 1, 1] 总共10个元素
    let c = vec![0, 1, 2, 3, 4, 5]; // [0, 1, 2, 3, 4, 5]
}
```
查看Rust```vec!```源码，我们可以发现该宏有三个匹配模式:

1. 没有任何参数，返回一个空数组
2. 有两个参数，但是分隔符是分号(;)，例如```vec![1; 10]```，调用```vec::from_elem```
3. 1个或者多个参数，分隔符为逗号(,)，例如```vec![0, 1, 2, 3]```，表示用这些元素初始化数组

我们重点看一下宏的不定长参数是如何实现的，声明宏重复匹配的格式是```$(...) sep rep```，具体说明如下: —— 《Rust编程之道》

- ```$(...)```: 代码要把重复匹配的模式置于其中。
- ```step```: 代表分隔符，常用逗号(,)、分号(,)、火箭符(=>)。这个分隔符可依据具体的情况省略。
- ```rep```: 代表控制重复次数的标记，目前支持两种: 星号(*)和加号(+)，代表的意义和正则表达式中的一致，分别是“重复零次及以上”和“重复一次及以上”。

## hashmap

了解声明宏的知识之后，我们来写一个hashmap的宏（该宏参考《Rust编程之道》）。```hashmap!```和```vec!```（+!突出是宏）类似用于初始化。使用方式如下:

```rust
fn main() {
    let m = hashmap!{
        "a" => 1,
        "b" => 2,
    };
    assert_eq!(m.get("a"), Some(&1));
    assert_eq!(m.get("b"), Some(&2));
    assert_eq!(m.len(), 2);
}
```
这个宏有几个特点:

1. 参数不固定
2. 参数形式为: $key => $value

我们可以模仿```vec!```宏进行实现:
```rust
macro_rules! hashmap {
    // 和vec!一样，没有任何参数则创建一个空的hashmap
    () => {
        {::std::collections::HashMap::new()}
    };
    // 这里表示匹配多个 $key => $value参数，分隔符是逗号(,)
    // 最后$(,)? 表示最后一个逗号(,)可有可无
    ($($key:expr => $value: expr),+$(,)?) => {
        { // 这里一定要有大括号包裹，因为这里有多条语句。使用大括号，产生一个块表达式。宏展开之后就看的比较清晰了
            let mut _m = ::std::collections::HashMap::new();
            $(
                _m.insert($key, $value);
            )*
            _m
        }
    }
}

fn main() {
    let m = hashmap! {
        "a" => 1,
        "b" => 2, // $(,)? 匹配这个逗号，如果没有这个匹配，这里会出错的
    };
}
```
通过上述宏实现可以发现

1. 匹配不定长多参的时候采用```*```或```+```
2. 生成代码的时候，针对多参数也是通过```*```或```+```进行展开。如```$(_m.insert($key, $value))*```，表示针对每个参数都执行这样的操作。
3. 宏内部实现需要有大括号包裹，创建一个块表达式，即这个块具有返回值。

```rust
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2021::*;
#[macro_use]
extern crate std;
fn main() {
    let m = { // 可以看到这个大括号的作用，就是创建一个代码块表达式，并有返回hashmap对象。
        let mut _m = ::std::collections::HashMap::new();
        _m.insert("a", 1);
        _m.insert("b", 2);
        _m
    };
```
关于这个例子更多更详细的资料请参考《Rust编程之道》。

# 过程宏

目前，使用过程宏可以实现三种类型的宏: ————《Rust编程之道》

- 自定义派生属性，可以自定义类似于```#[derive(Debug)]```这样的derive属性，可以自动为结构体或枚举类型进行语法扩展。
- 自定义属性，可以自定义类似于```#[Debug]```这种属性。
- Bang宏，和```macro_rules!```定义的宏类似，以Bang符号（就是叹号"!"）结尾的宏。

过程宏的特点就是基于```TokenStream```来分析原代码（结构体或者枚举等其他原代码），然后产生新的代码，还是以```TokenStream```返回给编译器。一般函数定义如下:
```rust
pub fn derive(input: TokenStream) -> TokenStream;
```
根据宏的类型不同，参数数量有所不同。

另外创建过程宏需要在```Cargo.toml```里面设置:
```rust
[lib]
proc-macro = true
```

下面来看一个简单的自定义派生属性宏的例子，目标是结构体A实现一个```hello```方法，并返回```hello from A```;
```rust
#[proc_macro_derive(Hello)]
pub fn derive(input: TokenStream) -> TokenStream {
    r#"
        impl A {
            pub fn hello(&self) -> String {
                "hello from A".to_string()
            }
        }
    "#.parse().unwrap()
}
```
上述代码就是实现自定义派生宏```Hello```，其中有几个重要的信息:

1. ```#[proc_macro_derive(Hello)]```属性表示其下方的函数专门处理自定义派生属性，其中```Hello```与```#[derive(Hello)]```中的```Hello```相对应，及派生属性名。
2. ```r#"..."#```表示可以写多行字符串
3. 这里为了方便演示没有对原始的```input: TokenStream```做任何解析和判断，直接返回了写死的代码。
4. 可以把字符串解析转换成```TokenStream```，这里生成的代码就是为A类型实现```hello```方法。

下面我们看看如何使用这个自定义派生宏（用过程宏实现的）以及宏展开之后的代码:
```rust
#[derive(Hello)]
struct A {
}
```
宏展开之后的代码:
```rust
struct A {}
impl A {
    pub fn hello(&self) -> String {
        "hello from A".to_string()
    }
}
```

## TokenStream

这里稍微解释一下什么是```TokenStream```，一般编译器在编译源代码的时候，其中比较重要的一个环节就是源代码经过词法分析器产生词法单元的序列，Rust这里就是```TokenStream```。
比如，假设一个源代码包含如下的赋值语句: ———— 例子来源《编译原理》
```c++
position = initial + rate * 60
```
经过词法分析之后，复制语句被表示成如下的词法单元序列:
```
<id,1><=><id,2><+><id,3><*><60>
```

1. ```<>```表示一个Token，例如```<id,1>```，其中```id```是表示标识符(identifier)的抽象符号，而1指向符号表中```position```对应的条目。
2. 赋值符号```=```是一个词素，被映射成词法单元```<=>```，同理```+```被映射成```<+>```。

其中空格或者注释等一些信息都被忽略了，将代码拆分成一个一个的Token，Token的类型可以使用一个变量，一个操作符，一个立即数等。

## syn, quote

要写出功能比较强大的宏，肯定离不开对```input: TokenStream```的解析。无论是直接通过```TokenStream```方式还是将其转换成字符串之后进行解析，都是比较困难的。如果是转成字符串再解析里面的内容，可能会有大量的字符串的匹配和正则表代码。一方面代码写起来很不方便，另一方面代码也不好维护。好在目前在实现过程宏的时候有两个比较强大的第三方库可以帮我解决大部分解析问题。

- syn: 将```TokenStream```解析成语法树结构。
- quote: 将syn的语法树结构转为```TokenStream```类型。

之前的Hello自定义派生属性宏，局限性比较大，只能给结构体名为"A"的结构体实现```hello```方法，这里我们通过```syn```和```quote```工具来增强实现一下:
```rust
#[proc_macro_derive(Hello)]
pub fn derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as syn::DeriveInput); // 将TokenStream解析成syn语法树
    let ident = input.ident; // 获取结构体标识，如果属性是作用于struct B，则ident就为B
    let name = ident.to_string(); // 将标识符转成字符串用于hello方法里面的字符串拼接
    quote! { // quote!宏将syn转成TokenStream
        impl #ident { // 用#{}引用syn类型或者rust变量
            pub fn hello(&self) -> String {
                format!("hello from {}", #name)
            }
        }
    }.into()
}
```

- ```parse_macro_input!```宏将```input```解析为```syn::DeriveInput```类型的抽象语法树结构
- ```input.ident```就是从```syn```语法树里面直接获取到结构体的标识，无需我们额外解析
- ```quote!```和```macro_rules!```用法差不多，不同点在于，```quote!```宏使用符号'#'

同样再来看看使用宏的代码以及宏展开之后的代码
```rust
#[derive(Hello)]
struct A {
}

#[derive(Hello)]
struct B {
}
```
宏展开之后的代码:
```rust
struct A {}
impl A {
    pub fn hello(&self) -> String {
        {
            let res = ::alloc::fmt::format(::core::fmt::Arguments::new_v1(
                &["hello from "],
                &[::core::fmt::ArgumentV1::new_display(&"A")],
            ));
            res
        }
    }
}
struct B {}
impl B {
    pub fn hello(&self) -> String {
        {
            let res = ::alloc::fmt::format(::core::fmt::Arguments::new_v1(
                &["hello from "],
                &[::core::fmt::ArgumentV1::new_display(&"B")],
            ));
            res
        }
    }
}
```

## heapsize

学习完过程宏的基础知识我来看看一个稍微正式的例子[heapsize](https://github.com/dtolnay/syn/tree/master/examples/heapsize)，这个例子是syn官方提供的example，也是比较有学习价值的。也可以先看看官方教程，再回来看看本文。

先来说说heapsize实现的目标:
首先定义一个```HeapSize```trait，这个trait有一个方法```fn heap_size_of_children(&self) -> usize```并返回结构体的heapsize（结构体的堆大小）。
```rust
pub trait HeapSize {
    /// Total number of bytes of heap memory owned by `self`.
    fn heap_size_of_children(&self) -> usize;
}
```
同时```HeapSize```宏可以帮结构体自动实现这个trait:
```rust
#[derive(HeapSize)]
struct Demo<'a, T: ?Sized> {
    a: Box<T>,
    b: u8,
    c: &'a str,
    d: String,
}
```
自动生成的代码如下:
```rust
impl<'a, T: ?Sized + heapsize::HeapSize> heapsize::HeapSize for Demo<'a, T> {
    fn heap_size_of_children(&self) -> usize {
        0 + heapsize::HeapSize::heap_size_of_children(&self.a)
            + heapsize::HeapSize::heap_size_of_children(&self.b)
            + heapsize::HeapSize::heap_size_of_children(&self.c)
            + heapsize::HeapSize::heap_size_of_children(&self.d)
    }
}
```
下面来一起分析如何实现这个heapsize。

1. [```HeapSize```](https://github.com/dtolnay/syn/blob/master/examples/heapsize/heapsize/src/lib.rs)trait

```rust
use std::mem;

pub use heapsize_derive::*;

pub trait HeapSize {
    /// Total number of bytes of heap memory owned by `self`.
    ///
    /// Does not include the size of `self` itself, which may or may not be on
    /// the heap. Includes only children of `self`, meaning things pointed to by
    /// `self`.
    fn heap_size_of_children(&self) -> usize;
}

//
// In a real version of this library there would be lots more impls here, but
// here are some interesting ones.
//

impl HeapSize for u8 {
    /// A `u8` does not own any heap memory.
    fn heap_size_of_children(&self) -> usize {
        0
    }
}

impl HeapSize for String {
    /// A `String` owns enough heap memory to hold its reserved capacity.
    fn heap_size_of_children(&self) -> usize {
        self.capacity()
    }
}

impl<T> HeapSize for Box<T>
where
    T: ?Sized + HeapSize,
{
    /// A `Box` owns however much heap memory was allocated to hold the value of
    /// type `T` that we placed on the heap, plus transitively however much `T`
    /// itself owns.
    fn heap_size_of_children(&self) -> usize {
        mem::size_of_val(&**self) + (**self).heap_size_of_children()
    }
}

impl<T> HeapSize for [T]
where
    T: HeapSize,
{
    /// Sum of heap memory owned by each element of a dynamically sized slice of
    /// `T`.
    fn heap_size_of_children(&self) -> usize {
        self.iter().map(HeapSize::heap_size_of_children).sum()
    }
}

impl<'a, T> HeapSize for &'a T
where
    T: ?Sized,
{
    /// A shared reference does not own heap memory.
    fn heap_size_of_children(&self) -> usize {
        0
    }
}
```
上述代码是syn官方demo的源代码，主要是定义了```HeapSize```trait，然后为一些基础类型实现默认的trait实现。例如```u8```的堆大小为0，```String```的堆大小为字符串的长度等等。

2. [```HeapSize!```](https://github.com/dtolnay/syn/blob/master/examples/heapsize/heapsize_derive/src/lib.rs)宏的实现

这里我们暂时不给出最终代码，而是一步一步的去实现这个自定义派生属性宏。

2.1 函数的声明并搭好架子（可以说这个是写派生属性宏的一般套路）
```rust
#[proc_macro_derive(HeapSize)]
pub fn derive_heap_size(input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    // Parse the input tokens into a syntax tree.
    let input = parse_macro_input!(input as DeriveInput);
    // ... 
    quote! {
    }.into()
}
```
这个是写派生属性宏的一般套路，就是把```TokenStream```转成```syn```的语法树，最终通过```quote!```把```syn```语法树转成```TokenStream```。

2.2 生成```HeapSize```trait实现定义

```rust
#[proc_macro_derive(HeapSize)]
pub fn derive_heap_size(input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    // Parse the input tokens into a syntax tree.
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;
    // ... 
    quote! {
        impl heapsize::HeapSize for #name {
            fn heap_size_of_children(&self) -> usize {
                0
            }
        }
    }.into()
}
```

根据之前```Hello```宏的套路，我们很快就能写出```HeapSize```的实现（这里临时写死返回值是0）。从```input```(```syn```的语法树)提取```ident```，这样```impl heapsize::HeapSize for #name```就可以为任意结构实现这个trait了。

但是某些情况下，上述代码是有问题的。例如泛型结构体等，如下结构体就是含有声明周期标注```'a```和模板参数```T: ?Sized```。

```rust
struct Demo<'a, T: ?Sized> {
    a: Box<T>,
    b: u8,
    c: &'a str,
    d: String,
}
```

这种情况我们上述的```impl heapsize::HeapSize for #name```实现就有问题了，因为正确的实现是```impl<'a, T: ?Sized> heapsize::HeapSize for #name```。这里就有一个问题如何提取这些泛型参数呢？

```rust
#[proc_macro_derive(HeapSize)]
pub fn derive_heap_size(input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    // Parse the input tokens into a syntax tree.
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;
    // 将结构的泛型参数split成三个部分，impl的泛型，类型的泛型，where从句
    let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();
    // ... 
    quote! {
        impl #impl_generics  heapsize::HeapSize for #name #ty_generics #where_clause {
            fn heap_size_of_children(&self) -> usize {
                0
            }
        }
    }.into()
}
```

其中```input.generics.split_for_impl()```也是基本套路用来处理含有泛型参数的结构体。例如上述的```struct Demo<'a, T:?Sized>```:

- ```impl_generics```: ```<'a, T: ?Sized>```
- ```ty_generics```: ```<'a, T>```
- ```where_clause```为空

2.3 为泛型参数增加trait限定，例如```struct Demo<'a, T: ?Sized>```需要对泛型参数```T```限定为: ```T: ?Sized + heapsize::HeapSize```，这样我们才能调用成员变量的```heap_size_of_children```函数，期待生成代码如下（还是```struct Demo<'a, T: Sized>```）
```rust
impl <'a, T: ?Sized + heapsize::HeapSize> heapsize::HeapSize for Demo<'a, T> {
    fn heap_size_of_children(&self) -> usize {
        ...
    }
}
```

添加泛型约束如下:

```rust
fn add_trait_bounds(mut generics: Generics) -> Generics {
    for param in &mut generics.params {
        if let GenericParam::Type(ref mut type_param) = *param {
            type_param.bounds.push(parse_quote!(heapsize::HeapSize));
        }
    }
    generics
}

#[proc_macro_derive(HeapSize)]
pub fn derive_heap_size(input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    // Parse the input tokens into a syntax tree.
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;
    // Add a bound `T: HeapSize` to every type parameter T.
    let generics = add_trait_bounds(input.generics);
    let (impl_generics, ty_generics, where_clause) = generics.split_for_impl();
    // 将结构的泛型参数split成三个部分，impl的泛型，类型的泛型，where从句
    let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();
    // ... 
    quote! {
        impl #impl_generics  heapsize::HeapSize for #name #ty_generics #where_clause {
            fn heap_size_of_children(&self) -> usize {
                0
            }
        }
    }.into()
}
```

这里稍微拓展一下，我们来看看```Generics```相关类型的定义:

```rust
pub struct DeriveInput { // input的类型
    /// Attributes tagged on the whole struct or enum.
    pub attrs: Vec<Attribute>,

    /// Visibility of the struct or enum.
    pub vis: Visibility,

    /// Name of the struct or enum.
    pub ident: Ident,

    /// Generics required to complete the definition.
    pub generics: Generics,

    /// Data within the struct or enum.
    pub data: Data,
}

pub struct Generics {
    pub lt_token: Option<Token![<]>,
    pub params: Punctuated<GenericParam, Token![,]>,
    pub gt_token: Option<Token![>]>,
    pub where_clause: Option<WhereClause>,
}

pub enum GenericParam {
    /// A generic type parameter: `T: Into<String>`.
    Type(TypeParam),

    /// A lifetime definition: `'a: 'b + 'c + 'd`.
    Lifetime(LifetimeDef),

    /// A const generic parameter: `const LENGTH: usize`.
    Const(ConstParam),
}

pub struct TypeParam {
    pub attrs: Vec<Attribute>,
    pub ident: Ident,
    pub colon_token: Option<Token![:]>,
    pub bounds: Punctuated<TypeParamBound, Token![+]>,
    pub eq_token: Option<Token![=]>,
    pub default: Option<Type>,
}
```
其中```DeriveInput```各个字段的含义如下: ————参考《Rust编程之道》

- attrs, 实际为```Vec<syn::Attribute>```类型，```syn::Attribute```代表属性，比如```#[repr(C)]```，使用```Vec<T>```代表可以定义多个属性。用于存储作用语结构体或枚举类型的属性。
- vis, 为```syn::Visibility```类型，代表结构体或枚举体的可见性。
- ident, 为```syn::Ident```，将会存储结构体或枚举体的名称。
- generics, 为```syn::Generics```，用于存储泛型信息。
- data, 为```syn::Data```，包括结构体、枚举体和联合体这三种类型。

其中```Generics```类型的成员```params```是```Punctuated<GenericParam, Token![,]>```类型，而```Punctuated<T, P>```类型在```syn```库中非常常见。我们来解释一下这个类型的含义：用分割符```P```分割出来的类型序列```T```。可以把```Punctuated<T, P>```当成```Vec<T>```。因为解析是```syn```工具做的事情，我们不太关心他是通过逗号分割得到的，还是通过+分割得来的。但是我们了解Rust语法肯定就知道，有些类型他是通过什么分隔符得来的（纯属个人看法）。比如FieldsNamed类型:

```rust
pub struct FieldsNamed {
    pub brace_token: token::Brace,
    pub named: Punctuated<Field, Token![,]>, // 结构体的field是通过逗号分割的(,)，这里他不可能写成其他分隔符
}
```

这些类型都是```syn```已经定义好了，我们使用就行了，不用太关心分隔符到底是啥，直接当成```Vec<T>```来使用。

2.4 实现```HeapSize```具体的业务逻辑
```rust
// Generate an expression to sum up the heap size of each field.
fn heap_size_sum(data: &Data) -> TokenStream {
    match *data {
        Data::Struct(ref data) => {
            match data.fields {
                Fields::Named(ref fields) => {
                    // Expands to an expression like
                    //
                    //     0 + self.x.heap_size() + self.y.heap_size() + self.z.heap_size()
                    //
                    // but using fully qualified function call syntax.
                    //
                    // We take some care to use the span of each `syn::Field` as
                    // the span of the corresponding `heap_size_of_children`
                    // call. This way if one of the field types does not
                    // implement `HeapSize` then the compiler's error message
                    // underlines which field it is. An example is shown in the
                    // readme of the parent directory.
                    let recurse = fields.named.iter().map(|f| {
                        let name = &f.ident;
                        quote_spanned! {f.span()=>
                            heapsize::HeapSize::heap_size_of_children(&self.#name)
                        }
                    });
                    quote! {
                        0 #(+ #recurse)*
                    }
                }
                Fields::Unnamed(ref fields) => {
                    // Expands to an expression like
                    //
                    //     0 + self.0.heap_size() + self.1.heap_size() + self.2.heap_size()
                    let recurse = fields.unnamed.iter().enumerate().map(|(i, f)| {
                        let index = Index::from(i);
                        quote_spanned! {f.span()=>
                            heapsize::HeapSize::heap_size_of_children(&self.#index)
                        }
                    });
                    quote! {
                        0 #(+ #recurse)*
                    }
                }
                Fields::Unit => {
                    // Unit structs cannot own more than 0 bytes of heap memory.
                    quote!(0)
                }
            }
        }
        Data::Enum(_) | Data::Union(_) => unimplemented!(),
    }
}

fn add_trait_bounds(mut generics: Generics) -> Generics {
    for param in &mut generics.params {
        if let GenericParam::Type(ref mut type_param) = *param {
            type_param.bounds.push(parse_quote!(heapsize::HeapSize));
        }
    }
    generics
}

#[proc_macro_derive(HeapSize)]
pub fn derive_heap_size(input: proc_macro::TokenStream) -> proc_macro::TokenStream {
    // Parse the input tokens into a syntax tree.
    let input = parse_macro_input!(input as DeriveInput);
    let name = input.ident;
    // Add a bound `T: HeapSize` to every type parameter T.
    let generics = add_trait_bounds(input.generics);
    let (impl_generics, ty_generics, where_clause) = generics.split_for_impl();
    // 将结构的泛型参数split成三个部分，impl的泛型，类型的泛型，where从句
    let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();

    let sum = heap_size_sum(&input.data); 

    quote! {
        impl #impl_generics  heapsize::HeapSize for #name #ty_generics #where_clause {
            fn heap_size_of_children(&self) -> usize {
                #sum
            }
        }
    }.into()
}
```

增加了一个```heap_size_sum```用于计算结构体成员变量的heapsize之和。这里重点是对```input.data: syn::Data```数据进行处理，我们先来看看```syn```相关的结构体:

```rust
pub enum Data {
    /// A struct input to a `proc_macro_derive` macro.
    Struct(DataStruct),

    /// An enum input to a `proc_macro_derive` macro.
    Enum(DataEnum),

    /// An untagged union input to a `proc_macro_derive` macro.
    Union(DataUnion),
}

pub struct DataStruct {
    pub struct_token: Token![struct],
    pub fields: Fields,
    pub semi_token: Option<Token![;]>,
}

pub enum Fields {
    /// Named fields of a struct or struct variant such as `Point { x: f64,
    /// y: f64 }`.
    Named(FieldsNamed),

    /// Unnamed fields of a tuple struct or tuple variant such as `Some(T)`.
    Unnamed(FieldsUnnamed),

    /// Unit struct or unit variant such as `None`.
    Unit,
}

pub struct FieldsNamed {
    pub brace_token: token::Brace,
    pub named: Punctuated<Field, Token![,]>,
}

pub struct Field {
    /// Attributes tagged on the field.
    pub attrs: Vec<Attribute>,

    /// Visibility of the field.
    pub vis: Visibility,

    /// Name of the field, if any.
    ///
    /// Fields of tuple structs have no names.
    pub ident: Option<Ident>,

    pub colon_token: Option<Token![:]>,

    /// Type of the field.
    pub ty: Type,
}
```

从上面的相关结构体定义可以看出：

- ```syn::Data```是一个枚举类型，有三种枚举类型```Struct```，```Enum```，```Union```，分别代表结构体，枚举体，联合体。
- ```DataStruct```表示结构体，其中```fields```字段存储结构字段的信息。
- ```Fields```表示结构体的字段信息，是一个枚举类型，有两种枚举类型```Named```和```Unnamed```，分别代表了命名结构体和匿名结构体。
- ```FieldsNamed```表示命名结构体，里面named字段就是包含各个字段信息的```Punctuated<Field, Token![,]>```类型，可以当成```Vec<Field>```。
- ```Field```表示字段的具体信息了，其中```ident```表示字段的名字，```ty```表示字段的类型等。

了解这些结构体的含义之后，```heap_size_sum```这个函数就比较好理解了。我们把匹配的代码去掉，看看核心的代码。

```rust
let recurse = fields.named.iter().map(|f| {
    // f就是Field类型
    let name = &f.ident; // 获取成员变量的名字
    quote_spanned! {f.span()=> // f.span() 是成员变量原代码的Trace信息，比如这个成员变量原始的代码位置
        heapsize::HeapSize::heap_size_of_children(&self.#name) // 调用成员变量HeapSize trait的方法
    }
});

quote! {
    0 #(+ #recurse)*
}
```

其中```fields.named```就可以认为是字段信息```Field```数组了，然后针对每一个成员变量调用```HeapSize```方法。

这里有几个需要主意的地方:

1. ```f.span()```返回一个```Span```对象，这个对象主要是定位原始代码信息，比如原始字段在代码的位置，几行几列。这样做的原因是，出错了方便定位原始代码。比如某个字段没有实现```HeapSize```trait，如果没有Span，可能报错的位置用户肯定看不懂，因为这块代码是动态生成的，没有行号和列号。加了```Span```之后，报错就报错在这个字段这里，并报告是因为没有实现```HeapSize```trait。一般配合```quote_spanned!```使用。
```bash
error[E0277]: the trait bound `std::thread::Thread: HeapSize` is not satisfied
 --> src/main.rs:7:5
  |
7 |     bad: std::thread::Thread,
  |     ^^^ the trait `HeapSize` is not implemented for `std::thread::Thread`
```

2. ```quote!```和```macro_rules!```类似，不过是'#'符号。```#(...)*```表示重复。

## [derive-new](https://github.com/nrc/derive-new)

通过上面的学习，如果觉得已经掌握了派生属性宏的知识，可以试着实现[```derive-new```](https://github.com/nrc/derive-new)。```derive-new```是一个开源的代码库，用于给结构体等数据结构自动实现```pub fn new(args...) -> Self```方法。

可以尝试自己实现这个宏，再看看源代码。如果觉得看源代码有点困难，可以再回来看看这个章节。

```rust
use proc_macro::TokenStream;
use quote::{quote, quote_spanned};
use syn::parse_macro_input;
use syn::parse_quote;
use syn::{Generics, GenericParam};

struct FieldExt<'a> {
    ty: &'a syn::Type,
    ident: syn::Ident,
}

impl<'a> FieldExt<'a> {
    fn new(field: &'a syn::Field) -> Self {
        Self {
            ty: &field.ty,
            ident: field.ident.clone().unwrap(),
        }
    }

    fn as_args(&self) -> proc_macro2::TokenStream {
        let name = &self.ident;
        let ty = self.ty;
        quote_spanned! {proc_macro2::Span::call_site() => #name: #ty}
    }

    fn as_init(&self) -> proc_macro2::TokenStream {
        let name = &self.ident;
        if self.is_phantom_data() {
            quote_spanned! {proc_macro2::Span::call_site() => #name: PhantomData}
        } else {
            quote_spanned! {proc_macro2::Span::call_site() => #name: #name}
        }
    }

    fn is_phantom_data(&self) -> bool {
        match *self.ty {
            syn::Type::Path(syn::TypePath {
                qself: None,
                ref path,
            }) => path
                .segments
                .last()
                .map(|x| x.ident == "PhantomData")
                .unwrap_or(false),
            _ => false,
        }
    }
}

#[proc_macro_derive(New)]
pub fn derive(input: TokenStream) -> TokenStream {
    let input = parse_macro_input!(input as syn::DeriveInput);
    let name = input.ident;
    let fields: Vec<_> = match input.data {
        syn::Data::Struct(ref s) => match s.fields {
            syn::Fields::Named(ref fields) => {
                fields.named.iter().map(|f| FieldExt::new(f)).collect()
            }
            _ => {
                unimplemented!()
            }
        },
        _ => {
            unimplemented!()
        }
    };
    let args = fields.iter().filter(|f| !f.is_phantom_data()).map(|f| f.as_args());
    let inits = fields.iter().map(|f| f.as_init());
    let fn_new = syn::Ident::new("new", proc_macro2::Span::call_site());
    let (impl_generics, ty_generics, where_clause) = input.generics.split_for_impl();
    let expanded = quote! {
        impl #impl_generics #name #ty_generics #where_clause {
            pub fn #fn_new(#(#args),*) -> Self {
                Self {
                    #(#inits),*
                }
            }
        }
    };
    expanded.into()
}
```
原本的```derive-new```有比较多的特性，支持命名结构体还有匿名结构体，这里为了方便分析只是把核心的命名结构体的逻辑抽离出来。

为了方便构造初始化代码还有参数代码，使用了```struct FieldExt<'a>```结构体进行辅助，参数一般形式是：变量名: 变量类型，如```fn as_args(&self) -> proc_macro2::TokenStream```。初始化一般形态是: ```Self {变量名: 参数名}```，这里成员变量和参数名都是一样的，另外一点如果成员是```PhantomData```，则不需要通过参数进行构造，默认填```PhantomData```。

```rust
struct FieldExt<'a> {
    ty: &'a syn::Type,
    ident: syn::Ident,
}

impl<'a> FieldExt<'a> {
    fn new(field: &'a syn::Field) -> Self {
        Self {
            ty: &field.ty,
            ident: field.ident.clone().unwrap(),
        }
    }

    fn as_args(&self) -> proc_macro2::TokenStream {
        let name = &self.ident;
        let ty = self.ty;
        quote_spanned! {proc_macro2::Span::call_site() => #name: #ty}
    }

    fn as_init(&self) -> proc_macro2::TokenStream {
        let name = &self.ident;
        if self.is_phantom_data() {
            quote_spanned! {proc_macro2::Span::call_site() => #name: PhantomData}
        } else {
            quote_spanned! {proc_macro2::Span::call_site() => #name: #name}
        }
    }

    fn is_phantom_data(&self) -> bool {
        match *self.ty {
            syn::Type::Path(syn::TypePath {
                qself: None,
                ref path,
            }) => path
                .segments
                .last()
                .map(|x| x.ident == "PhantomData")
                .unwrap_or(false),
            _ => false,
        }
    }
}
```

# 最后

本文也是在学习Rust宏系统中的一些经验和感悟。如有不对的地方，欢迎提出反馈，谢谢。如果有其他想要了解的也可以留言，有时间再继续研究研究。

# 参考资料

- 《Rust编程之道》
- heapsize: https://github.com/dtolnay/syn/tree/master/examples/heapsize
- derive-new: https://github.com/nrc/derive-new
