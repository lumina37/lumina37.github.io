---
title: C++20协程标准库，从入门到了解
categories: HPC
date: 2025/7/21 13:29:00
mathjax: true
---

# 前言

最近我要重写一个Windows MediaFoundation的封装，其中涉及到一些异步场景，计划用C++20的Coroutine实现异步逻辑，因此写下这篇文章作为学习总结，顺便帮一些想学习C++协程的朋友少走弯路。另外也是觉得最近休息时间什么都不想做，秋招的高压之下确实需要一个锚点用来保持学习状态。

# 什么是协程

**协程**，就是一个可以被挂起（suspend，与暂停意思相近）和恢复的函数。

这个定义可能不好理解。但其实**协程和线程在逻辑上几乎就是一个东西**。当线程长时间占用CPU时，操作系统可能会将其挂起，转而恢复其他优先级更高的线程的执行；当线程尝试一个自旋锁达到特定失败次数时，可能会主动让出CPU，恢复其他线程的执行流程；类似地，当协程遇到一个需要较长IO时间的操作时，也可能会主动让出CPU，转而去恢复其他协程的运行。

线程和协程都是状态机。对于一个可以被挂起和恢复的状态机，它**必须**在内部状态中记录一条“刚刚在何处挂起，接下来应在何处恢复”的关键信息。线程和协程的本质区别在于——如果说**线程**是由操作系统在内核态管理的状态机，那么**协程**就是完全由开发者在用户态管理的状态机。

这样的设计有以下好处：

1. **提高了开发者对任务调度、资源调度等调度模型的掌控能力**。更具体的，在任务调度模型方面，协程的挂起与恢复时机都可以由开发者完全控制，协程可以选用协作式的调度模型，主动出让CPU给事件循环或其他协程；在资源调度模型方面，协程可以使用更自适应的上下文大小，实现单机上百万的协程并发。
2. **减少操作系统调用的开销**。这其中不仅包括线程切换所需的系统调用，还包括同步原语所需的一些系统调用。

特别记住，所有的协程模型都需要在某个地方存放这样一条有关**挂起恢复点**的关键信息。我们可以通过理解这个挂起恢复点的**存储形式**和**变更方式**，快速地理解一个协程模型的本质，不论它是C++20协程还是goroutine。

# 协程的历史

## 起源

协程（Coroutine）一词，最早于上世纪60年代初，由Melvin Conway在他的操作系统著作中提出。

当时间来到80年代，多线程设计兴起。在多线程设计中，用于保存线程状态的结构体（`task_struct`）与调度器由操作系统内部实现。操作系统对外提供标准化接口，开发者无须关注内部细节，就可以较为轻松的扩大并行规模。线程之间通过回调函数来接收异步结果。对比之下，协程的状态结构和调度器均需要编程语言或开发者自行实现。由于当时的系统调用与线程调度所消耗的性能相较缓慢的业务程序而言实在不值一提，回调函数也足以应付较小的程序规模，协程相较多线程几乎没有优势。

## 回调地狱

直到进入21世纪，随着计算机的提速与普及化，加之光纤技术在互联网通信中的铺开应用，应用程序的体量不断膨胀。在追求高性能的大体量应用程序中，系统调用与线程调度的耗时占比越来越多。开发者对于异步框架的灵活性和易用性的需求也越来越大。

在当时，程序普遍通过回调函数来接收异步结果。当需要在回调函数中再调用其他回调函数，实现类似A→B→C的嵌套调用时，开发者往往不得不写出极其丑陋的代码：

```js
loadData((err, data) => {
    if (err) {
        console.error('Failed to load data:', err);
        return;
    }
    processData(data, (err, processedData) => {
        if (err) {
            console.error('Failed to process data:', err);
            return;
        }
        saveData(processedData, (err, savedData) => {
            if (err) {
                console.error('Failed to save data:', err);
                return;
            }
            console.log('Data flow complete:', savedData);
        });
    });
});
```

这种回调函数组成的异步代码，往往难以阅读，难以维护，难以在调试器中通过调用栈进行debug，更是极易因为回调函数中需要用到的资源已不慎被提前释放，导致发生各种难以排查的bug。

而在协程中，我们可以使用类似同步调用的写法实现异步功能。只需要用类似`await`的关键字标记出协程挂起点，剩下的大部分调度及资源管理工作都可以由协程框架自动完成，天生具备极佳的可读性。

至此，为了解决回调函数模式导致的种种问题，逐渐开始有开发者将目光投向协程。

## 逐渐伟大

在最近的20年间，业界主流的设计范式从操作系统大包大揽的重内核设计，逐渐向高自由度+低overhead的旁路内核设计转移。这种设计有如一把双刃剑，将更丰富的底层细节暴露给上游开发者，允许上游开发者执行更深入的优化，也相应地使得框架的学习曲线更陡峭，提高了工程的开发和维护成本。

对于各家编程语言的协程规范制定者以及协程库作者来说，协程正是这样一类高度复杂的语言功能。这里我们暂且不展开过多细节。关于无栈协程的复杂性，相信各位在后面对C++20协程的学习过程中就会有所体会。

经历各种曲折探索后，直到2012年，微软才向C#语言的核心特性中添加了async/await关键字，这标志着协程首次进入主流编程语言的视野。随后，2015年的Python3.5和JavaScript ES6分别添加了async/await和future/promise来支持协程开发。不过，在2010年后诞生的年轻语言，大部分都非常重视对协程的原生支持，出厂便自带了协程语法糖（如Rust/Kotlin）或是运行时设施（如Go）。2017年，承载着标准协程库使命的Coroutine TS被提交到C++标准委员会，并于2020年的C++20中正式发布为标准协程库<coroutine>。

# 协程的分类

C++中的标准协程库采用了**无栈**设计，同时兼容非对称与对称写法。当然，目前的C++协程框架中，大部分都采用了非对称设计。

## 有栈与无栈

首先，对无栈协程与有栈协程做一下区分。有栈协程的工作方式与线程非常相似。众所周知，所有线程都会有一个调用栈。当函数调用时，操作系统会将返回地址与参数压入调用栈；函数返回时，再从栈上弹出返回地址，并跳转到对应地址处继续执行。有栈协程也有一个调用栈，这正是“有栈”名字的由来。只不过，有栈协程的调用栈由协程库在用户态管理，而线程的调用栈由操作系统在内核态管理。

相对的，“无栈”的意思，顾名思义就是无栈协程没有这个模拟出来的调用栈。这意味着无栈协程的上下文状态往往需要一个比当前栈帧更长的生命周期，因此无栈协程的状态一般被存放在堆上。

上面讲了无栈协程与有栈协程在设计上的不同点。下面再看看这种设计差异又会带来什么功能上的差异。有栈协程的优势包括：

1. 有栈协程可以在任意位置挂起，再转去执行抢占式调度器或是另一个有栈协程。换一种更精准的表述就是，有栈协程可以在任意时间将其**在CPU核上执行的权利**交还给抢占式调度器，或是转交给另一个有栈协程。这种切换和线程的切换方式非常类似，保存一下调用栈指针和寄存器等上下文信息就可以执行切换。在腾讯2013年开源的有栈协程库[libco](https://github.com/Tencent/libco)中就使用了这么一段[汇编代码](https://github.com/Tencent/libco/blob/master/coctx_swap.S)来实现上下文状态的保存与协程切换。而无栈协程只能在一个特定的标记点（一般由`await`或`yield`标记）挂起，并将执行权交还给主调度器。
2. 将业务代码迁移到有栈协程几乎不需要任何改动。而无栈协程往往需要在上游的每个需要挂起的地方，都添加上`await`标记点，用于辅助编译器或解释器生成状态机，这就是俗称的`async`/`await`污染。

相对的，有栈协程的劣势有以下几点：

1. 有栈协程的完整实现需要将操作系统中耗时较长的同步函数封装成异步的形式。
2. 在调试调用栈时，有栈协程需要自行设计栈回溯机制。这种第三方机制很难受到调试器的官方支持；而那些依托语言标准的无栈协程一般会有较好的调试器支持。
3. 有栈协程的调用栈大小难以预估，当函数嵌套调用过多时容易导致栈溢出，动态扩容的机制实现起来又较为复杂。
4. 有栈协程的切换过程需要保存大量上下文状态，切换耗时约为几十到几百纳秒，而无栈协程的切换普遍仅需几纳秒。

为何包括C++/C#/Python/Rust/R/Kotlin/Swift在内的大部分编程语言的标准协程库都是无栈协程？而大公司在生产环境应用的协程库，包括谷歌的goroutine、腾讯的tRPC在内，又有相当一部分是有栈协程？

这是因为，大部分编程语言都希望尽可能避免引入额外的运行时开销；并且，在有栈协程中，系统函数的异步封装方式、栈扩容方法、上下文保存方法都难以被标准化。因此这些编程语言标准才纷纷选择无栈协程的路线。而在实际工程中，企业最优先关注的往往是对原有业务的兼容性，其次是故障率，然后是扩展性与可维护性，最后才是非必要不考虑的性能优化。因此，可以与旧业务代码流畅兼容，更不易出bug的有栈协程自然更受青睐。

## 非对称与对称

非对称协程，意味着在每个挂起点，协程都要将**在CPU核上执行的权利**交还给主协程。协程一般需要有一个`suspend`方法用于挂起，以及一个`resume`方法用于恢复执行。而对称协程可以将在CPU核上执行的权利交给另一个协程。协程之间的“地位”是对称的。此时协程一般需要有一个`yield_to`或者`resume_on`方法用来指定接下来要运行的协程。

在有栈协程上实现对称协程较为安全。而在无栈协程上实现对称协程，无异于允许goto乱飞，非常容易出bug。因此绝大多数无栈协程都是非对称协程。

通常，非对称协程中的主协程是一个事件循环（EventLoop）。事件循环会轮流恢复（resume）唤醒队列中的协程，以推动它们进一步执行。在此过程中，若协程执行完毕，事件循环会将其移除；若协程再次挂起，事件循环会将其从唤醒队列移入等待队列。当唤醒队列为空时，事件循环会在一个等待IO、计时器或信号事件的操作系统调用（如`epoll_wait`）上阻塞。当这个等待新事件的操作系统调用返回时，事件循环会将这些事件对应的所有协程从等待队列移入唤醒队列，并轮流恢复它们的执行。

# C++协程

本章节，我们将从无栈协程中各类资源的生命周期入手，对无栈协程的实现细节建立初步理解；随后学习C++20协程的标准用法和时序图，掌握C++20协程的基础语法；然后脱掉C++协程的语法糖，巩固对底层机制的理解；最后阅读一些知名开源协程库的源码，了解行业内的最佳实践，初步掌握C++20协程的工程化应用。

C++20协程是一个上手难度较高的语言特性，它开放给用户定制的功能点非常多，市面上有关其最优实践的免费教程更是几乎没有。个人认为，从资源管理出发的学习路线虽然比从demo直接上手的路线更陡峭，但也更能避免在生命周期等疑难杂症上踩坑。最后的源码阅读与最优实践章节，更能帮助那些希望在复杂工程中应用C++20协程的同学尽快上手。

推荐一个[B站视频](https://www.bilibili.com/video/BV1Cz9NYFE8E)，来自up主“程序员陈子青”。他的讲解通俗易懂，思路也是先从协程的资源管理出发，稍后再深入语言细节。本文受到了该视频的很多启发。

## 无栈协程的实现细节

无栈协程的本质是一个可以被多次挂起、恢复执行的状态机。而**协程帧**中保存了一个无栈协程的所有状态。这意味着，协程帧的生命周期必须独立于当前的函数调用栈帧，不能因为调用栈析构，就将协程状态一并析构。因此协程帧**必须**动态分配在堆上或其他具有较长生命周期的内存池中。特别留意**协程帧**这个术语，下文会反复使用。

协程帧中一般会保存以下信息：

1. 传入的参数。按值传入协程的参数全都需要复制到协程帧内部，以保证在整个协程的生命周期内都可以访问入参。按引用传递的参数则保持原样，由用户负责保证引用的生命周期。
2. 一些协程内使用的临时变量。只有那些跨越了协程挂起点（一般由`co_await`挂起）的临时变量才需要持久化状态，才需要被存入协程帧。
3. **挂起点的信息**。也就是上文提到的“当前协程刚刚在何处挂起，接下来应在何处恢复”的关键信息，用于确定下次协程恢复时需要从哪里恢复执行。
4. 上下级协程的协程帧地址（可选，但大部分情况下需要）。大部分情况下，如果需要从调用的下级协程中获取返回值，或是控制下级协程的生命周期，比如在当前协程帧析构时将下级协程的协程帧一并销毁，就必须保存下级协程的协程帧地址；如果要在当前协程结束后，恢复上级协程的执行，那么当前协程帧内也必须保存上级协程的协程帧地址。这样一来，协程帧就会以类似双向链表的形式串成一串。需要注意的是，这里保存上下级协程的协程帧地址的逻辑需要开发者自行实现，编译器不会代劳。

接下来，我们将深入语言特性，学习一些C++20协程的用法。

## 标准库设施（基础）

为了简化场景，这里仅展开讨论`co_await`和`co_return`，先不讨论异步生成器相关的内容。

### `co_await`与Awaitable

await意为等待，那么Awaitable就是“可等待的东西”。`co_await`是一个用于“等待”的C++关键字。它的出现意味着当前协程需要等待`co_await`右侧的Awaitable对象在未来的某一时刻返回结果。在等待时，当前协程可能需要挂起。

并不是所有的对象都能被放在`co_await`的右边。一个合格的Awaitable需要满足若干要求，先举一个例子：

```cpp
struct MyAwaitable {
    bool await_ready() const noexcept { return false; }
    void await_suspend(std::coroutine_handle<> handle) const { handle.resume(); }
    int await_resume() const noexcept { return 42; }
};
```

在这个例子中，我们定义了一个符合要求的`MyAwaitable`。

一个合格的Awaitable需要至少实现三个public成员函数：

1. `await_ready`：控制协程**是否需要挂起**。有时候，Awaitable可以立即返回结果。比如需要的资源已经进入缓存。那么`await_ready`就可以返回`true`，跳过挂起状态直接返回结果。这里我们返回的是`false`，意味着总是需要挂起。
2. `await_suspend`：控制协程**在挂起时的行为**。该函数会拿到一个至关重要的`std::coroutine_handle<> handle`参数。你可以将`std::coroutine_handle`理解为协程的“遥控器”，它指向当前协程的内部状态，可以被用于控制协程的恢复与销毁。因此不难看出，我们需要在这个函数内实现与主协程（或事件循环）的交互工作。虽然我们这里直接调用了`handle.resume()`来恢复协程，但一般来说，在`await_suspend`的函数体中，我们应该将`handle`关联到某个事件（event），再将这个事件追加到异步框架（如epoll等）的等待列表。待事件被触发时，在事件的回调函数中，我们可以从事件的`userdata`中取出`handle`，再调用`handle.resume()`来恢复协程。
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

协程函数的返回类型并不是`result`的类型`int`，而是一个自定义类型。这个自定义类型中**必须**包含一个内嵌类型`promise_type`。这个内嵌的`promise_type`可以包含一些特定的成员函数，用于定制协程函数的调度行为。上面提到的`MyTask`的详细定义如下：

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

C++20标准协程难以上手，很大程度上是因为我们需要定义这个`promise_type`嵌套在`MyTask`中的结构。要理解这个结构的设计思路，最好结合协程帧在内存中的布局以及时序图来学习。

`promise_type`的存储位置位于协程帧内。关于协程帧的内存布局，我们在前文已经有了一些铺垫。回顾一下就是多个协程帧按调用顺序，以类似双向链表的形式串在一起。同时，大部分情况下，上下级协程的handle都会保存在当前协程帧的`promise_type`中。只不过这个保存逻辑需要由开发者在他们定制的`promise_type`中实现，编译器不会代劳。

下面我们按照触发的时间顺序来看看，协程的执行过程会经历哪些步骤，以及`promise_type`中的这些成员函数分别在其中实现了哪些功能。协程的执行过程会经历以下步骤：

1. 为协程帧申请内存空间。
2. 将入参拷贝到协程帧内。
3. 在当前协程帧上`promise_type`所在的位置调用其构造函数。
4. 上层调用方的协程帧的临时变量区内，会给返回值`MyTask`预留一段空间。用户定义的`get_return_object`成员函数的返回值，将会被用于初始化这个调用方协程帧上的`MyTask`。也就是说，嵌套在`MyTask`中的`promise_type`的`get_return_object`成员函数，必须返回一个`MyTask`对象。并且用户可以在这个`get_return_object`中自定义初始化逻辑。大部分协程库都会给`MyTask`传入一个`std::coroutine_handle<promise_type>::from_promise(*this)`。注意到`promise_type`位于当前协程帧内，因此通过`std::coroutine_handle<promise_type>::from_promise(*this)`我们就能拿到指向当前协程帧的`std::coroutine_handle`。再将这个`std::coroutine_handle`传递给上层调用方，就能让上层调用方通过这个handle获取当前协程的返回值，或是控制当前协程的生命周期。
5. 调用`initial_suspend`获取一个Awaitable，并等待这个Awaitable执行完毕。一般我们会返回一个`std::suspend_always`，说明协程将立即挂起（懒惰模式），并将执行权交还给主调度器；或是返回一个`std::suspend_never`，说明协程将立即开始执行（饥饿模式），直到遇到`co_await`语句时再挂起。
6. （可选）如果发生了未捕获的异常，则在捕获异常后，在catch块内调用`unhandled_exception`。
7. 到达`co_return`。如果`co_return`没有返回值，那么`return_void`将被调用；否则，如果`co_return`返回了值，那么`return_value`将被调用，传入的参数就是`co_return`返回的值。这个`return_value`的意义就是给当前协程帧一个机会来保存返回值。
8. 析构协程中那些没有跨越挂起点的临时变量。跨越了挂起点的临时变量会被存放在协程帧上，跟随协程帧一起析构。
9. 调用`final_suspend`获取一个Awaitable，并等待这个Awaitable执行完毕。一般我们会返回一个特殊的Awaitable。这个特殊的Awaitable中保存了上层协程的handle，用于恢复上层协程的执行。而这个上层协程的handle的来源，正是当前协程的协程帧的`promise_type`中保存的那个上层协程handle。
10. 调用`promise_type`的析构函数。
11. 调用各个协程入参的析构函数。
12. 释放协程帧的内存空间。
13. 将执行权返还给主调度器。

### `std::suspend_*`

上面提到过的`std::suspend_always`和`std::suspend_never`是标准库中定义的两类特殊Awaitable，在`promise_type::initial_suspend`的返回类型处很常见。

其中`std::suspend_always`的定义如下：

```cpp
class suspend_always {
    constexpr bool await_ready() const noexcept { return true; }
    constexpr void await_suspend( std::coroutine_handle<> ) const noexcept {}
    constexpr void await_resume() const noexcept {}
};
```

其中`await_ready`始终返回`true`，意味着总是需要挂起。而`await_suspend`和`await_resume`均为空操作。

`std::suspend_never`与`std::suspend_always`类似，`await_suspend`和`await_resume`亦为空操作，而`await_ready`始终返回`false`，意味着不需要挂起。

### `std::coroutine_handle`

你可能注意到了`await_suspend`中传入的参数是`std::coroutine_handle<void>`类型，而`MyTask`中保存的成员变量类型是`std::coroutine_handle<promise_type>`类型。这两种模板实例化的区别在于，`std::coroutine_handle<void>`是`std::coroutine_handle<Promise>`在类型擦除后的泛化类型，擦除了`promise_type`相关的信息。这个设计是为了方便其他函数在看不到`promise_type`定义的情况下，依然能透过`std::coroutine_handle<void>`操纵协程。

任何`std::coroutine_handle<promise_type>`都可以被静态转换为`std::coroutine_handle<void>`。

通过`std::coroutine_handle<void>`我们依然能操作：

1. `bool is_done = h.done();` - 检查协程是否完成
2. `h.resume();` - 恢复协程执行
3. `h.destroy();` - 销毁协程
4. `bool valid = static_cast<bool>(h);` - 检查handle是否仍有效
5. `void* ptr = h.address();` - 导出协程帧的地址
6. `auto h = std::coroutine_handle<>::from_address(ptr);` - 将一个协程帧的地址导入为handle

但通过`std::coroutine_handle<void>`不能执行与具体的`promise_type`类型相关的操作，否则会发生编译失败：

1. `promise_type p = h.promise();` - 获取协程帧上的`promise_type`对象
2. `auto h = std::coroutine_handle<>::from_promise(p);` - 从`promise_type`对象的地址反推handle的值

### 时序图

协程调用的时序图如下图所示：

<img src="https://cdn.jsdelivr.net/gh/lumina37/picx-images-hosting@master/2507_blog_coroutine/00-timeline.webp" alt="协程时间轴（来自B站视频BV1Cz9NYFE8E）"/>

## 脱语法糖

下面，我们将针对下面这个简单demo，使用[C++ Insights](https://cppinsights.io/)来获取近似脱去协程语法糖后的代码，以此巩固对底层机制的理解。

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

<details>
<summary>近似脱去语法糖后可以得到（点击展开代码块）</summary>

```cpp
#include <coroutine>

struct MyAwaitable {
    inline bool await_ready() const noexcept { return false; }

    inline void await_suspend(std::coroutine_handle<void> handle) const { handle.resume(); }

    inline int await_resume() const noexcept { return 42; }
};

struct MyTask {
    struct promise_type {
        inline MyTask get_return_object() { return MyTask{std::coroutine_handle<promise_type>::from_promise(*this)}; }

        inline std::suspend_never initial_suspend() noexcept { return {}; }

        inline std::suspend_never final_suspend() noexcept { return {}; }

        inline void return_value(int v) {}

        inline void unhandled_exception() {}

        // inline constexpr promise_type() noexcept = default;
    };

    std::coroutine_handle<promise_type> handle_;
    inline MyTask(std::coroutine_handle<promise_type> h) : handle_{std::coroutine_handle<promise_type>(h)} {}

    inline ~MyTask() noexcept {
        if (this->handle_.operator bool()) {
            this->handle_.destroy();
        }
    }
};

struct __exampleFrame {
    void (*resume_fn)(__exampleFrame *);
    void (*destroy_fn)(__exampleFrame *);
    std::__coroutine_traits_impl<MyTask>::promise_type __promise;
    int __suspend_index;
    bool __initial_await_suspend_called;
    int result;
    std::suspend_never __suspend_26_8;
    MyAwaitable __suspend_27_27;
    int __suspend_27_27_res;
    std::suspend_never __suspend_26_8_1;
};

MyTask example() {
    /* Allocate the frame including the promise */
    /* Note: The actual parameter new is __builtin_coro_size */
    __exampleFrame *__f = reinterpret_cast<__exampleFrame *>(operator new(sizeof(__exampleFrame)));
    __f->__suspend_index = 0;
    __f->__initial_await_suspend_called = false;

    /* Construct the promise. */
    new (&__f->__promise) std::__coroutine_traits_impl<MyTask>::promise_type{};

    /* Forward declare the resume and destroy function. */
    void __exampleResume(__exampleFrame * __f);
    void __exampleDestroy(__exampleFrame * __f);

    /* Assign the resume and destroy function pointers. */
    __f->resume_fn = &__exampleResume;
    __f->destroy_fn = &__exampleDestroy;

    /* Call the made up function with the coroutine body for initial suspend.
       This function will be called subsequently by coroutine_handle<>::resume()
       which calls __builtin_coro_resume(__handle_) */
    __exampleResume(__f);

    return __f->__promise.get_return_object();
}

/* This function invoked by coroutine_handle<>::resume() */
void __exampleResume(__exampleFrame *__f) {
    try {
        /* Create a switch to get to the correct resume point */
        switch (__f->__suspend_index) {
            case 0:
                break;
            case 1:
                goto __resume_example_1;
            case 2:
                goto __resume_example_2;
            case 3:
                goto __resume_example_3;
        }

        /* co_await insights.cpp:26 */
        __f->__suspend_26_8 = __f->__promise.initial_suspend();
        if (!__f->__suspend_26_8.await_ready()) {
            __f->__suspend_26_8.await_suspend(
                std::coroutine_handle<MyTask::promise_type>::from_address(static_cast<void *>(__f))
                    .operator std::coroutine_handle<void>());
            __f->__suspend_index = 1;
            __f->__initial_await_suspend_called = true;
            return;
        }

    __resume_example_1:
        __f->__suspend_26_8.await_resume();

        /* co_await insights.cpp:27 */
        __f->__suspend_27_27 = MyAwaitable{};
        if (!__f->__suspend_27_27.await_ready()) {
            __f->__suspend_27_27.await_suspend(
                std::coroutine_handle<MyTask::promise_type>::from_address(static_cast<void *>(__f))
                    .operator std::coroutine_handle<void>());
            __f->__suspend_index = 2;
            return;
        }

    __resume_example_2:
        __f->__suspend_27_27_res = __f->__suspend_27_27.await_resume();
        __f->result = __f->__suspend_27_27_res;
        /* co_return insights.cpp:28 */
        __f->__promise.return_value(__f->result);
        goto __final_suspend;
    } catch (...) {
        if (!__f->__initial_await_suspend_called) {
            throw;
        }

        __f->__promise.unhandled_exception();
    }

__final_suspend:

    /* co_await insights.cpp:26 */
    __f->__suspend_26_8_1 = __f->__promise.final_suspend();
    if (!__f->__suspend_26_8_1.await_ready()) {
        __f->__suspend_26_8_1.await_suspend(
            std::coroutine_handle<MyTask::promise_type>::from_address(static_cast<void *>(__f))
                .operator std::coroutine_handle<void>());
        __f->__suspend_index = 3;
        return;
    }

__resume_example_3:
    __f->destroy_fn(__f);
}

/* This function invoked by coroutine_handle<>::destroy() */
void __exampleDestroy(__exampleFrame *__f) {
    /* destroy all variables with dtors */
    __f->~__exampleFrame();
    /* Deallocating the coroutine frame */
    /* Note: The actual argument to delete is __builtin_coro_frame with the promise as parameter */
    operator delete(static_cast<void *>(__f), sizeof(__exampleFrame));
}

int main() {
    example();
    return 0;
}
```
</details>


来分段看一下脱糖后的代码。前两段都是我们自定义的`MyAwaitable`和`MyTask`的定义。

下面这一段代码展示了`example`协程函数的协程帧定义。注释中标注了各个字段的含义。

```cpp
struct __exampleFrame {
    void (*resume_fn)(__exampleFrame *);  // 协程恢复执行时将调用的“回调函数”
    void (*destroy_fn)(__exampleFrame *);  // 协程销毁时将调用的“回调函数”
    // `promise_type`对象
    // 其中`std::__coroutine_traits_impl<MyTask>::promise_type`用于
    // 萃取`MyTask`中嵌套的`promise_type`类型
    std::__coroutine_traits_impl<MyTask>::promise_type __promise;
    int __suspend_index;  // 从何处挂起
    bool __initial_await_suspend_called;  // `initial_suspend`是否已被调用
    int result;  // 临时变量`result`
    std::suspend_never __suspend_26_8;  // `initial_suspend`返回的Awaitable
    MyAwaitable __suspend_27_27;  // `MyAwaitable`对象
    int __suspend_27_27_res;  // `MyAwaitable`的`await_resume`的返回值
    std::suspend_never __suspend_26_8_1;  // `final_suspend`返回的Awaitable
};
```

`example`函数中都是一些初始化的工作。除了第一次对`resume_fn`的调用需要解释一下意义——是为了间接执行`init_suspend`的逻辑。

```cpp
MyTask example() {
    /* 为协程帧申请内存空间 */
    __exampleFrame *__f = reinterpret_cast<__exampleFrame *>(operator new(sizeof(__exampleFrame)));
    __f->__suspend_index = 0;
    __f->__initial_await_suspend_called = false;

    /* placement new构造`promise_type` */
    new (&__f->__promise) std::__coroutine_traits_impl<MyTask>::promise_type{};

    /* 恢复和销毁回调函数的前向声明 */
    void __exampleResume(__exampleFrame * __f);
    void __exampleDestroy(__exampleFrame * __f);

    /* 初始化回调函数指针 */
    __f->resume_fn = &__exampleResume;
    __f->destroy_fn = &__exampleDestroy;

    /* 通过调用一次resume来间接执行`initial_suspend`逻辑 */
    __exampleResume(__f);

    /* 创建返回值 */
    return __f->__promise.get_return_object();
}
```

`__exampleResume`中实现了状态机在各阶段的逻辑，揭示了C++20无栈协程的核心机制。调度器会通过反复执行`__exampleResume`来推动协程状态的变化，直到协程执行完毕。

```cpp
void __exampleResume(__exampleFrame *__f) {
    try {
        /* Create a switch to get to the correct resume point */
        switch (__f->__suspend_index) {
            case 0:
                break;
            case 1:
                goto __resume_example_1;
            case 2:
                goto __resume_example_2;
            case 3:
                goto __resume_example_3;
        }

        /* co_await insights.cpp:26 */
        // 在这里执行`initial_suspend`的逻辑
        // `initial_suspend`中的异常会被下方的catch捕获
        // 但不会交由`unhandled_exception`处理
        __f->__suspend_26_8 = __f->__promise.initial_suspend();
        // 如果`initial_suspend`返回的Awaitable不需要挂起（比如`std::suspend_never`）
        // 那么直接跳到`__resume_example_1`处继续执行
        if (!__f->__suspend_26_8.await_ready()) {
            __f->__suspend_26_8.await_suspend(
                std::coroutine_handle<MyTask::promise_type>::from_address(static_cast<void *>(__f))
                    .operator std::coroutine_handle<void>());
            // 如果需要挂起，则将`__suspend_index`向前步进一位
            // 下次恢复时将直接从`__resume_example_1`开始执行
            __f->__suspend_index = 1;
            __f->__initial_await_suspend_called = true;
            // 返回，等待下一次恢复
            return;
        }

    __resume_example_1:
        // 先执行`initial_suspend`返回的Awaitable的`await_resume`
        __f->__suspend_26_8.await_resume();

        /* co_await insights.cpp:27 */
        __f->__suspend_27_27 = MyAwaitable{};
        // 如果`MyAwaitable{}`不需要挂起
        // 那么直接跳到`__resume_example_2`处继续执行
        if (!__f->__suspend_27_27.await_ready()) {
            __f->__suspend_27_27.await_suspend(
                std::coroutine_handle<MyTask::promise_type>::from_address(static_cast<void *>(__f))
                    .operator std::coroutine_handle<void>());
            // 如果需要挂起，同样将`__suspend_index`向前步进一位
            // 下次恢复时将直接从`__resume_example_2`开始执行
            __f->__suspend_index = 2;
            return;
        }

    __resume_example_2:
        // 先执行`MyAwaitable{}`的`await_resume`拿到返回值
        __f->__suspend_27_27_res = __f->__suspend_27_27.await_resume();
        // 返回值再赋给临时变量
        __f->result = __f->__suspend_27_27_res;
        /* co_return insights.cpp:28 */
        // 到达`co_return`处，调用`return_value`
        __f->__promise.return_value(__f->result);
        // 离开try块后，所有未跨越挂起点的临时变量都将被析构
        goto __final_suspend;
    } catch (...) {
        if (!__f->__initial_await_suspend_called) {
            // 不处理`initial_suspend`中的异常
            throw;
        }

        __f->__promise.unhandled_exception();
    }

__final_suspend:
    /* co_await insights.cpp:26 */
    __f->__suspend_26_8_1 = __f->__promise.final_suspend();
    // 如果`final_suspend`返回的Awaitable不需要挂起（比如`std::suspend_never`）
    // 那么直接跳到`__resume_example_3`处开始销毁协程帧
    if (!__f->__suspend_26_8_1.await_ready()) {
        __f->__suspend_26_8_1.await_suspend(
            std::coroutine_handle<MyTask::promise_type>::from_address(static_cast<void *>(__f))
                .operator std::coroutine_handle<void>());
        // 如果需要挂起，同样将`__suspend_index`向前步进一位
        // 下次恢复时将直接从`__resume_example_3`开始销毁协程帧
        __f->__suspend_index = 3;
        return;
    }

__resume_example_3:
    // 销毁协程帧
    __f->destroy_fn(__f);
}
```

## 检验学习效果

到此为止，我们应该已经可以解答下面的一些问题：

1. 我们为什么要使用协程？
2. 无栈协程有哪些优劣势？
3. C++20协程的“挂起恢复点”的信息保存在何处？
4. 一个可以被`co_await`的类型需要满足哪些特征？
5. `await_suspend`在何时被调用？我们一般会在其中实现什么功能？
6. `co_await ...`表达式的返回值由哪个函数的返回值决定？
7. 协程函数需要具备哪些特征？协程函数的返回类型需要满足哪些条件？
8. 为什么`std::suspend_always`可以被`co_await`？我们通常会出于什么目的去`co_await`一个`std::suspend_always`？
9. `std::coroutine_handle<promise_type>`和`std::coroutine_handle<void>`的区别是什么？为什么需要设计一个`std::coroutine_handle<void>`类型？

参考答案将随后提供，读者可以先利用以上问题检验以下学习效果。

## 标准库设施（进阶）

要写出一个简单的协程库，我们还需要了解更多的标准库功能。

### 利用`await_suspend`恢复上层协程

在返回`void`时，`await_suspend`会在执行完毕后挂起，并将执行权返回给主调度器。

除了返回`void`，`await_suspend`还可以返回`bool`。当`await_suspend`返回`true`时，表明需要阻塞，需要将执行权返回给主调度器；当其返回`false`时，表明不需要阻塞，直接转到`await_resume`执行。

此外，`await_suspend`可以通过返回另一个协程的`std::coroutine_handle`来恢复对应协程的执行。这个功能被普遍用于恢复上层协程执行。以[jbaldwin/libcoro](https://github.com/jbaldwin/libcoro)这个库为例——在当前协程的`await_suspend`中，拿到当前协程的handle之后，将这个handle填入下层协程的`m_continuation`字段中。

```cpp
auto await_suspend(std::coroutine_handle<> awaiting_coroutine) noexcept -> std::coroutine_handle<>
{
    m_coroutine.promise().continuation(awaiting_coroutine);  // 在这里设置了m_continuation
    return m_coroutine;
}
```

还记得我们上面提到的`final_suspend`的作用吗？在下层协程的`final_suspend`中，我们会返回一个Awaitable。这个Awaitable中保存了上层协程的handle，并且在它的`await_suspend`中，我们会返回这个上层协程的handle，用于恢复上层协程的执行：

```cpp
template<typename promise_type>
auto await_suspend(std::coroutine_handle<promise_type> coroutine) noexcept -> std::coroutine_handle<>
{
    auto& promise = coroutine.promise();
    if (promise.m_continuation != nullptr)
    {
        return promise.m_continuation;  // 在这里恢复上层协程
    }
    else
    {
        return std::noop_coroutine();
    }
}
```

### Awaitable与Awaiter的概念辨析


### 自定义协程帧的内存分配方法

