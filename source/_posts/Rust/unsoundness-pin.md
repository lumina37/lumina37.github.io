---
title: Unsoundness in Pin
categories: Rust
date: 2024/8/6 21:36:00
---

## 译者序

`Pin`是Rust针对自引用结构体在移动时发生指针悬垂所提出的解决方案。本文是一篇Rust官方论坛语言设计板块的[帖文](https://internals.rust-lang.org/t/unsoundness-in-pin/11311)的中译（机翻+人工精修），主要讨论`Pin`的不健全性（Unsoundness）。

在阅读前需要牢记一个印象：大部分“平凡”对象如`i32`、`String`都是`Unpin`的，这意味着这些对象可以被安全地移动到其他内存位置，只有`Future`等自引用对象才有必要成为`!Unpin`。

## 正文

最近@withoutboats（Rust异步设计的核心参与者）向我提出了一个[挑战](https://www.reddit.com/r/rust/comments/dtfgsw/comment/f7bdzyx/?context=3)。他要求我提出一个具有不同保证（也就是不靠屏蔽`&mut T`来实现自引用安全）的`Pin`版本，并说明这个新`Pin`的健全性。然而，就在我做这项工作的时候，我偶然发现了当前`Pin`的不健全性。我以前从未见过这方面的报道，不过也有可能是我疏忽了。

playground：https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=70b7b28b690e0020f52695b9e086f5c2

这段代码演示了从`&mut Pin<P>`获取`Pin<&mut P::Target>`的两个相关漏洞，最终导致了一个段错误。该代码使用稳定版本编译，没有不安全代码块。唯一的非标准库导入是来自futures的两样东西；它们只是为了方便，对漏洞利用并不重要。

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

译注：在playground中作者通过`replace`方法将`&Cell`的内部值修改成了`None`，这事实上已经违背了`Pin`的规则。

输入和输出都必须指向`Foo`，这在一定程度上造成了限制，但并不是致命的。为了能够在不使用不安全代码的情况下返回引用（我们可以使用`Box::leak`安全地返回`&'static mut`引用，但这样就无法利用漏洞触发UB：因为事后无法恢复引用），引用需要存储在输入`Foo`中，而输出`Foo`必须是某种`!Unpin`类型，在pin之后后移动这种类型实际上是很危险的。最直接的方法（尽管不是唯一的方法）是将`Foo`变为一个Trait对象，这样输入和输出就可以完全是不同的具体类型。更多详情，请参阅playground。

### 方法2 - `Clone`

`Pin`派生了`Clone`特性。

```rust
#[derive(Copy, Clone, Hash, Eq, Ord)]
pub struct Pin<P>
```

与`Pin::as_mut`在指针上调用`deref_mut`并pin住返回值的方式类似，派生的`Pin::clone`在指针上调用`clone`并pin住返回值。

和之前一样，通过`Pin::clone`触发UB的唯一方式是向现有的本地指针类型中添加一个`Clone`实现。可选项依然是`&Foo`、`&mut Foo`、`Box<Foo>`和`Pin<Foo>`。这次`&Foo`不适用，因为它已经实现了`Clone`（有`Copy`就默认有`Clone`）。`Pin<Foo>`依然没有用处（TODO：为什么？）。

如果`Foo`实现了`Clone`，那么`Box<Foo>`就会自动实现`Clone`。如果`Foo`没有实现`Clone`，我们可以为`Box<Foo>`添加一个自定义的`Clone`实现，但这并没有什么用。如果我们拿到一个`Box<Foo>`，`Pin::clone`会将其转换为`Pin<Box<Foo>>`。但已经有一种方法可以做到这一点，也就是标准库中的`impl<T> From<Box<T>> for Pin<Box<T>>`。将现有的`Box`pin住是完全安全的，因为一旦`Box`被`Pin`包装，就无法再将其取出。

剩下的就是`&mut Foo`，它没有内置的`Clone`实现。与`Box`相比，pin住已存在的引用是危险的，因为我们可以pin住一个重新借用的引用（例如从`RefCell`借用），然后通过让生命周期过期并访问我们最初重新借用的引用，有效地“将其重新取出”。这就是方法1的工作原理，我们在这里也可以这样做。

方法2所需的类型签名很奇怪，但与方法1中的签名类似。上次我们必须实现：

```rust
fn deref_mut(self: &mut &'a Foo) -> &mut Foo
```

这次我们必须实现：

```rust
fn clone(self: &&'a mut Foo) -> &'a mut Foo
```

这里不再是可变引用指向不可变引用，而是不可变引用指向可变引用。生命周期的位置也不同。但除此之外都是一样的，所以漏洞利用实际上非常相似。

### 方法3 - 不稳定特性`CoerceUnsized`

好吧，我不能说这是我发现的，因为在注释中已经或多或少地解释过了：

注释内容：注意：这意味着任何允许从实现了`Deref<Target=impl !Unpin>`的类型自动强制转换（术语为[Type Coercions](https://doc.rust-lang.org/reference/type-coercions.html)）到实现了`Deref<Target=Unpin>`的类型的`Trait CoerceUnsized`实现都是不健全的。不过，出于其他原因，任何这样的实现也可能是不健全的，因此我们只需注意不要让类似下面这样的实现进入标准库。

```rust
impl<P, U> CoerceUnsized<Pin<U>> for Pin<P>
where
    P: CoerceUnsized<U>,
{}
```

这段实现允许你进行未定大小的强制转换，例如将`Pin<&T>`转换为`Pin<&Trait>`，这需要结合`&T`本身的`CoerceUnsized`实现。

与评论相反，我认为最直接的危险是从`Unpin`转换为`!Unpin`。具体来说，就是一个实现允许从一个实现了`Deref`和`Target: Unpin`的类型`P`，强制转换为一个实现了`Deref`和`Target: !Unpin`的类型`U`。这样，你可以使用`Pin::new`创建一个`Pin<P>`，然后将其强制转换为`Pin<U>`。这种实现并不会出于其他原因而“必须不安全”。特别地，它不需要不安全代码，因为`P`和`U`的`Deref`实现并没有必要彼此关联：它们可以返回完全不同类型的引用。

目前，`CoerceUnsized`功能被放在一个特性开关（feature gate）后面，因此评论中提醒“注意不要让此类实现进入标准库”是有道理的。如果没有这些实现，这个问题在稳定版上是无法被利用的。但需要说明的是，通过在nightly版中激活该特性开关，这个问题是可以被利用的。

如果在稳定版上无法被利用，那我为什么还要提到它呢？因为迟早我们会以某种形式将`CoerceUnsized`稳定下来。即使你不打算编写自己的标准库，实现自定义的智能指针类型也是很常见的。然而，虽然标准库中的智能指针都实现了`CoerceUnsized`，例如允许你将`Rc<T>`强制转换为`Rc<Trait>`，但目前对于你自己的类型来说，还没有办法获得同样的功能。这显然是需要解决的问题。

根据该讨论来看，稳定版的`CoerceUnsized`可能会与当前版本有所不同，并且它可能自身也存在健全性问题。但是，如果没有 `Pin`，我认为`Deref`的实现没有任何理由会影响健全性。现在却会产生影响，我不确定是应该通过更改强制转换来解决，还是通过更改`Pin`来解决。更多关于修复的讨论请见下文。

### 关于已有的`CoerceUnsized`实现

是否有办法利用标准库中现有的`CoerceUnsized`实现来攻击 `Pin`？目标是找到一种实现了`CoerceUnsized`的类型，我们可以任意控制它解引用到的内容：要么通过添加我们自己的`Deref`实现，要么通过某种方式利用现有的实现。

省流：没有。

以下列举了一些`CoerceUnsized`的实现：

智能指针类型（以`impl<T, U> CoerceUnsized<Foo<U>> for Foo<T> where T: Unsize<U>`的形式）：

+ `Ref<T>`
+ `RefMut<T>`
+ `Unique<T>`
+ `NonNull<T>`
+ `&T`
+ `&mut T`
+ `*const T`
+ `*mut T`

在这些类型中，`Ref<T>`、`RefMut<T>`、`&T`和`&mut T`都已经实现了 Deref，但它们比较“无趣”：它们只是简单地将自身指针转换为不同的类型，没有什么是我们可以控制的。对于其余的类型，我们可以尝试添加自己的`Deref`实现，但它们都不是基础类型。

（我们可以为`&T`添加一个`DerefMut`实现，但那又回到了方法1，而不需要使用强制转换。）

令人有些惊讶的是，裸指针（raw pointers）并不是基本类型，而引用是——换句话说，你无法为`*const MyType`实现任何特性。这看起来像是一个应该修复的疏忽，但这样做会使这个问题变得可被利用。

类包装器（Wrapper-like）类型（以`impl<T, U> CoerceUnsized<Foo<U>> for Foo<T> where T: CoerceUnsized<U>`的形式）：

+ `Cell<T>`
+ `RefCell<T>`
+ `UnsafeCell<T>`
+ `Pin<T>`

前面三个没有实现 `Deref`。

`Pin`实现了`impl<P: Deref> Deref for Pin<P>`。它也是基础类型，因此我们可以为`Pin<Foo<T>>`实现 `Deref`，但前提是`Foo`是本地的（local-ish）且没有实现`Deref`，这与添加`Deref`实现所需的条件相同。所以我们最终需要的正是最初想要的：一个我们可以控制或自己编写`Deref`实现的类型，同时还实现了`CoerceUnsized`。这并没有帮助。

### 不算方法的方法 - 子类型

另一种更改`Pin`类型的方法是使用变体（variance），但这只允许你更改生命周期。

理论上，你可以有一个类型，其是否为`Unpin`取决于生命周期的条件，因此在更改生命周期后，你会得到一个指向`!Unpin`类型的`Pin`。

```rust
struct Foo<'a>(&'a ());
impl Unpin for Foo<'static> {}

fn test<'a>(p: Pin<&'a Foo<'a>>) {
    Pin::into_inner(p); // error: does not impl Unpin
}
fn main() {
    test(Pin::new(&Foo(&())));
}
```

但是我不觉得实际上有任何途径可以利用这个漏洞。

### 可能的修复方法

因为这个概念验证可以在稳定版上运行，所以要修复它本质上需要一个破坏性更改。

最直接的修复方法似乎是阻止用户为不可变引用实现`DerefMut`或为可变引用实现`Clone`。我认为可以通过在标准库中添加虚拟的blanket实现来实现这一点：

译注：blanket实现指的是一种泛型实现，它适用于所有满足某些条件的类型。例如，在Rust的标准库中，有一个`impl<T> Deref for T where T: SomeTrait`的blanket实现。这意味着只要一个类型`T`实现了`SomeTrait`，那么它就会自动获得`Deref`特性，而无需手动为`T`实现 `Deref`。这意味着我们可以添加一些blanket实现来阻止某些类型的不合法实现（为不可变引用实现 `DerefMut`，或者为可变引用实现 `Clone`）。

```rust
pub trait DummyTrait {}
pub struct DummyStruct<T>(T);
impl<'a, T> DerefMut for &'a T where DummyStruct<T>: DummyTrait { ... }
impl<'a, T> Clone for &'a mut T where DummyStruct<T>: DummyTrait { ... }
```

我想了解这是否会破坏任何实际代码。

当`CoerceUnsized`稳定后，该怎么办？我们如何在自己的类型上安全地实现`CoerceUnsized`？我能想到两种基本方法：

1. 我们可以要求`CoerceUnsized`与`Deref`和`DerefMut`保持“一致性”，也就是说，在类型转换前后调用`deref()`和`deref_mut()`应该返回相同的值。

不过，编译器可能很难强制执行这一点，尤其是考虑到我们甚至不要求`deref`和`deref_mut`是纯函数。我们可以将`CoerceUnsized`标记为不安全的，并将一致性检查的责任交给程序员，但如果它本质上并不需要不安全，这似乎不太理想。

或者，我们可以简单地将`Pin`的`CoerceUnsized`实现依赖到一个不安全的标记特性。因此，如果你定义了一个从`Foo`强制转换为`Bar`的结构体，你仍然无法将`Pin<Foo>`强制转换为`Pin<Bar>`，除非你为`Foo`实现了那个不安全的特性。

不过，很有可能我错过了一些更好的解决方案，尤其考虑到`CoerceUnsized`的稳定版本还没有完全设计好。
