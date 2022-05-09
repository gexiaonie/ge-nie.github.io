---
title: Rust LRU
date: 2022-04-25 19:39:53
tags: [Rust,Algorithm]
cover: https://www.rust-lang.org/static/images/rust-logo-blk.svg
categories: Rust
---

# 前言

Lru算法本身其实不难，双向链表 + HashMap即可实现O(1)复杂度的Lru Cache。但是双向链表是Rust一块硬砖，因为双向链表各个节点有相互引用，而Rust的所有权以及生命周期等特性，导致用Rust来实现双向链确实有一点的难度。可以阅读这篇文章：[Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/)，如果你对Rust有了一点基础，但是写双向链表还是比较吃力，这篇文章很适合你。（看这篇文章就是作者用Rust写双向链表的血泪史，编译报错然后各种fix，再报错，再fix...）

本文在介绍Lru实现之前，介绍了一些基础知识，如果已经了解了可以直接跳到[Lru](#Lru)段落。Lru源码地址: https://github.com/remove-if/lru

# 双向链表

## Safe Rust

最开始看Rust的我，直接跳过unsafe了。因为我觉得我这种追求完美的人，怎么可能会写unsafe代码呢？但是是人都逃不过真香定律，被啪啪打脸。
先来看看如何用Rust safe code定义一个双向链表

```rust
struct Node<T> {
    ele: T,
    prev: Option<Rc<RefCell<Node<T>>>>,
    next: Option<Rc<RefCell<Node<T>>>>,
}

pub struct LinkedList<T> {
    head: Option<Rc<RefCell<Node<T>>>>,
    tail: Option<Rc<RefCell<Node<T>>>>,
}
```

稍微解释一下node的prev, next类型: ```Option<Rc<RefCell<Node<T>>>>```

- 首先prev,next是可有可无的，所以是Option类型
- 由于prev和next是对其他节点的引用，所以没有对应节点的所有权，采用Rc共享所有权（Rc表示不可变的shared_ptr）
- 因为双向链表的节点变动会牵涉prev和next字段的变动，但是Rc是不可变的（就算node设置为mut也不行），所以需要RefCell包裹一下变成可变的。一般Rc都和RefCell配合使用的
- 双向链表使用智能指针会存在相互引用的问题，可以使用Weak。但是Weak解引用时unsafe的，所以这里还是采用Rc
- 可以自定义析构函数主动释放内存，从而解决Rc相互应用的问题

下面双向链表部分具有代表性的API的实现
```rust
use std::cell::{RefCell, Ref};
use std::rc::Rc;
struct Node<T> {
    ele: T,
    prev: Option<Rc<RefCell<Node<T>>>>,
    next: Option<Rc<RefCell<Node<T>>>>,
}

impl<T> Node<T> {
    fn new(ele: T) -> Self {
        Self {
            ele,
            prev: None,
            next: None,
        }
    }
}

pub struct LinkedList<T> {
    head: Option<Rc<RefCell<Node<T>>>>,
    tail: Option<Rc<RefCell<Node<T>>>>,
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            head: None,
            tail: None,
        }
    }

    pub fn push_back(&mut self, ele: T) {
        // 创建node，不需要设置为mut
        // 因为prev, next字段都由RefCell包裹，
        // 所以prev, next都是可变的
        let node = Rc::new(RefCell::new(Node::new(ele)));
        // 这里要特别主意一下
        // 因为match会导致所有权转移
        // tail是属于self的字段，rust不允许所有权被转移走
        // 这里使用Option的take方法，把内部值转移走，而self.tail变为None
        match self.tail.take() {
            Some(tail) => {
                // borror_mut是RefCell的方法，让内部的值变为可变
                // Rc没有实现Copy Trait，但是实现clone方法，不过需要手动调用一下该方法
                // 代码逻辑比较简单了: 如果tail存在则往后追加节点，并把节点链接起来
                tail.borrow_mut().next = Some(node.clone());
                node.borrow_mut().prev = Some(tail);
                self.tail = Some(node);
            }
            None => {
                // 如果self.tail是None表示第一次push，则更新一下self.head
                // 因为双向链表只有一个值，self.head和self.tail应该是一样的
                self.head = Some(node.clone());
                self.tail = Some(node);
            }
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        // take方法见push_back方法中的注解
        // 因为pop_back方法有返回值，采用Option::map的方式
        // 比较自然，如果self.tail是None就直接返回None
        self.tail.take().map(|node| {
            // 判断最后一个节点有没有prev节点
            // 如果有则断开，如果没有则把self.head和tail一起变成None
            match node.borrow_mut().prev.take() {
                Some(head) => {
                    head.borrow_mut().next = None;
                    self.tail = Some(head);
                }
                None => {
                    self.head = None;
                    self.tail = None;
                }
            }
            // 这里比较关键
            // 我们来捋捋，node是Rc类型，
            // 表示智能指针，共享了所有权
            // 但是pop则表示把node从双向链表中删除，即所有权转移走
            // 我们又不知道有没有其他地方共享了所有权，所以使用Rc::try_unwrap
            // 这个try很关键，因为编译器不知道，所以需要运行时判断
            // 中间的ok函数表示把Result类型转成Option类型
            // into_inner是将RefCell<T>转成T，最终所有权被释放出来了
            Rc::try_unwrap(node).ok().unwrap().into_inner().ele
        })
    }
    /*
    pub fn peek_back(&self) -> Option<&T> {
        self.tail.as_ref().map(|node| {
            // error[E0515]: cannot return value referencing temporary value
            // 这是因为node.borrow()返回了一个临时变量Ref<Node<T>>
            // 所以无法返回临时变量的引用（这个也是C++中常见的问题）
            &node.borrow().ele
        })
    }
    */
    pub fn peek_back(&self) -> Option<Ref<T>> {
        self.tail.as_ref().map(|node|{
            // 由于node.borrow()返回的是Ref<Node<T>>
            // 如果peek_back直接返回Ref<Node<T>>，则把内部的细节Node类型
            // 暴露给用户，所以需要把内部细节屏蔽掉
            // 使用Ref::map可以把内部字段映射出来
            Ref::map(node.borrow(), |node| &node.ele)
        })
    }
}

// 由于节点相互依赖，所以无法主动释放内存
// 需要自定义析构函数主动释放(pop_back)所有元素
impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        while self.pop_back().is_some() {}
    }
}
```

上述代码实现了LinkedList的部分API，push_back、pop_back、peek_back，front_xxx等API道理和push相似可以模仿自行实现。
下面来总结一下上述代码的核心知识点

- 节点类型Option<Rc<RefCell<Node<T>>>>
- Option的take方法很重要，可以在不移动原始值的所有权的情况下把内部的value转移走。针对Option是成员变量，需要使用match操作很方便，见API push_back和pop_back
- Option的take方法是移走内部的value，但是如果Option是不可变的怎么处理呢？可以使用Option的as_ref()方法，这个函数作用是将Option<T>转成Option<&T>。match匹配还有另一种方式，见下面的Option模式匹配代码
- peek_back的返回值只能是Option<Ref<T>>，不能返回Option<&T>。主要原因是node.borrow()返回的是类型为Ref<Node<T>>的临时变量，而函数内部的临时变量无法作为返回引用（局部变量离开作用域就销毁了）
- 由于双向链表的节点相互应用了，所以会造成循环依赖。需要自定义析构函数，再析构函数中主动释放(pop_back)所有元素。（为什么不采用Weak呢？因为Weak解引用时需要使用unsafe code，这里我们只讲safe code实现方式）
- 内存泄露不属于内存安全问题，所以即使Safe Code也是有可能有内存泄露的问题

## Option匹配

```rust
// 错误 self.tail所有权被移走了，这个和self.tail是None是有区别的.
match self.tail {
    Some(tail) => {},
    None => {}
}
// 正确，这里是引用方式匹配
match self.tail {
    Some(ref tail) => {},
    None => {}
}
// 正确，这里是引用方式匹配
match self.tail.as_ref() {
    Some(tail) => {},
    None => {}
}
// 正确，把self.tail移动走，让self.tail变为None，但是self.tail所有权还在
match self.tail.take() {
    Some(tail) => {},
    None => {}
}
```

## Unsafe Rust
被啪啪打脸的我，开始看unsafe rust部分的知识了。首先我们要有一个认知，写unsafe代码是否就代表我们的代码存在不安全的风险？并不一定，只是Rust不帮你保障绝对安全，但是我们自己可以保障安全嘛。但是有人会问：我都写unsafe了，为啥不直接写C++呢？我来简单回答一下这个问题：我们完全可以用C++写出一个非常安全的Lru Cache来，各种单测和Effective C++来保证。那我们为什么还要用Rust呢? 个人理解，简单来说Rust把unsafe代码圈在一个很小范围，而外部任然会有Rust safe保驾护航。比如Lru Cache内部使用unsafe实现，但是我们的接口都是safe的。很小的范围，开发者还是比较有能力保障其正确性的。同样出问题，也可优先排查unsafe代码块。

预备知识

- ```unsafe```
- ```NonNull<T>```

### Unsafe
- Unsafe Rust是指在进行一下五种操作的时候，并不会提供任何检查:
- 解引用裸指针
- 调用unsafe的函数或方法
- 访问或修改可变静态变量
- 实现unsafe trait
- 读写Union联合体的字段 —— 《Rust编程之道》

本次我们主要需要使用到是unsafe的解引用裸指针。

Rust提供```*const T```（不变）和```*mut T```（可变）两种指针类型。因为这两种指针和C语言的指针十分相似，所以也叫其原始指针（Raw Pointer）。原始指针具有一下特点:

- 并不保证指向合法的内存。比如很有可能是一个空指针
- 不能像智能指针那样自动清理内存。需要像C语言那样手动管理内存
- 没有生命周期的概念。也就是说，编译器不会对其提供借用检查
- 不能保障线程安全 —— 《Rust编程之道》

```rust
pub struct Node<T> {
    ele: T,
    prev: *const Node<T>,
    next: *const Node<T>,
}

impl<T> Node<T> {
    pub fn new(ele: T) -> Self {
        Self {
            ele,
            prev: std::ptr::null(),
            next: std::ptr::null(),
        }
    }
}

fn main() {
    // 声明两个node，相互引用
    let mut node1 = Node::new(1);
    let mut node2 = Node::new(2);
    // 这里演示对node进行*mut T的引用
    // 所以Rust并不会检测mut引用的唯一性
    // 再者将&T转成*cont T，或者&mut T转成*mut T
    // 是安全操作，不需要unsafe包裹
    // 因为获取一个变量的地址是安全的（这里不讨论右值）
    let n1 = &mut node1 as *mut Node<i32>;
    let n2 = &mut node1 as *mut Node<i32>;
    // 将两个node串联起来
    node1.next = &node2 as *const Node<i32>;
    node2.prev = &node1 as *const Node<i32>;
    // 这里是对node1的next字段解引用
    // 因为不知道这个地址是否有效
    // 所以解引用裸指针是unsafe操作，用户需要保证正确
    let node = unsafe { &*node1.next };
    // 创建Box，类似unique_ptr
    let node3 = Box::new(Node::new(3));
    // 将Box主动泄露，则需要手动释放内存
    let node4 = Box::leak(node3) as *const Node<i32>;
}
```

裸指针用起来感觉和C/C++比较像，没有Rust的所有权和生命周期的"限制"。所以在某些场景用起来还是很顺手的，比如双向链表的节点引用用裸指针更合适。
不过```*const T```和```&T```以及mut等裸指针和引用之间的相互转换，还是比较繁琐的。比如```*const T```转成```&T```，```unsafe{ &*node.next }```，首先```*node```表示对裸指针解应用。裸指针只是对值的引用，没有所用权。所以```let node1 = unsafe{ *node }```是错误的，因为这句话表示node的所有权被转移到node1，所以一定需要增加一个引用才行，一般都是```&*```操作。
而且我们还需要像C/C++一样要判断指针是否为空，例如```node.is_null()```。不过写起来不像Rust，有点像C/C++，Rust更倾向于使用```Option```。所以采用```Option<NonNull<T>>```的方式会更加优雅，更加Rust。

### ```NonNull<T>```

```rust
pub struct NonNull<T: ?Sized> {
    pointer: *const T,
}

impl<T: ?Sized> NonNull<T> {
    pub const fn as_ptr(self) -> *mut T {
        self.pointer as *mut T
    }

    pub unsafe fn as_ref<'a>(&self) -> &'a T {
        unsafe { &*self.as_ptr() }
    }

    pub unsafe fn as_mut<'a>(&mut self) -> &'a mut T {
        unsafe { &mut *self.as_ptr() }
    }

    #[stable(feature = "nonnull", since = "1.25.0")]
    impl<T: ?Sized> Clone for NonNull<T> {
        #[inline]
        fn clone(&self) -> Self {
            *self
        }
    }

    #[stable(feature = "nonnull", since = "1.25.0")]
    impl<T: ?Sized> Copy for NonNull<T> {}
}
```

上面是```NonNull<T>```的部分源码，可以看出其只有一个成员变量```pointer: *const T```，即只有一个裸指针成员。方法```as_ref```和```as_mut```分别解引用为引用和可变引用，替代我们之前繁琐的```unafe {&*ptr}```。从语义来说```NonNull<T>```表示一个非空的裸指针，所以一般配合```Option```一起使用，这样就可以使用```match```或者```map```语法了。

## Unafe LinkedList

```rust
use std::{marker::PhantomData, ptr::NonNull};

struct Node<T> {
    ele: T,
    prev: Option<NonNull<Node<T>>>,
    next: Option<NonNull<Node<T>>>,
}

pub struct LinkedList<T> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    // 幽灵数据，本质上是一个标记。因为LinkedList使用了NonNull，类似裸指针，
    // 所以LinkedList和T之间的关系就有点模糊不清了。
    // 比如LinkedList析构的时候是否需要析构T，
    // 如果把LinkedList使用默认的析构函数，
    // 那么T肯定没有被析构，存在内存泄露。
    // 所以使用LinkedList的人就会比较迷惑，所以需要加个标记，
    // 标记LinkedList拥有T，即LinkedList析构，T也就析构了，
    // 同理T的生命周期不可能超过LinkedList，这里的不超过指的是生命周期的结束点
    marker: PhantomData<T>,
}

impl<T> LinkedList<T> {
    pub fn new() -> Self {
        Self {
            head: None,
            tail: None,
            marker: PhantomData,
        }
    }
    pub fn push_back(&mut self, ele: T) {
        // 这里使用Box::new创建一个堆对象，
        // 然后通过Box::leak主动将Box对象泄露出去，
        // 因为Box类似unique_ptr，离开作用域就析构了
        // 所以Box::leak(Box::new) 就等效于C++中的new，如果不主动释放则存在内存泄露
        let node = Box::leak(Box::new(Node {
            ele,
            prev: self.tail,
            next: None,
        }))
        .into();
        // 这里和之前的safe代码很像了。
        // 如果函数有返回值，推荐使用Option::map方法。
        // 如果函数没有返回值，推荐使用match匹配Option。
        // 这里为什么没有使用 match self.tail.take呢？
        // 是不是self.tail所有权被移走了呢？
        // 其实这里是因为NonNull具有Copy语义，复制了。
        // 如果T是Copy语义，Option<T>也具备Copy语义
        match self.tail {
            Some(mut tail) => unsafe {
                tail.as_mut().next = Some(node);
                self.tail = Some(node);
            },
            None => {
                self.head = Some(node);
                self.tail = Some(node);
            }
        }
    }

    pub fn pop_back(&mut self) -> Option<T> {
        // 如push_back函数说明一样，这里采用Option::map方法
        // 主意NonNull的as_mut方法的签名是pub unsafe fn as_mut<'a>(&mut self) -> &'a mut T
        // 所以需要NonNull是可变的，所以在map的闭包中的参数是mut
        self.tail.take().map(|mut tail| unsafe {
            match tail.as_mut().prev {
                Some(mut prev) => {
                    prev.as_mut().next = None;
                    tail.as_mut().prev = None;
                    self.tail = Some(prev);
                }
                None => {
                    self.head = None;
                    self.tail = None;
                }
            }
            // 这里特别要主意一下，因为是pop_back，
            // 所以需要主动释放对应Node的内存
            // 这里就类似让unique_ptr重新接管裸指针，
            // 离开作用域自动析构
            let node = Box::from_raw(tail.as_ptr());
            // 这里把node的ele成员所有权转移走了
            // 为什么这里可以转移某个变量的成员呢？
            // 之前写Safe Rust的时候，self.tail的所有权为什么无法转移呢？
            // 这是因为node已经没有地方再使用了，而且node没有自定义析构函数。
            // 可以认为ele成员没有任何地方再使用了，所以可以安全被转移走了。
            node.ele
        })
    }

    pub fn peek_back(&self) -> Option<&T> {
        // 这里的peek_back就比较简单了，而且返回值更加自然Option<&T>，
        // 而Safe Code只能返回Option<Ref<T>>，原因见Safe Code代码说明
        self.tail.map(|node| unsafe { &node.as_ref().ele })
    }
}

impl<T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        while self.pop_back().is_some() {}
    }
}
```
上述代码参考了Rust标准库里面的```std::collections::LinkedList```实现，完整的代码可以直接阅读标准库代码。可以看到标准库都合理的使用了Unsafe Rust了，我们又有什么理由完全拒绝呢？再次被啪啪打脸。

关于Unsafe的LinkedList实现原理和逻辑，在上述代码中已经给出了比较详细的注释了，这里再简单总结几点:

- NonNull实现了Copy语义
- 可以通过Box::leak(Box::new).into生成NonNull，类似C++中的new
- NonNull的as_mut方法需要NonNull自身是可变的
- NonNull需要手动释放内存，可以通过Box::from_raw(xx.as_ptr())，让智能指针重新接管裸指针

# Lru

之前前言中已经简单说过Lru O(1)复杂度的实现方案，即hashmap + 双线链表。其中双向链表维护着Cache访问热点，优先淘汰冷数据；而hashmap才是O(1)的关键，可以快速通过key查询。

不过用Rust实现Lru cache比双向链表多了一个挑战，就是key的共享问题。我们来思考几个问题：

- 双向链表中的节点肯定需要存储key和value，因为如果数据被删除了，需要通过key清理map中的数据
- 需要通过key在hashmap中查找对应的value信息，所以key肯定需要存储再map中

由上面分析可见，key肯定需要再hashmap和双向链表中进行共享。

为什么我们用C++写不会有这么多破事呢？

因为C++默认都是值复制即Rust的Copy语义，用C++写其实我们都复制了key，即一份再双向链表中，一份再hashmap中。

为什么Rust不能像C++一样复制key呢？

如果在Rust里面需要复制key，则需要key至少实现Clone Trait。这样使用范围就变得很局限，有些时候有些Key无法实现Clone，这样就无法使用该库了。

再看key共享之前，先来看看HashMap的get方法定义，

```rust
impl<K, V> HashMap<K, V> {
    pub fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq,
    {
        self.base.get(k)
    }
}
```
主意这里的get方法的参数k类型不是&K，而是&Q，而且有一个trait限定```K: Borrow<Q>```。再来看看这个```Borrow``` trait定义，
```rust
pub trait Borrow<Borrowed: ?Sized> {
    /// Immutably borrows from an owned value.
    ///
    /// # Examples
    ///
    /// ```
    /// use std::borrow::Borrow;
    ///
    /// fn check<T: Borrow<str>>(s: T) {
    ///     assert_eq!("Hello", s.borrow());
    /// }
    ///
    /// let s = "Hello".to_string();
    ///
    /// check(s);
    ///
    /// let s = "Hello";
    ///
    /// check(s);
    /// ```
    #[stable(feature = "rust1", since = "1.0.0")]
    fn borrow(&self) -> &Borrowed;
}
```
Borrow trait表示从当前类型出借一个Borrowed类型，举个栗子：
```rust
use std::borrow::Borrow;


pub struct Node<K, V> {
    key: K,
    value: V
}

impl<K, V> Node<K, V> {
    pub fn get_key(&self) -> &K {
        &self.key
    }
}

impl<K, V> Borrow<K> for Node<K, V> {
    fn borrow(&self) -> &K {
        &self.key
    }
}

fn main() {
    let node = Node{key: 1, value: 2};
    // 下面两种写法是等价的，都是获取key不可变引用
    let k:&i32 = node.borrow();
    let k1 = node.get_key();
}
```
上述代码中我们可以通过两种方式获取key的不可变引用，第一种直接通过方法```get_key```返回，第二种通过Borrow trait返回了key的引用。

了解了```Borrow``` trait再来看看HashMap的get方法定义，
```rust
impl<K, V> HashMap<K, V> {
    pub fn get<Q: ?Sized>(&self, k: &Q) -> Option<&V>
    where
        K: Borrow<Q>,
        Q: Hash + Eq,
    {
        self.base.get(k)
    }
}
```
这样我们就能理解为什么get方法中的参数k的类型是&Q了，因为HashMap的查询是通过HashMap存储的Key的Borrow方法返回值和参数k进行对比。举个例子，
```rust
use std::{borrow::Borrow, collections::HashMap, hash::Hash};

pub struct Node<K, V> {
    key: K,
    value: V,
}

impl<K, V> Node<K, V> {
    fn new(key: K, value: V) -> Self {
        Node { key, value }
    }
}

impl<K: Hash, V> Hash for Node<K, V> {
    fn hash<H: std::hash::Hasher>(&self, state: &mut H) {
        self.key.hash(state)
    }
}

impl<K, V> Borrow<K> for Node<K, V> {
    fn borrow(&self) -> &K {
        &self.key
    }
}

impl<K: Eq, V> PartialEq for Node<K, V> {
    fn eq(&self, other: &Self) -> bool {
        self.key.eq(&other.key)
    }
}

impl<K: Eq, V> Eq for Node<K, V> {}

fn main() {
    let mut m = HashMap::new();
    // HashMap的key是Node
    m.insert(Node::new(1, 1), 1);
    // 虽然key的类型是Node，但是我们可以用&i32查询
    let v = m.get(&1);
}
```
上述例子中，HashMap的key类型是```Node<i32, i32>```。因为Node类型实现了```Borrow<Q>```，所以get获取的时候可以通过```&Q```进行查询，其中```Q```即i32类型。

通过上面讲解的HashMap的get方法查询原理，是不是已经想到了共享Key的方法了？

共享Key的方法就是: HashMap的Key和Value类型都是NonNull，然后给NonNull实现Borrow trait，出借key。

了解双向链表和HashMap的get方法，实现Lru Cache就不难了。

```rust
use std::{borrow::Borrow, collections::HashMap, hash::Hash, marker::PhantomData, ptr::NonNull};

struct Node<K, V> {
    k: K,
    v: V,
    prev: Option<NonNull<Node<K, V>>>,
    next: Option<NonNull<Node<K, V>>>,
}

// 因为crate的孤儿规则，无法给其他模块实现trait实现，
// 所以这里类似给NonNull alias一个新的名字
struct KeyRef<K, V>(NonNull<Node<K, V>>);

// Borrow出借key
impl<K: Hash + Eq, V> Borrow<K> for KeyRef<K, V> {
    fn borrow(&self) -> &K {
        unsafe { &self.0.as_ref().k }
    }
}

// 转发到node.k上面
impl<K: Hash, V> Hash for KeyRef<K, V> {
    fn hash<H: std::hash::Hasher>(&self, state: &mut H) {
        unsafe { self.0.as_ref().k.hash(state) }
    }
}

// 转发到node.k上面
impl<K: Eq, V> PartialEq for KeyRef<K, V> {
    fn eq(&self, other: &Self) -> bool {
        unsafe { self.0.as_ref().k.eq(&other.0.as_ref().k) }
    }
}

// 转发到node.k上面
impl<K: Eq, V> Eq for KeyRef<K, V> {}

impl<K, V> Node<K, V> {
    fn new(k: K, v: V) -> Self {
        Self {
            k,
            v,
            prev: None,
            next: None,
        }
    }
}

pub struct LruCache<K, V> {
    head: Option<NonNull<Node<K, V>>>,
    tail: Option<NonNull<Node<K, V>>>,
    // 本质上HashMap的Key和Value类型都是NonNull类型，
    // 这样就达到共享Key的目的了
    map: HashMap<KeyRef<K, V>, NonNull<Node<K, V>>>, 
    cap: usize,
    marker: PhantomData<Node<K, V>>, // 和LinkedList类似
}

impl<K: Hash + Eq + PartialEq, V> LruCache<K, V> {
    pub fn new(cap: usize) -> Self {
        assert!(cap > 0);
        Self {
            head: None,
            tail: None,
            map: HashMap::new(),
            cap,
            marker: PhantomData,
        }
    }

    pub fn put(&mut self, k: K, v: V) -> Option<V> {
        // new 一个新node
        let node = Box::leak(Box::new(Node::new(k, v))).into();
        // 删除可能存在的老的key
        let old_node = self.map.remove(&KeyRef(node)).map(|node| {
            // 如果存在老的key，
            // 将其从双向链表中删除
            self.detach(node);
            node
        });
        // 如果容量已经超限，则删除尾部元素
        if self.map.len() >= self.cap {
            let tail = self.tail.unwrap();
            self.detach(tail);
            self.map.remove(&KeyRef(tail));
        }
        // 新节点插入到双向链表头部
        self.attach(node);
        // 并且将Node关系添加到map中
        self.map.insert(KeyRef(node), node);
        // 如果该key存在老数据，则返回老key的value
        old_node.map(|node| unsafe {
            let node = Box::from_raw(node.as_ptr());
            node.v
        })
    }

    pub fn get(&mut self, k: &K) -> Option<&V> {
        // 在map中查询value
        if let Some(node) = self.map.get(k) {
            let node = *node;
            // 查询到之后，需要提升为最热的数据
            // 将node从双向链表中删掉，并添加到头部
            self.detach(node);
            self.attach(node);
            unsafe { Some(&node.as_ref().v) }
        } else {
            None
        }
    }

    fn detach(&mut self, mut node: NonNull<Node<K, V>>) {
        // 删除双向链表的节点
        unsafe {
            match node.as_mut().prev {
                Some(mut prev) => {
                    prev.as_mut().next = node.as_ref().next;
                }
                None => {
                    self.head = node.as_ref().next;
                }
            }
            match node.as_mut().next {
                Some(mut next) => {
                    next.as_mut().prev = node.as_ref().prev;
                }
                None => {
                    self.tail = node.as_ref().prev;
                }
            }

            node.as_mut().prev = None;
            node.as_mut().next = None;
        }
    }

    fn attach(&mut self, mut node: NonNull<Node<K, V>>) {
        // 将节点添加到双向链表头部
        match self.head {
            Some(mut head) => {
                unsafe {
                    head.as_mut().prev = Some(node);
                    node.as_mut().next = Some(head);
                    node.as_mut().prev = None;
                }
                self.head = Some(node);
            }
            None => {
                unsafe {
                    node.as_mut().prev = None;
                    node.as_mut().next = None;
                }
                self.head = Some(node);
                self.tail = Some(node);
            }
        }
    }
}

impl<K, V> Drop for LruCache<K, V> {
    fn drop(&mut self) {
        // 需要手动关系内存，将所有node都pop出去
        while let Some(node) = self.head.take() {
            unsafe {
                self.head = node.as_ref().next;
                drop(node.as_ptr());
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use crate::LruCache;

    #[test]
    fn it_works() {
        let mut lru = LruCache::new(3);
        assert_eq!(lru.put(1, 10), None);
        assert_eq!(lru.put(2, 20), None);
        assert_eq!(lru.put(3, 30), None);
        assert_eq!(lru.get(&1), Some(&10));
        assert_eq!(lru.put(2, 200), Some(20));
        assert_eq!(lru.put(4, 40), None);
        assert_eq!(lru.get(&2), Some(&200));
        assert_eq!(lru.get(&3), None);
    }
}
```

# 最后

本文主要是个人在学习Rust，并用Rust写Lru Cache的一些想法和感悟。如有不对的地方，欢迎提供反馈。如有其他想要了解的也可以留言，有时间可以再研究研究。觉得不错就点个赞吧。

