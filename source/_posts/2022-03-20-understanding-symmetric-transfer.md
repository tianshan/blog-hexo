---
title: "【翻译】C++ Coroutines: Understanding Symmetric Transfer"
date: 2022-03-20 22:25:04
tags: coroutine
---

这是*Lewis Baker*讲解协程的第四篇。这是原作者时隔2年后的文章，第一部分从简单例子中介绍并回顾之前的例子，从例子中引出无限递归会导致的栈溢出问题，然后第二部分介绍对称转移协程的设计以及是如何解决这个问题的。
本篇进一步从例子中来讲述实现协程的方式。

原文：https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer

* {% post_link 2022-02-07-coroutine-theory "第一篇 Coroutine Theory" %}
* {% post_link 2022-03-05-understanding-operator-co-await "第二篇 C++ Coroutines: understanding-operator-co-await" %}
* {% post_link 2022-03-12-understanding-the-promise-type "第三篇 C++ Coroutines: understanding-the-promise-type" %}

协程提案提供了很方便的方式来按照同步的方式写异步代码。你只需要在合适的点带上`co_await`，然后编译器就会处理好协程的挂起、保存挂起点前后的状态，以及当操作完成时恢复协程。

然而，协程提案中，特别是早期的版本，有很大的局限性，在不仔细处理时会很容易导致栈溢出。并且为了避免栈溢出，你不得不引入额外的同步开销来安全的保障你的`task<T>`类型。

幸运的是，在2018年协程的设计对这做了修补，增加了一项对称转移(symmetric transfer)的能力，允许你挂起一个协程然后恢复另外一个协程而不需要任何额外的栈空间。该项能力消除协程提案中的关键限制，使得实现异步协程类型可以更简单且更高效，而不用牺牲为了防止栈溢出所需的安全保证。

本篇中我会尝试解释栈溢出问题，以及“对称转移”能力是如何解决该问题的。

<!-- more -->

# 首先介绍协程任务如何工作的背景

有如下协程：

{% codeblock lang:c++ line_number:false %}
task foo() {
  co_return;
}

task bar() {
  co_await foo();
}
{% endcodeblock %}

假设我们有一个简单`task`类型，在其他协程await时惰性执行。这个特别的`task`类型不支持返回一个值。

让我们来解开看下当`bar()`执行`co_await foo()`时发生什么。

* `bar()`协程调用`foo()`函数。注意到对于调用者来说，协程和普通函数一样；
* 调用`foo()`时会执行如下步骤：
  * 为协程栈分配存储空间（通常在堆上）
  * 拷贝参数到协程栈（本例中没有参数所以没有执行）
  * 在协程栈上狗仔promise对象
  * 调用`promise.get_return_object()`来获取`foo()`的返回值。其中构造了即将返回的`task`对象，并用指向刚创建协程栈的`std::coroutine_handle`来初始化。
  * 在初始挂起点挂起协程的执行（例如左大括号那）
  * 返回`task`对象到`bar()`
* 接下来协程`bar()`对基于`foo()`返回`task`的`co_await`表达式求值；
  * 协程`bar()`挂起，然后对返回的`task`调用`await_suspend()`方法，并传入指向`bar()`协程栈的`std::coroutine_handle`
  * 方法`await_suspend()`存储`bar()`的`std::coroutine_handle`在`foo()`的promise对象，然后通过对`foo()`的`std::coroutine_handle`调用`.resume()`来恢复`foo()
* 协程`foo()`同步的运行到结束
* 协程`foo()`在最终挂起点（例如右大括号那）挂起，然后恢复在开始前保存在promise对象中的`std::coroutine_handle`所指示的协程（例如这里的协程`bar()`）
* 协程`bar()`恢复继续运行，并最终到最后一条包含`co_await`表达式的语句，在这个点调用从`foo()`返回的临时对象`task`的析构函数
* `task`析构函数中对`foo()`的协程句柄调用方法`.destroy()`，该方法中会销毁协程栈以及promise对象和所有参数拷贝

OK，看起来一个简单的调用也包含了很多步骤。

为了帮助更深入的理解，我们来使用协程提案（不包括对称转移）进一步实现`task`。

# 概要实现`task`

class task的轮廓如下：

{% codeblock lang:c++ line_number:false %}
class task {
public:
  class promise_type { /* see below */ };

  task(task&& t) noexcept
  : coro_(std::exchange(t.coro_, {}))
  {}

  ~task() {
    if (coro_)
      coro_.destroy();
  }

  class awaiter { /* see below */ };

  awaiter operator co_await() && noexcept;

private:
  explicit task(std::coroutine_handle<promise_type> h) noexcept
  : coro_(h)
  {}

  std::coroutine_handle<promise_type> coro_;
};
{% endcodeblock %}

调用协程中创建的协程栈对应的`std::coroutine_handle`是`task`独有的。对象`task`是一个RAII对象，在`task`生命周期结束时会确保对`std::coroutine_handle`调用`.destroy()`。

下一步让我们扩展`promise_type`。

# 实现`task::promise_type`

在{% post_link 2022-03-12-understanding-the-promise-type "上一篇" %}中，我们知道`promise_type`成员为在协程中创建的**Promise**对象类型，控制了协程的行为。

首先，我们需要实现`get_return_object()`来构造用于在协程调用时返回的`task`对象。该方法只需要用新创建协程栈的`std::coroutine_handle`来初始化。

我们可以用`std::coroutine_handle::from_promise()`方法来从promise对象中创建句柄。

{% codeblock lang:c++ line_number:false %}
class task::promise_type {
public:
  task get_return_object() noexcept {
    return task{std::coroutine_handle<promise_type>::from_promise(*this)};
  }
{% endcodeblock %}

下一步，我们需要协程在大括号起始点挂起，以便于之后在返回的`task`被期待(awaited)时从该点恢复协程。

惰性开始协程有如下一些好处：

1. 意味着我们可以在开始运行协程之前链接协程的`std::coroutine_handle`，也即我们不需要使用线程同步来仲裁链接到与协程运行到结束。
2. 意味着`task`析构函数可以无条件的销毁协程栈 - 我们不需要担心协程可能在另外一个线程上执行，因为协程并不会开始运行直到我们await，当协程运行时调用者协程会被挂起，且不会尝试调用task的析构函数支持协程结束运行。这使得编译器可以有更好的机会讲协程栈的初始化内联到调用者栈帧中。参考{% link P0981R0 https://wg21.link/P0981R0 %}或者更多关于Heap Allocation eLision Optimisation (HALO).
3. 这同样改进了协程代码的异常安全性。如果你不立即对返回的`task`执行`co_await`，而执行了一些可能抛出异常的逻辑并导致退栈，那么然后会执行`task`的析构函数，因为我们知道还没有开始运行所以可以安全的销毁协程。我们不会在 分离、可能留下悬空引用、在析构函数中阻塞、终止或未定义行为中做出艰难选择。在{% link CppCon 2019 talk on Structured Concurrency https://www.youtube.com/watch?v=1Wy5sq3s2rg %}中我给出了更多的细节。

为了使协程能在大括号起始点挂起，我们定义一个返回内建`suspend_always`类型的`initial_suspend()`方法。

{% codeblock lang:c++ line_number:false %}
  std::suspend_always initial_suspend() noexcept {
    return {};
  }
{% endcodeblock %}

下一步，我们需要定义`return_void()`方法，会在执行`co_return;`或者在协程运行到结束时被调用。这个方法并不需要做任何真的做任何事，仅需要该方法存在让编译器知道`co_return;`在该协程类型中是有效的。

{% codeblock lang:c++ line_number:false %}
  void return_void() noexcept {}
{% endcodeblock %}

我们同样需要增加方法`unhandled_exception()`，在未被捕获异常抛出协程体时会被调用。在我们的例子中可以认为`task`协程体是`noexcept`，当出现异常时则调用`std::terminate()`。

{% codeblock lang:c++ line_number:false %}
  void unhandled_exception() noexcept {
    std::terminate();
  }
{% endcodeblock %}

最终，当协程运行到右大括号时，我们希望协程能够在最终挂起点挂起，且在之后能继续运行。例如继续运行等待本协程的awaiting协程。

为了支持这点，在promise的数据成员中，我们需要存储继续运行者的`std::coroutine_handle`。我们同时需要定义返回awaitable对象的`final_suspend()`方法，来在当前协程于最终挂起点挂起时恢复继续运行着。

在当前协程挂起之后再恢复继续运行是重要的，因为继续运行者中可能会立即调用`task`析构函数即对协程栈调用`.destroy()`。方法`.destroy()`只能对挂起的协程执行，因此在当前协程挂起前恢复执行会是未定义行为。

在右大括号结束时，编译器插入代码来对语句`co_await promise.final_suspend();`求值。

需要重点注意的是在调用`final_suspend()`方法时协程并不处于挂起状态。我们需要在协程挂起前等待直到返回的awaitable的被调用`await_suspend()`方法。

{% codeblock lang:c++ line_number:false %}
  struct final_awaiter {
    bool await_ready() noexcept {
      return false;
    }

    void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
      // The coroutine is now suspended at the final-suspend point.
      // Lookup its continuation in the promise and resume it.
      h.promise().continuation.resume();
    }

    void await_resume() noexcept {}
  };

  final_awaiter final_suspend() noexcept {
    return {};
  }

  std::coroutine_handle<> continuation;
};
{% endcodeblock %}

OK，这就是完整的`promise_type`。最后需要实现的一部分是`task::operator co_await()`。

# 实现`task::operator co_await()`

你可能记得在{% post_link 2022-03-05-understanding-operator-co-await "Understanding operator co_await() post" %}中，当对`co_await`表达式求值时，编译器会生成对`operator co_await()`的调用，如果定义了一种，然后返回的对象必须有用`await_ready()`，`await_suspend()`和`await_resume()`方法。

当协程awaits一个`task`，我们希望awaiting协程总是挂起的，一旦挂起了，就存储awaiting协程的句柄到即将恢复的协程promise中，对`task`的`std::coroutine_handle`调用`.resume()`来开始执行task。

因此直观代码如下：

{% codeblock lang:c++ line_number:false %}
class task::awaiter {
public:
  bool await_ready() noexcept {
    return false;
  }

  void await_suspend(std::coroutine_handle<> continuation) noexcept {
    // Store the continuation in the task's promise so that the final_suspend()
    // knows to resume this coroutine when the task completes.
    coro_.promise().continuation = continuation;

    // Then we resume the task's coroutine, which is currently suspended
    // at the initial-suspend-point (ie. at the open curly brace).
    coro_.resume();
  }

  void await_resume() noexcept {}

private:
  explicit awaiter(std::coroutine_handle<task::promise_type> h) noexcept
  : coro_(h)
  {}

  std::coroutine_handle<task::promise_type> coro_;
};

task::awaiter task::operator co_await() && noexcept {
  return awaiter{coro_};
}
{% endcodeblock %}

于是完成了一个功能型`task`类型所需的代码。

可以在这里看到完整的代码：https://godbolt.org/z/-Kw6Nf

# 栈溢出问题

然而，一些实现的限制出现了。当你开始在协程中写循环时，`co_await`任务可能会在循环体中同步的完成。

例如：

{% codeblock lang:c++ line_number:false %}
task completes_synchronously() {
  co_return;
}

task loop_synchronously(int count) {
  for (int i = 0; i < count; ++i) {
    co_await completes_synchronously();
  }
}
{% endcodeblock %}

当`task`如上描述的简单实现时，`loop_synchronously()`函数在`count`是10、1000或者甚至100 000都是能正常工作的。但是一定会有值会最终导致协程开始异常。

例如，在 https://godbolt.org/z/gy5Q8q 中，当`count`是1'000'000时崩溃。

崩溃的原因是栈溢出了。

为了理解为什么会导致栈溢出，我们需要仔细看下代码运行时发生了什么。特别的，栈上发生了什么。

在其他协程对返回的`task`执行`co_await`时，`loop_synchronously()`协程开始运行。这同时会挂起awaiting的协程并调用`task::awaiter::await_suspend()`，即对task的`std::coroutine_handle`调用`resume()`。

因此当`loop_synchronously()`开始时，栈会看起来如下：

{% codeblock lang:c++ line_number:false %}
           Stack                                                   Heap
+------------------------------+  <-- top of stack   +--------------------------+
| loop_synchronously$resume    | active coroutine -> | loop_synchronously frame |
+------------------------------+                     | +----------------------+ |
| coroutine_handle::resume     |                     | | task::promise        | |
+------------------------------+                     | | - continuation --.   | |
| task::awaiter::await_suspend |                     | +------------------|---+ |
+------------------------------+                     | ...                |     |
| awaiting_coroutine$resume    |                     +--------------------|-----+
+------------------------------+                                          V
|  ....                        |                     +--------------------------+
+------------------------------+                     | awaiting_coroutine frame |
                                                     |                          |
                                                     +--------------------------+
{% endcodeblock %}

> 注意：编译器通常会把协程函数分为两部分：
> 1. ”ramp function“部分，用于处理协程栈的构造，参数拷贝，promise构造，以及创建返回值
> 2. ”coroutine body“部分，包含协程体中的用户逻辑。
>
> 我会使用`$resume`后缀来代表协程的”coroutine body“部分。
后续的博客中会进一步分析拆分的逻辑。

然后当`loop_synchronously()`对返回自`completes_synchronously()`的`task`执行await时，当前协程会挂起并调用`task::awaiter::await_suspend()`。然后`await_suspend()`方法会调用`completes_synchronously()`协程对应句柄的`.resume()`。

这会恢复`completes_synchronously()`协程，同步地运行到结束，并在最终挂起点挂起。然后调用`task::promise::final_awaiter::await_suspend()`，这里会调用`loop_synchronously()`对应协程句柄的`.resume()`。

下面网状的结果是在`loop_synchronously()`协程恢复后且在由`completes_synchronously()`返回的临时`task`销毁前即分号点的程序状态下堆栈的样子：

{% codeblock lang:c++ line_number:false %}
           Stack                                                   Heap
+-------------------------------+ <-- top of stack
| loop_synchronously$resume     | active coroutine -.
+-------------------------------+                   |
| coroutine_handle::resume      |            .------'
+-------------------------------+            |
| final_awaiter::await_suspend  |            |
+-------------------------------+            |  +--------------------------+ <-.
| completes_synchronously$resume|            |  | completes_synchronously  |   |
+-------------------------------+            |  | frame                    |   |
| coroutine_handle::resume      |            |  +--------------------------+   |
+-------------------------------+            '---.                             |
| task::awaiter::await_suspend  |                V                             |
+-------------------------------+ <-- prev top  +--------------------------+   |
| loop_synchronously$resume     |     of stack  | loop_synchronously frame |   |
+-------------------------------+               | +----------------------+ |   |
| coroutine_handle::resume      |               | | task::promise        | |   |
+-------------------------------+               | | - continuation --.   | |   |
| task::awaiter::await_suspend  |               | +------------------|---+ |   |
+-------------------------------+               | - task temporary --|---------'
| awaiting_coroutine$resume     |               +--------------------|-----+
+-------------------------------+                                    V
|  ....                         |               +--------------------------+
+-------------------------------+               | awaiting_coroutine frame |
                                                |                          |
                                                +--------------------------+
{% endcodeblock %}

下一步要做的是调用`task`析构函数来销毁`completes_synchronously()`栈。然后增加变量`count`并进入下一轮循环，创建一个新的`completes_synchronously()`栈并恢复。

事实上，这里发生的是`loop_synchronously()`和`completes_synchronously()`在互相递归调用。每一次发生时都会消耗一部分栈空间，最终，在足够迭代后，超出了栈空间并导致未定义行为，典型的导致程序崩溃。

在协程中以这种方式写循环很容易导致无限的递归，虽然看起来并没有任何递归。

所以，在原始协程提案设计中如何解决这个问题？

# 协程提案的解决方案

OK，我们可以怎么做来避免无限递归呢？

通过上述实现，我们使用的`await_suspend()`返回void的版本。协程提案中海油一个版本的`await_suspend()`返回`bool` - 如果返回`true`那么协程会挂起且运行权返回到`resume()`的调用者，如果返回`false`那么协程会立即恢复，这时不会消耗任何额外的栈空间。

所以，为了避免无限相互递归，我们需要做的是，在task同步完成时使用返回`bool`版本的`await_suspend()`从`task::awaiter::await_suspend()`方法中返回`false`来恢复当前协程，而不是用`std::coroutine_handle::resume()`递归的恢复协程。

通用解决方案的实现包含两部分：

1. 方法`task::awaiter::await_suspend()`中，可以通过调用`.resume()`来开始运行协程。然后调用`.resume()`返回时，检查协程是否运行到了结束。如果运行到完成了那么返回`false`，表明awaiting协程应该理解恢复，或者返回`true`，表明运行权需要返回到`std::coroutine_handle::resume()`的调用者；
2. 在`task::promise_type::final_awaiter::await_suspend()`中，即协程运行到结束时会被调用的函数，我们需要检查awaiting协程是否已经或者即将在`task::awaiter::await_suspend()`中返回`true`，如果是那么通过调用`.resume()`来恢复他。否则，我们应该避免恢复协程而是通知`task::awaiter::await_suspend()`来返回`false`；

然而，还有一个复杂问题，要实现上述目标，需要让协程能够在当前线程开始运行然后挂起，之后在调用`.resume()`返回前于不同的线程恢复并运行到结束。因此我们需要解决第一部分和第二部分同时发生的潜在竞争冲突。

这里使用了一个`std::atomic`值来决定竞争结果。

现在对于代码我们可以做如下修改：

{% codeblock lang:c++ line_number:false %}
class task::promise_type {
  ...

  std::coroutine_handle<> continuation;
  std::atomic<bool> ready = false;
};

bool task::awaiter::await_suspend(
    std::coroutine_handle<> continuation) noexcept {
  promise_type& promise = coro_.promise();
  promise.continuation = continuation;
  coro_.resume();
  return !promise.ready.exchange(true, std::memory_order_acq_rel);
}

void task::promise_type::final_awaiter::await_suspend(
    std::coroutine_handle<promise_type> h) noexcept {
  promise_type& promise = h.promise();
  if (promise.ready.exchange(true, std::memory_order_acq_rel)) {
    // The coroutine did not complete synchronously, resume it here.
    promise.continuation.resume();
  }
}
{% endcodeblock %}

可以在这里看到样例：https://godbolt.org/z/7fm8Za 可以观察到执行`count == 1'000'000`时不再崩溃。

这也是`cppcoro::task<T>` {% link implementation https://github.com/lewissbaker/cppcoro/blob/master/include/cppcoro/task.hpp %}中用来避免无限递归的方法（在某些平台仍然如此），并且运作良好。

Woohoo！问题解决了，对吗？

# 问题所在

上述解决方案虽然解决了递归问题，但是有一些缺点。

首先，引入了`std::atomic`操作，带来了不小的开销。在调用者挂起awaiting协程时需要一次原子交互，在被调者运行到结束时有另外一次原子交换。如果你的程序只在单线程运行，那么你需要为额外付出线程间同步的原子操作开销。

其次，引入了额外的分支。其一在调用者端，需要决定是挂起还是立即恢复协程，其二在被调者端，需要决定是继续运行或者挂起。

注意额外分支的开销，甚至是原子操作，对比协程中的业务逻辑的开销通常是不值一提的。然而，协程通常被宣传为零开销的抽象，甚至有使用协程来挂起函数运行来避免L1-cache-miss(Gor的视频{% link "CppCon talk on nanocoroutines" https://www.youtube.com/watch?v=j9tlJAqMV7U %}有更多细节)。

第三点，可能是最重要的，在awaiting协程恢复的上下文运行中引入了不确定性。

例如我有如下代码：

{% codeblock lang:c++ line_number:false %}
cppcoro::static_thread_pool tp;

task foo()
{
  std::cout << "foo1 " << std::this_thread::get_id() << "\n";
  // Suspend coroutine and reschedule onto thread-pool thread.
  co_await tp.schedule();
  std::cout << "foo2 " << std::this_thread::get_id() << "\n";
}

task bar()
{
  std::cout << "bar1 " << std::this_thread::get_id() << "\n";
  co_await foo();
  std::cout << "bar2" << std::this_thread::get_id() << "\n";
}
{% endcodeblock %}

在原始实现中，我们保证了在`co_await foo()`之后的代码会内联在`foo()`结束的线程上运行。

例如，一个可能的输出结果：

{% codeblock lang:c++ line_number:false %}
bar1 1234
foo1 1234
foo2 3456
bar2 3456
{% endcodeblock %}

然而，在使用原子操作后，`foo()`的完成可能会和`bar()`的挂起竞争，并且在某些情况下，意味着`co_await foo()`之后的代码可能会在`bar()`一开始的线程上运行。

例如，也可能会发生如下的输出序列：

{% codeblock lang:c++ line_number:false %}
bar1 1234
foo1 1234
foo2 3456
bar2 1234
{% endcodeblock %}

对于很多使用场景，这个行为并不会导致区别。然而对于以转移运行上下文为目的的运算就会是问题。

例如，算法`via()`中await一些Awaitable对象然后在特定的调度器运行上下文来使用。一个简化版本的算法如下：

{% codeblock lang:c++ line_number:false %}
template<typename Awaitable, typename Scheduler>
task<await_result_t<Awaitable>> via(Awaitable a, Scheduler s)
{
  auto result = co_await std::move(a);
  co_await s.schedule();
  co_return result;
}

task<T> get_value();
void consume(const T&);

task<void> consumer(static_thread_pool::scheduler s)
{
  T result = co_await via(get_value(), s);
  consume(result);
}
{% endcodeblock %}

初始版本的`consume()`调用始终保证了再线程池`s`上运行。然而在使用了atomic的改进版本中，`consume()`可能会运行在调度器`s`的线程上或者协程`consumer()`开始运行的线程上。

所以在不引入原子操作、额外分支以及非确定恢复上下文时，我们该如何解决栈溢出问题？

# 使用”对称转移“！

Gor Nishanov在{% link P0913R0 https://wg21.link/P0913R0 %}（2018）论文中提议了一种解决方案，提供允许一个协程挂起然后对称的恢复另外一个协程的机制，不消耗任何额外的栈空间。

论文中提出了两个关键点：

* 允许从`await_suspend()`中返回`std::coroutine_handle<T>`，作为指示运行权应该对称的转移到返回句柄对应的协程
* 增加一个返回特殊`std::coroutine_handle`的`std::experimental::noop_coroutine()`函数，这个句柄可以被`await_suspend()`返回来挂起当前协程，并且从`.resume()`调用中返回，而不是转移运行到其他协程

所以”对称转移“到底意味着什么呢？

当你通过对`std::coroutine_handle`调用`.resume()`来恢复一个协程时，`.resume()`的调用者在被恢复协程运行的同时在栈上保持存活。在这个协程挂起且在挂起点调用`await_suspend()`返回`void`（表明无条件挂起）或者`true`（表明有条件挂起），然后`.resume()`的调用会返回。

这可以被认为是协程的一种”非对称转移“运行，和普通函数调用一样。`.resume()`的调用者可以是任何函数（可以是协程也可以不是）。当协程挂起且在`await_suspend()`中返回`true`或者`void`，然后运行权会从`.resume()`的调用中返回。并且，每一次通过调用`.resume()`恢复一个协程都会为协程的运行创建一个新的栈帧。

然而，在”对称转移“中，我们可以很简单的挂起一个协程并恢复另外一个协程。在两个协程中并没有隐式包含调用者和被调者的关系 - 当一个协程挂起时可以转移运行到任何挂起协程（包括自己），且在下一次挂起或者完成时并不一定要转移运行到上一次的协程。

让我们来看下编译器会如何映射`co_await`表达式，即awaiter在什么时候使用对称转移：

{% codeblock lang:c++ line_number:false %}
{
  decltype(auto) value = <expr>;
  decltype(auto) awaitable =
      get_awaitable(promise, static_cast<decltype(value)&&>(value));
  decltype(auto) awaiter =
      get_awaiter(static_cast<decltype(awaitable)&&>(awaitable));
  if (!awaiter.await_ready())
  {
    using handle_t = std::coroutine_handle<P>;

    //<suspend-coroutine>

    auto h = awaiter.await_suspend(handle_t::from_promise(p));
    h.resume();
    //<return-to-caller-or-resumer>
    
    //<resume-point>
  }

  return awaiter.await_resume();
}
{% endcodeblock %}

让我们深入看下和其他`co_await`形式的关键不同之处：

{% codeblock lang:c++ line_number:false %}
auto h = awaiter.await_suspend(handle_t::from_promise(p));
h.resume();
//<return-to-caller-or-resumer>
{% endcodeblock %}

一旦协程的状态机映射完（另外一篇的主题），`<return-to-caller-or-resumer>`部分基本就变为`return;`语句，即导致把恢复协程的`.resume()`调用返回给调用者。

这意味着我们会遇到相同签名的调用会转到另外一个函数的情形，`std::coroutine_handle::resume()`，紧跟着当前函数的`return;`，该函数本身既是`std::coroutine_handle::resume()`调用的主题。

一些编译器，在编译优化打开时，可以某些情形下对在尾部（就在返回前）另外一个函数的调用优化为尾调用。

这类尾调用优化就是我们需要的，用来避免前述的栈溢出问题。但是与其等待编译器来决定是否执行尾调用优化，我们希望保证尾调用转化一定发生，即使没有开启优化。

但是首先我们来看下什么是尾调用。

## 尾调用(Tail-calls)

尾调用是指在调用之前弹出当前栈帧，且把当前函数返回地址作为被调用者的返回地址。例如，被调用者会直接返回此函数的调用者。

在x86/x64架构上，这意味着编译器会生成代码，首先弹出当前栈帧，然后使用`jmp`指令跳转到被调用函数入口，而不是使用`call`指令然后在`call`返回时弹出当前栈帧。

然而这个优化通常只能在有限的情形下使用。

特别的，需要：

* 调用者和被调用者的调用约定都要支持尾调用；
* 返回类型一致；
* 在返回调用者之前，不需要在调用之后运行non-trivial的析构函数；
* 调用不在try/catch块中；

`co_await`的对称转移形式被特别设计用于使得协程满足以上所有要求。让我们一个个看一下。

**调用约定** 当编译器映射协程为机器码时，事实上会分裂协程为两部分：the ramp（分配并初始化协程栈）和函数体（包含用户编写的协程体状态机）。

协程的函数签名（以及任何用户特定的调用约定）仅影响ramp部分，而协程体部分是在编译器的控制下且并不会被任何用户代码直接调用 - 仅会被ramp函数以及`std::coroutine_handle::resume()`调用。

协程体部分的调用约定并不对用户可见，并且完全取决于编译器，因此可以选择一种合适的调用约定，支持尾调用且被用于所有协程体。

**返回类型一致** 源和目标协程的`.resume()`方法的返回类型都是`void`，因此要求是满足的。

**没有non-trivial析构函数** 当执行尾调用时，我们需要在调用目标函数前能释放当前栈帧，这需要所有栈分配对象的生命周期在调用前都已结束。

通常来说，只要在范围内有任何已经分配在栈上，且有non-trivial析构函数对象的生命周期还没结束就是有问题的。

然而，当协程挂起时在不退出任何作用域的情况就能达到，实现的方式是将任何生命周期跨越挂起点的对象放到协程栈上而不是在栈上。

生命周期不跨越挂起点的局部变量可能会在栈上分配，但是这些对象的生命周期已经结束，且在协程下一个挂起点调用之前就会调用他们的析构函数。

因此在尾调用返回之后，不需要运行在栈上分配的对象的non-trivial析构函数。

**不在try/catch块中调用** 这有一点tricky，因为在每个协程中，隐式的包含了一个try/catch块包住了用户编写的协程体。

举例说明，我们假设协程定义如下：

{% codeblock lang:c++ line_number:false %}
{
  promise_type promise;
  co_await promise.initial_suspend();
  try { F; }
  catch (...) { promise.unhandled_exception(); }
final_suspend:
  co_await promise.final_suspend();
}
{% endcodeblock %}

其中`F`是用户编写的协程体。

因此每一个用户编写的`co_await`表达式（除了初始和最终挂起点）都包含在一个try/catch的上下文中。

但是，实现是通过在try块上下文之外再实际执行`.resume()`调用来解决。

我希望能在另外一篇博客中进一步深挖这部分细节，即协程映射为机器码的细节（本篇已经足够长了）。

> 然而，注意到当前C++规范中对要求实现的这点措辞并不明确，这只是一个非规范性说明，暗示这可能是必须的。希望我们在未来能解决。

所以我们看到协程执行对称转移总体上是满足所有要求来执行尾调用。编译器保证了这总是尾调用，不管优化开启与否。

这意味着通过使用返回`std::coroutine_handle`的`await_suspend()`，我们可以挂起当前协程以及转移运行到另外一个协程而不消耗额外栈空间。

这允许我们编写任何深度的互相递归且互相恢复的协程而不用担心栈溢出。

这就是我们需要来修复`task`的实现。

# 重新设计`task`

所以在拥有新的”对称转移“能力后，让我们来回头修复`task`类型的实现。

我们需要改动两个`await_suspend()`方法的实现：

* 首先当我们await task时，执行一次对称转移来恢复task的协程
* 其次当task的协程完成时，执行一次对称转移来恢复awaiting协程

为了解决await方法，我们需要改变`task::awaiter`方法，从：

{% codeblock lang:c++ line_number:false %}
void task::awaiter::await_suspend(
    std::coroutine_handle<> continuation) noexcept {
  // Store the continuation in the task's promise so that the final_suspend()
  // knows to resume this coroutine when the task completes.
  coro_.promise().continuation = continuation;

  // Then we resume the task's coroutine, which is currently suspended
  // at the initial-suspend-point (ie. at the open curly brace).
  coro_.resume();
}
{% endcodeblock %}

改变为：

{% codeblock lang:c++ line_number:false %}
std::coroutine_handle<> task::awaiter::await_suspend(
    std::coroutine_handle<> continuation) noexcept {
  // Store the continuation in the task's promise so that the final_suspend()
  // knows to resume this coroutine when the task completes.
  coro_.promise().continuation = continuation;

  // Then we tail-resume the task's coroutine, which is currently suspended
  // at the initial-suspend-point (ie. at the open curly brace), by returning
  // its handle from await_suspend().
  return coro_;
}
{% endcodeblock %}

为了解决返回路径，我们需要更新`task::promise_type::final_awaiter`方法，从：

{% codeblock lang:c++ line_number:false %}
void task::promise_type::final_awaiter::await_suspend(
    std::coroutine_handle<promise_type> h) noexcept {
  // The coroutine is now suspended at the final-suspend point.
  // Lookup its continuation in the promise and resume it.
  h.promise().continuation.resume();
}
{% endcodeblock %}

改变为：

{% codeblock lang:c++ line_number:false %}
std::coroutine_handle<> task::promise_type::final_awaiter::await_suspend(
    std::coroutine_handle<promise_type> h) noexcept {
  // The coroutine is now suspended at the final-suspend point.
  // Lookup its continuation in the promise and resume it symmetrically.
  return h.promise().continuation;
}
{% endcodeblock %}

现在我们有了一个`task`实现，不会出现返回`void`版本的`await_suspend`所具有的栈溢出问题，并且不会有返回`bool`版本的`await_suspend`会出现的不确定上下文恢复问题。

# 栈的表现

现在我们再回头看一下最初的样例：

{% codeblock lang:c++ line_number:false %}
task completes_synchronously() {
  co_return;
}

task loop_synchronously(int count) {
  for (int i = 0; i < count; ++i) {
    co_await completes_synchronously();
  }
}
{% endcodeblock %}

当`loop_synchronously()`协程第一次开始运行时，会因为其他协程`co_await`该`task`而返回。这会通过其他协程的对称转移来启动，即通过调用`std::coroutine_handle::resume()`来恢复。

因此在`loop_synchronously()`开始时，堆栈会看起来如下：

{% codeblock lang:c++ line_number:false %}
           Stack                                                Heap
+---------------------------+  <-- top of stack   +--------------------------+
| loop_synchronously$resume | active coroutine -> | loop_synchronously frame |
+---------------------------+                     | +----------------------+ |
| coroutine_handle::resume  |                     | | task::promise        | |
+---------------------------+                     | | - continuation --.   | |
|     ...                   |                     | +------------------|---+ |
+---------------------------+                     | ...                |     |
                                                  +--------------------|-----+
                                                                       V
                                                  +--------------------------+
                                                  | awaiting_coroutine frame |
                                                  |                          |
                                                  +--------------------------+
{% endcodeblock %}

现在当运行`co_await completes_synchronously()`时，会执行对称转移到`completes_synchronously`协程。

通过如下操作实现：

* 调用`task::operator co_await()`，然后返回`task::awaiter`对象
* 然后挂起并调用`task::awaiter::await_suspend()`，然后返回`completes_synchronously`协程的`coroutine_handle`
* 再然后执行尾调用/跳转到`completes_synchronously`协程。这会在激活`completes_synchronously`栈帧前弹出`loop_synchronously`的栈帧；

如果我们在`completes_synchronously`恢复后查看堆栈状态如下：

{% codeblock lang:c++ line_number:false %}
              Stack                                          Heap
                                            .-> +--------------------------+ <-.
                                            |   | completes_synchronously  |   |
                                            |   | frame                    |   |
                                            |   | +----------------------+ |   |
                                            |   | | task::promise        | |   |
                                            |   | | - continuation --.   | |   |
                                            |   | +------------------|---+ |   |
                                            `-, +--------------------|-----+   |
                                              |                      V         |
+-------------------------------+ <-- top of  | +--------------------------+   |
| completes_synchronously$resume|     stack   | | loop_synchronously frame |   |
+-------------------------------+ active -----' | +----------------------+ |   |
| coroutine_handle::resume      | coroutine     | | task::promise        | |   |
+-------------------------------+               | | - continuation --.   | |   |
|     ...                       |               | +------------------|---+ |   |
+-------------------------------+               | task temporary     |     |   |
                                                | - coro_       -----|---------`
                                                +--------------------|-----+
                                                                     V
                                                +--------------------------+
                                                | awaiting_coroutine frame |
                                                |                          |
                                                +--------------------------+
{% endcodeblock %}

请注意此处的栈帧数量并没有变多。

在`completes_synchronously`协程完成后，运行到右大括号会对`co_await promise.final_suspend()`求值。

这会挂起协程并调用`final_awaiter::await_suspend()`，其中会返回协程的`std::coroutine_handle`（例如指向`loop_synchronously`协程的句柄）。然后执行一次对称转移/尾调用来恢复`loop_synchronously`协程。

在`loop_synchronously`恢复后，查看堆栈状态如下：

{% codeblock lang:c++ line_number:false %}
           Stack                                                   Heap
                                                   +--------------------------+ <-.
                                                   | completes_synchronously  |   |
                                                   | frame                    |   |
                                                   | +----------------------+ |   |
                                                   | | task::promise        | |   |
                                                   | | - continuation --.   | |   |
                                                   | +------------------|---+ |   |
                                                   +--------------------|-----+   |
                                                                        V         |
+----------------------------+  <-- top of stack   +--------------------------+   |
| loop_synchronously$resume  | active coroutine -> | loop_synchronously frame |   |
+----------------------------+                     | +----------------------+ |   |
| coroutine_handle::resume() |                     | | task::promise        | |   |
+----------------------------+                     | | - continuation --.   | |   |
|     ...                    |                     | +------------------|---+ |   |
+----------------------------+                     | task temporary     |     |   |
                                                   | - coro_       -----|---------`
                                                   +--------------------|-----+
                                                                        V
                                                   +--------------------------+
                                                   | awaiting_coroutine frame |
                                                   |                          |
                                                   +--------------------------+
{% endcodeblock %}

`loop_synchronously`协程一旦返回后首先要做的是，在运行到分号时，调用对`completes_synchronously`调用中返回的临时`task`的析构函数。这会销毁协程栈，释放内存，以及进入如下状态：

{% codeblock lang:c++ line_number:false %}
           Stack                                                   Heap
+---------------------------+  <-- top of stack   +--------------------------+
| loop_synchronously$resume | active coroutine -> | loop_synchronously frame |
+---------------------------+                     | +----------------------+ |
| coroutine_handle::resume  |                     | | task::promise        | |
+---------------------------+                     | | - continuation --.   | |
|     ...                   |                     | +------------------|---+ |
+---------------------------+                     | ...                |     |
                                                  +--------------------|-----+
                                                                       V
                                                  +--------------------------+
                                                  | awaiting_coroutine frame |
                                                  |                          |
                                                  +--------------------------+
{% endcodeblock %}

现在我们回到运行`loop_synchronously`协程，拥有和开始时一样多的栈帧和协程栈，并且每运行完一轮循环皆是如此。

因此我们可以使用固定大小的存储空间来执行任意多的循环。

`task`类型的对称转移完整样例版本参见：https://godbolt.org/z/9baieF 。

# 对称转移作为await_suspend的通用形式

现在我们看到了awaitable概念的对称转移的能力与重要性，我想展示这确实是通用的形式，理论上可以替代返回`void`和`bool`形式的`await_suspend()`。

但是首先我们需要看下{% link P0913R0 https://wg21.link/P0913R0 %}提议中给协程增加的其他设计`std::noop_coroutine()`。

## 结束递归

拥有对称转移后的协程，每次挂起时会对称的恢复其他协程。在有另外的协程可以恢复时完全没问题，但是有时我们并没有其他协程在运行，仅需要挂起并把运行返回到`std::coroutine_handle::resume()`的调用者。

返回`void`和`bool`版本的`await_suspend()`都允许协程挂起并从`std::coroutine_handle::resume()`中返回，所以我们该如何使用对称转移做到这点呢？

答案是通过使用特别内建的`std::coroutine_handle`，由函数`std::noop_coroutine()`产生的"noop coroutine handle"。

"noop coroutine handle"被称为如此，是因为他的`.resume()`实现是立即返回。例如，恢复的协程是no-op。典型的实现是包含单一一条`ret`指令。

如果`await_suspend()`方法返回`std::noop_coroutine()`句柄，然后会转移运行回到`std::coroutine_handle::resume()`的调用者，而不是转移运行到下一个协程。

## 其他版本`await_suspend()`的形式

基于手中的信息，我们现在可以展现其他版本`await_suspend()`使用对称转移的形式。

返回`void`的版本：

{% codeblock lang:c++ line_number:false %}
void my_awaiter::await_suspend(std::coroutine_handle<> h) {
  this->coro = h;
  enqueue(this);
}
{% endcodeblock %}

返回`bool`的版本：

{% codeblock lang:c++ line_number:false %}
bool my_awaiter::await_suspend(std::coroutine_handle<> h) {
  this->coro = h;
  enqueue(this);
  return true;
}
{% endcodeblock %}

以及用对称转移形式编写如下版本：

{% codeblock lang:c++ line_number:false %}
std::noop_coroutine_handle my_awaiter::await_suspend(
    std::coroutine_handle<> h) {
  this->coro = h;
  enqueue(this);
  return std::noop_coroutine();
}
{% endcodeblock %}

原本返回`bool`的版本：

{% codeblock lang:c++ line_number:false %}
bool my_awaiter::await_suspend(std::coroutine_handle<> h) {
  this->coro = h;
  if (try_start(this)) {
    // Operation will complete asynchronously.
    // Return true to transfer execution to caller of
    // coroutine_handle::resume().
    return true;
  }

  // Operation completed synchronously.
  // Return false to immediately resume the current coroutine.
  return false;
}
{% endcodeblock %}

同样可以使用对称转移变为：

{% codeblock lang:c++ line_number:false %}
std::coroutine_handle<> my_awaiter::await_suspend(std::coroutine_handle<> h) {
  this->coro = h;
  if (try_start(this)) {
    // Operation will complete asynchronously.
    // Return std::noop_coroutine() to transfer execution to caller of
    // coroutine_handle::resume().
    return std::noop_coroutine();
  }

  // Operation completed synchronously.
  // Return current coroutine's handle to immediately resume
  // the current coroutine.
  return h;
}
{% endcodeblock %}

## 为什么需要所有3个版本？

所以在有对称转移后，为什么我们还需要返回`void`和`bool`版本的`await_suspend()`呢？

一部分是历史原因，一部分是实用性，一部分是性能问题。

从`await_suspend()`中返回`void`的版本可以被返回`std::noop_coroutine_handle`类型完全替代，因为对编译器来说都是无条件转移运行到`std::coroutine_handle::resume()`的调用者。

在我看来，这个版本被保留，一部分是因为在引入对称转移前已经使用了，一部分是因为在无条件转移场景`void`版本可以写更少的代码。

而对于返回`bool`的版本，在某些场景下，优化性能会略优于对称转移。

考虑我们有在`await_suspend()`返回`bool`的方法定义在另外的编译单元。这种情况下编译器可以在awaiting协程中生成代码，用于挂起当前协程，然后在调用`await_suspend()`返回后通过执行下一段代码有条件的恢复它。如果`await_suspend()`返回`false`就会确切的知道接下来要执行的代码。

在使用对称转移后我们仍然需要表示相同的结果，要么返回到调用者/恢复者，或者恢复当前协程。我们需要返回`std::noop_coroutine()`或者当前协程的句柄而不是`true`或者`false`。我们可以把这两类句柄都强制转换为`std::coroutine_handle<void>`类型并返回。

然而现在因为`await_suspend()`方法定义在另外的编译单元，编译器并不能看到返回的句柄指向哪个协程，所以对比返回`bool`版本的单一分支，现在恢复协程时必须执行更多昂贵的间接调用或者可能一些分支来恢复协程。

我们可能有一天会获得相同性能的对称转移版本。例如，我们可以如下方式写代码，把`await_suspend()`定义为内联，但是调用一个在其他地方定义的返回`bool`方法，有条件的返回合适的句柄。

例如：

{% codeblock lang:c++ line_number:false %}
struct my_awaiter {
  bool await_ready();

  // Compilers should in-theory be able to optimise this to the same
  // as the bool-returning version, but currently don't do this optimisation.
  std::coroutine_handle<> await_suspend(std::coroutine_handle<> h) {
    if (try_start(h)) {
      return std::noop_coroutine();
    } else {
      return h;
    }
  }

  void await_resume();

private:
  // This method is defined out-of-line in a separate translation unit.
  bool try_start(std::coroutine_handle<> h);
}
{% endcodeblock %}

然而，当前的编译器(Clang 10)还不支持把他优化为和返回`bool`的版本一样高效的代码。话虽如此，除非是在一个非常紧凑的循环中等待，否则可能都不会注意到差异。

总的来说，一般的规则如下：

* 如果需要无条件返回到`.resume()`调用者，使用`void`版本；
* 如果需要有条件的返回到`.resume()`调用者或者恢复当前协程，使用`bool`版本；
* 如果需要恢复另外的协程，使用对称转移版本；

# 总结

添加到C++20协程新的对称转移能力使得写协程更容易了，即递归的互相恢复不再需要担心栈溢出。这个能力是编写高效且安全异步协程类型的关键，例如这里的`task`类型。