---
title: C++20协程标准库，从入门到了解
categories: HPC
date: 2025/7/21 13:29:00
mathjax: true
---

# 前言

最近要重写一个Windows MediaFoundation的封装，其中涉及到一些异步场景，计划用C++20的Coroutine实现异步逻辑，因此我就写下这篇文章作为学习总结，顺便帮一些想学习C++协程的朋友少走弯路。另外也是觉得最近休息时间什么都不想做，秋招的高压之下确实需要一个锚点用来保持学习状态。

# 什么是协程

如果说**线程**是由操作系统在内核态管理的状态机，那么**协程**就是完全由开发者在用户态管理的状态机。这样的设计有以下好处：

1. **提高了开发者对任务调度、资源调度等调度模型的掌控能力**。更具体的，在任务调度模型方面，协程的挂起与恢复时机都可以由开发者完全控制，协程可以选用协作式的调度模型，主动出让CPU给事件循环或其他协程；在资源调度模型方面，协程可以使用更自适应的上下文大小，实现单机上百万的协程并发。
2. **减少操作系统调用的开销**。这其中不仅包括线程切换所需的系统调用，还包括同步原语所需的一些系统调用。

# 协程的历史

协程（Coroutine）一词，最早于1963年由Melvin Conway在他的操作系统著作中提出。

当时间来到1980年代，多线程设计兴起。在多线程设计中，用于保存线程状态的结构体（`task_struct`）与调度器由操作系统内部实现。操作系统对外提供标准化接口，开发者无须关注内部细节，就可以较为轻松的扩大并行规模。对比之下，协程的状态结构和调度器均需要编程语言或开发者自行实现。由于当时的系统调用与线程调度所消耗的性能相较缓慢的业务程序而言实在不值一提，协程相较多线程几乎没有优势。

直到千禧年后，随着计算机的提速与光纤技术在互联网通信中的铺开应用，在高性能网络程序中，系统调用与线程调度的耗时占比越来越多，开发者对框架灵活性的需求也越来越大。在2012年，微软向C#语言的核心特性中添加了async/await关键字，这标志着协程首次进入主流编程语言的视野。随后，2015年的Python3.5和JavaScript ES6分别添加了async/await和future/promise来支持协程开发。在10年后诞生的年轻语言，大多出厂便自带协程语法糖（如Rust/Kotlin）或是运行时设施（如Go）。2017年，承载着标准协程库使命的Coroutine TS被提交到C++标准委员会，并于2020年的C++20中正式发布为标准协程库<coroutine>。

在最近的20年间，业界主流的设计范式更是从操作系统大包大揽的重内核设计，逐渐向高自由度+低overhead的旁路内核设计转移。这种设计有如一把双刃剑，将更丰富的底层细节暴露给上游开发者，允许上游开发者执行更深入的优化，也相应地使得框架的学习曲线更陡峭，提高了工程的开发和维护成本。

# 协程的分类

C++中的标准协程库采用了**无栈**设计，同时兼容非对称与对称写法。当然，目前的C++协程框架中，大部分采用了非对称设计。

## 有栈与无栈

我们先对无栈协程与有栈协程做一下区分。有栈协程的工作方式与线程非常相似。众所周知，每个线程都有一个调用栈。函数调用时，操作系统会将返回地址与参数压入调用栈；函数返回时，再从栈上弹出返回地址，并跳转到对应地址处继续执行。有栈协程也有调用栈，这正是“有栈”名字的由来。只不过，有栈协程的调用栈由协程库在用户态申请，而线程的调用栈由操作系统在内核态申请。

顾名思义，“无栈”的意思就是无栈协程没有这个模拟出来的调用栈了。无栈协程的上下文状态需要比当前栈帧更长的生命周期，因此一般被存放在堆上。

上面讲了无栈协程与有栈协程在设计上的不同点。这种设计差异又会带来什么功能上的差异呢？有栈协程的优势包括：

1. 有栈协程可以在任意位置挂起，再转去执行抢占式调度器或是另一个有栈协程。换一种更精准的表述就是，有栈协程可以在任意时间将其**在CPU核上执行的权利**交还给抢占式调度器，或是转交给另一个有栈协程。这种切换和线程的切换方式非常类似，保存一下调用栈指针和寄存器等上下文信息就可以执行切换。在腾讯2013年开源的有栈协程库[libco](https://github.com/Tencent/libco)中就使用了这么一段[汇编代码](https://github.com/Tencent/libco/blob/master/coctx_swap.S)来实现上下文状态的保存与协程切换。而无栈协程只能在一个特定的标记点（一般由`await`或`yield`标记）挂起，并将执行权交还给主调度器。
2. 将业务代码迁移到有栈协程几乎不需要任何改动。而无栈协程往往需要在上游的每个需要挂起的地方，都添加上`await`标记点，用于辅助编译器或解释器生成状态机，这就是俗称的`async`/`await`污染。

相对的，有栈协程的劣势有以下几点：

1. 有栈协程的完整实现需要将操作系统中耗时较长的同步函数封装成异步的形式。
2. 在调试调用栈时，有栈协程需要自行设计栈回溯机制。这种第三方机制很难受到调试器的官方支持；而那些依托语言标准的无栈协程一般会有较好的调试器支持。
3. 有栈协程的调用栈大小难以预估，当函数嵌套调用过多时容易导致栈溢出，动态扩容的机制实现起来又较为复杂。
4. 有栈协程的切换过程需要保存大量上下文状态，切换耗时约为几十到几百纳秒，而无栈协程的切换普遍仅需几纳秒。

为何包括C++/C#/Python/Rust/R/Kotlin/Swift在内的大部分编程语言的标准协程库都是无栈协程？而大公司在生产环境应用的协程库，包括谷歌的goroutine、腾讯的tRPC在内，又有相当一部分是有栈协程？

大部分编程语言都希望尽可能避免引入额外的运行时开销。在有栈协程中，系统函数的异步封装方式、栈扩容方法、上下文保存方法都难以被标准化，因此这些编程语言才纷纷选择无栈协程的路线。而在实际工程中，企业最优先关注的往往是对原有业务的兼容性，其次是故障率，然后是扩展性与可维护性，最后才是非必要不考虑的性能优化。因此，可以与旧业务代码流畅兼容，更不易出bug的有栈协程自然更受青睐。

## 非对称与对称

非对称协程，意味着在每个挂起点，协程都要将**在CPU核上执行的权利**交还给主协程。协程一般需要实现一个`suspend`方法用于挂起，以及一个`resume`方法用于恢复执行。而对称协程可以将在CPU核上执行的权利交给另一个协程。协程之间的“地位”是对称的。此时协程一般需要实现一个`yield_to`或者`resume_on`方法用来指定接下来要运行的协程。

在有栈协程上实现对称协程较为安全。而在无栈协程上实现对称协程无异于允许goto乱飞，非常容易出bug。因此绝大多数无栈协程都是非对称协程。

通常，非对称协程中的主协程是一个事件循环（EventLoop）。事件循环会轮流恢复（resume）唤醒队列中的协程，以推动它们进一步执行。在此过程中，若协程执行完毕，事件循环会将其移除；若协程再次挂起，事件循环会将其从唤醒队列移入等待队列。当唤醒队列为空时，事件循环会在一个等待IO、计时器或信号事件的操作系统调用（如`epoll_wait`）上阻塞。当这个等待新事件的操作系统调用返回时，事件循环会将这些事件对应的所有协程从等待队列移入唤醒队列，并轮流恢复它们的执行。

# C++协程

本章节我们将从无栈协程中各类资源的生命周期入手，对无栈协程的实现细节建立初步理解；然后学习C++20协程的标准用法、最佳实践和时序图，初步掌握C++20协程的工程应用；最后剥开C++协程的语法糖，巩固对底层机制的理解。

协程是一个相对复杂的语言特性，开放给用户定制的功能点较多。因此，个人认为从资源管理出发的学习路线，虽然比从demo上手的路线更陡峭，但也更能避免在生命周期等疑难杂症上踩坑，更适合那些希望在复杂工程中应用C++20协程的同学。

推荐一个[B站视频](https://www.bilibili.com/video/BV1Cz9NYFE8E)，来自up主“程序员陈子青”。他的讲解通俗易懂，思路也是先从协程的资源管理出发，稍后再深入语言细节。本文受到了该视频的很多启发。

## 无栈协程的实现细节

下文将那些与某个无栈协程相关联的资源集合称作**协程帧**。无栈协程的本质是一个可以被多次挂起、恢复执行的状态机。这意味着，协程帧的生命周期必须独立于当前的函数调用栈帧，不能因为调用栈析构就将协程状态机一并送走。因此协程帧只能**动态分配**在堆上或其他具有较长生命周期的内存池中。协程帧中一般会保存以下信息：

1. 传入的参数。按值传入协程的参数均需要复制到协程帧内部，以保证在整个协程的生命周期内都可以访问入参。
2. 一些协程内使用的临时变量。只有那些跨越了挂起点（一般由await挂起）的临时变量才需要持久化状态，才需要被存入协程帧。
3. 挂起点的信息。用于确定下次协程恢复时需要从哪里恢复执行。
4. 调用的子协程的协程帧地址（可选，但大部分情况下需要）。大部分情况下，如果需要从调用的子协程中获取返回值，或是控制子协程的生命周期，比如在当前协程帧析构时将子协程的协程帧一并销毁，就必须保存子协程的协程帧的地址。协程帧以类似单向链表的形式串成一串。

接下来，我们将深入语言特性，学习一些C++20协程的用法。

## 标准库组件

为了简化场景，这里仅展开讨论`co_await`和`co_return`，先不讨论异步生成器相关的内容。

### `co_await`与Awaitable

await意为等待。`co_await`是一个C++关键字，用于等待右侧的Awaitable对象在未来的某一时刻返回结果，并挂起当前协程。

并不是所有的对象都能被放在`co_await`的右边。一个合格的Awaitable需要满足若干要求，先举一个例子：

```cpp
struct MyAwaitable {
    bool await_ready() const noexcept { return false; }
    void await_suspend(std::coroutine_handle<> handle) const { handle.resume(); }
    int await_resume() const noexcept { return 42; }
};
```

在这个例子中，我们自定义了一个符合Awaitable要求的`MyAwaitable`。

一个合格的Awaitable需要至少实现三个public成员函数：

1. `await_ready`：控制协程**是否需要挂起**。有时候，Awaitable可以立即返回结果。比如需要的资源已经进入缓存。那么`await_ready`就可以返回`true`，跳过挂起状态直接返回结果。这里我们返回的是`false`，意味着总是需要挂起。
2. `await_suspend`：控制协程**在挂起时的行为**。该函数会拿到一个至关重要的`std::coroutine_handle<> handle`参数。你可以将`std::coroutine_handle`理解为协程的“遥控器”，它指向当前协程的内部状态，可以被用于控制协程的恢复与销毁。因此不难看出，我们需要在这个函数内实现与主协程（或事件循环）的交互工作。虽然我们这里直接调用了`handle.resume()`来恢复协程，但一般来说，在`await_suspend`的函数体中，我们应该将`handle`关联到某个事件（event），再将这个事件追加到异步框架（如epoll等）的等待列表。待事件被触发时，在回调函数中，我们再从事件的`userdata`中取出`handle`，最后调用`handle.resume()`来恢复协程。
3. `await_resume`：控制协程**在恢复后的行为**。该函数的返回值会作为整个`co_await ...`表达式的返回值。这里我们直接返回了一个42作为返回值。需要特别注意的是，不论协程有没有被挂起过，这个`await_resume`始终都会被执行。

### `co_return`与`promise_type`

当一个函数体内出现了`co_await`或`co_return`，这个函数就自动成为了协程函数。从上面对`co_await`的介绍我们知道，`co_await`被用于在协程函数内异步地等待另一个函数的结果。`co_return`则被用来返回当前协程函数的最终结果，下面我们来看一个例子：

```cpp
MyTask example() {
    int result = co_await MyAwaitable{};
    co_return result;
}
```

在协程函数`example`中，我们先使用`co_await MyAwaitable{}`表达式异步地获得了一个值`int result`，然后使用`co_return`语句将`result`返回给上层协程函数。

协程函数的返回类型并不是`result`的类型。而是一个自定义类型，其中**必须**包含一个内嵌类型`promise_type`。这个内嵌的`promise_type`可以包含一些特定的成员函数，用于定制协程函数的调度行为。上面提到的`MyTask`的详细定义如下：

```cpp
struct MyTask {
    struct promise_type {
        MyTask get_return_object() { return MyTask{std::coroutine_handle<promise_type>::from_promise(*this)}; }
        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_value(int v) {}
        void unhandled_exception() {}
    };

    std::coroutine_handle<promise_type> handle_;

    MyTask(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~MyTask() {
        if (handle_) handle_.destroy();
    }
};
```

C++20标准协程学习曲线陡峭，语法复杂度饱受诟病，很大程度上是因为这个`promise_type`嵌套在`MyTask`中的结构。我认为，要理解这个结构的设计初衷，就必须结合协程帧在内存中的布局以及时序图来学习。

协程帧的内存布局，我们在前文已经有了一些铺垫，回顾一下就是多个协程帧按调用顺序，以类似单向链表的形式串在一起。下面我们按照触发的时间顺序来看看，协程的执行过程会经历哪些步骤，以及`promise_type`中的这些成员函数分别在其中实现了哪些功能。协程的执行过程会经历以下步骤：

1. 为协程帧申请内存空间。
2. 将入参拷贝到协程帧内。
3. 在当前协程帧上`promise_type`所在的位置调用其构造函数
4. 上层调用方的协程帧的临时变量区内，会给返回值`MyTask`预留一段空间。用户定义的`get_return_object`成员函数的返回值，将会被用于初始化这个调用方协程帧上的`MyTask`。也就是说，嵌套在`MyTask`中的`promise_type`的`get_return_object`成员函数，必须返回一个`MyTask`对象。并且用户可以在这个`get_return_object`中自定义初始化逻辑。大部分协程库都会给`MyTask`传入一个`std::coroutine_handle<promise_type>::from_promise(*this)`。注意到`promise_type`位于当前协程帧内，因此通过`std::coroutine_handle<promise_type>::from_promise(*this)`我们就能拿到指向当前协程帧的`std::coroutine_handle`。再将这个`std::coroutine_handle`传递给上层调用方，就能让上层调用方通过这个handle获取当前协程的返回值，或是控制当前协程的生命周期。
5. 调用`initial_suspend`得到一个Awaitable，并等待这个Awaitable执行完毕。一般我们会返回一个`std::suspend_always`，说明协程立即挂起，将`std::suspend_never`，说明协程；或者

协程调用的时序图如下图所示：

TODO

## 剥语法糖

```cpp
#include <coroutine>

struct MyAwaitable {
    bool await_ready() const noexcept { return false; }
    void await_suspend(std::coroutine_handle<> handle) const { handle.resume(); }
    int await_resume() const noexcept { return 42; }
};

struct MyTask {
    struct promise_type {
        MyTask get_return_object() { return MyTask{std::coroutine_handle<promise_type>::from_promise(*this)}; }
        std::suspend_never initial_suspend() noexcept { return {}; }
        std::suspend_never final_suspend() noexcept { return {}; }
        void return_value(int v) {}
        void unhandled_exception() {}
    };

    std::coroutine_handle<promise_type> handle_;

    MyTask(std::coroutine_handle<promise_type> h) : handle_(h) {}
    ~MyTask() {
        if (handle_) handle_.destroy();
    }
};

MyTask example() {
    int result = co_await MyAwaitable{};
    co_return result;
}

int main() { example(); }
```
