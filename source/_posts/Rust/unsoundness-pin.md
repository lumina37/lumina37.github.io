---
title: 不完善的Pin
categories: Rust
date: 2024/8/6 21:36:00
---

## 译者序

`Pin`是Rust针对自引用结构体在移动时发生指针悬垂所提出的解决方案。本文是一篇Rust官方论坛语言设计板块的[帖文](https://internals.rust-lang.org/t/unsoundness-in-pin/11311)的中译（DeepL机翻+人工精修），主要讨论`Pin`的不完善性。

在阅读前需要牢记一个印象：大部分“平凡”对象如`i32`、`String`都是`Unpin`的，这意味着这些对象可以被安全地移动到其他内存位置，只有`Future`等自引用对象才有必要成为`!Unpin`。

## 正文

最近@withoutboats（Rust异步设计的核心参与者）向我提出了一个[挑战](https://www.reddit.com/r/rust/comments/dtfgsw/comment/f7bdzyx/?context=3)。他要求我提出一个具有不同保证（也就是不靠屏蔽`&mut T`来实现自引用安全）的`Pin`版本，并说明这个新`Pin`的完善性。然而，就在我做这项工作的时候，我偶然发现了当前`Pin`的不完善性。我以前从未见过这方面的报道，不过也有可能是我疏忽了。

playground：https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=70b7b28b690e0020f52695b9e086f5c2

这段代码演示了从`&mut Pin<P>`获取`Pin<&mut P::T>`的两个相关漏洞，最终导致了一个段错误。该代码使用稳定版本编译，没有不安全代码块。唯一的非标准库导入是来自futures的两样东西；它们只是为了方便，对漏洞利用并不重要。

以下列举一些使用“安全”代码触发UB的方法：

+ 方法1：impl `DerefMut` for `&dyn Trait`
+ 方法2：impl `Clone` for `&mut dyn Trait`
+ 依赖非稳定特性的方法3：`CoerceUnsized`到具有不同`Deref`实现的类型
+ 方法4在帖文的评论区补充

### 方法1 - `DerefMut`

以下是`Pin::as_mut`的简化版定义：

```rust
impl<P: DerefMut> Pin<P> {
    pub fn as_mut(&mut self) -> Pin<&mut P::Target> {
        unsafe { Pin::new_unchecked(&mut *self.pointer) }
    }
}
```

注释里声明这么做是安全的：对`Pointer::DerefMut`的恶意实现会被`Pin::new_unchecked`的约定排除。

译注：所谓“对`Pointer::DerefMut`的恶意实现”是指改变对象内存位置的`DerefMut`实现，例如在解引用时重分配内存并返回一个全新的对象。

这在很大程度上是正确的。通常你无法安全地获取所有自定义对象`P`的`Pin<P>`。如果`P`实现了`Deref<Target: Unpin>`，那么直接用安全的`Pin::new`就完事了。但`Pin`一个`Unpin`的对象通常是无趣的（译注：因为`Unpin`对象天然可以自由移动，`Pin`在这里只是一把无趣的枷锁）。否则，就只能使用不安全的`Pin::new_unchecked`，或者针对性地返回`Pin<P>`的各种安全封装，例如`Pin<Box<T>>`、`Pin<&mut T>`和`Pin<&T>`（`T`本身是任意的，但必须用这些特定指针类型之一封装）。

但是，我们不能使用自定义的类型并不意味着我们不能使用自定义的实现！

供参考：一致性规则意味着用户只能为那些我称之为“本地（local-ish）”的类型实现外部Trait，如`DerefMut`。本地类型通常是指当前crate中定义的结构体、枚举和`Trait`对象。但如果`T`是本地类型，那么`&T`、`&mut T`、`Box<T>`和`Pin<T>`也会被视为局部类型。这些包装器被称为“基础（fundamental）”，如[RFC 1023](https://rust-lang.github.io/rfcs/1023-rebalancing-coherence.html)所述。

因此，对于某个本地类型`Foo`，我们可以尝试为`&Foo`、`&mut Foo`、`Box<Foo>`或`Pin<Foo>`中的任何一个添加 `DerefMut`实现。其中，我们可以排除`&mut Foo`和`Box<Foo>`的可能性，因为它们已经实现了`DerefMut`。我们可以排除`Pin<Foo>`的可能性，因为它对我们的目的没有用处，因为没有办法获得它的pinned版本（即`Pin<Pin<Foo>>`）。

这样就只剩下`&Foo`了——而且这条路确实走得通！`&Foo`没有实现`DerefMut`，我们只需使用`Pin::as_ref`就能得到它的`Pin`版本（即`Pin<&Foo>`）。

下面为`&Foo`实现`DerefMut`：

```rust
impl<'a> DerefMut for &'a Foo {
    // ...
}
```

我们无法自定义`Deref`的类型。`DerefMut`重用了`Deref<Target>`中的关联类型`Target`，而不可变引用已经有了`Deref`的实现。在本例中，`Target`将是`Foo`，因此我们必须提供一个具有以下签名的函数：

```rust
fn deref_mut(self: &mut &'a Foo) -> &mut Foo
```

然后，我们必须获得一个合法pinned的`Pin<&Foo>`，并调用`as_mut`。`as_mut`将被委托给我们自定义的`deref_mut`。`self`将是一个合法的pinned引用，但我们可以在`deref_mut`中返回一个完全不同的引用，然后`as_mut()`将以不安全的方式将其封装为`Pin`。

在playground中作者通过`replace`方法将`&Cell`的内部值修改成了`None`，这事实上已经违背了`Pin`的规则。

输入和输出都必须指向`Foo`，这在一定程度上造成了限制，但并不是致命的。为了能够在不使用不安全代码的情况下返回引用（我们可以使用`Box::leak`安全地返回`&'static mut`引用，但这样就无法利用漏洞触发UB：因为事后无法恢复引用），引用需要存储在输入`Foo`中，而输出`Foo`必须是某种`!Unpin`类型，在pin之后后移动这种类型实际上是很危险的。最直接的方法（尽管不是唯一的方法）是将`Foo`变为一个Trait对象，这样输入和输出就可以完全是不同的具体类型。更多详情，请参阅playground。

### 方法2 - `Clone`

`Pin`派生了`Clone`特性。

```rust
#[derive(Copy, Clone, Hash, Eq, Ord)]
pub struct Pin<P>
```
