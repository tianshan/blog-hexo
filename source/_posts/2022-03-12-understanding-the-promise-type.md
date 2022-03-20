---
title: "【翻译】C++ Coroutines: understanding-the-promise-type"
date: 2022-03-12 21:26:19
tags: coroutine
---

这是*Lewis Baker*讲解协程的第三篇。本篇主要描述如何自定义协程关键字的行为。
原文：https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type

* {% post_link 2022-02-07-coroutine-theory "第一篇 Coroutine Theory" %}
* {% post_link 2022-03-05-understanding-operator-co-await "第二篇 C++ Coroutines: understanding-operator-co-await" %}

本篇主要讲解编译器如何转义协程中的代码，以及如何通过自定义**Promise**类型来定制协程行为。

<!-- more -->

# 协程概念

协程提案添加了3个关键字：`co_await`, `co_yield`和`co_return`。无论何时只要在函数体内使用了其中之一的关键字，就会触发编译器把该函数编译为协程而不是普通函数。

编译器使用了很机械的转换，即转换代码为状态机，允许在函数内特定位置暂停，以及之后的恢复执行。

前面两篇中我描述了协程提案中介绍的两个新接口中的第一个：**Awaitable**接口。第二个对于代码转换很重要的接口是**Promise**接口。

**Promise**接口指定了定义协程本身行为的方法。库编写者可以自定义协程被调用后的行为，协程返回后的行为（包括正常返回以及异常返回），以及协程内任何`co_await`或者`co_yield`表达式的行为。

# Promise对象

**Promise**对象定义并控制了协程本身的行为，通过实现协程执行中在特殊点调用的方法来达成。

> 在继续之前，我希望你能摆脱对”promise“的任何预想的观点。虽然在一些用例中，协程的promise对象确实和`std::future`对的`std::promise`角色很像，对于其他的用例，这样的类比会有一点延伸。把协程的promise对象想象成”协程状态控制者“对象更容易，即控制协程的行为以及可以被用来跟踪它的状态。

promise对象的实例是在每一个协程函数调用中的协程栈内构造的。

编译器会在协程运行中的关键点生成调用promise对象的对应方法。

接下来的例子中，假设对一个协程的贴别调用中在协程栈上创建的promise对象为`promise`。

以及一个协程函数，函数体为`<body-statements>`，包含了3个关键字之一（`co_return`,`co_await`,`co_yield`），然后协程体会大致转义类似如下：

{% codeblock lang:c++ line_number:false %}
{
  co_await promise.initial_suspend();
  try
  {
    <body-statements>
  }
  catch (...)
  {
    promise.unhandled_exception();
  }
FinalSuspend:
  co_await promise.final_suspend();
}
{% endcodeblock %}

当一个协程函数被调用，会有一系列步骤在协程体之前执行，这和正常函数有一点不同。

这是概要的步骤（后续会进一步讲解这些步骤）：

1. 使用`operator new`来分配一个协程栈（可选）；
2. 拷贝任何函数参数到协程栈；
3. 调用promise对象类型`P`的构造函数；
4. 在协程第一次暂停时调用`promise.get_return_object()`方法获取结果来返回调用者。保存结果为局部变量；
5. 调用`promise.initial_suspend()`方法并`co_await`结果；
6. 当`co_await promise.initial_suspend()`表达式恢复（立即或者异步的），然后协程开始执行编码的协程体语句；

执行到`co_return`声明时会执行额外的步骤：

1. 调用`promise.return_void()`或者`promise.return_value(<expr>)`；
2. 按创建的倒序销毁所有自动存续期内的变量；
3. 调用`promise.final_suspend()`并`co_await`结果

如果执行是在`<body-statements>`中通过未处理异常返回的：

1. 在catch-block中捕获异常并调用`promise.unhandled_exception()`
2. 调用`promise.final_suspend()`并`co_await`结果

一旦执行传播到协程体之外，协程栈就会被销毁。销毁协程栈包括如下几步：

1. 调用promise对象额析构函数；
2. 调用复制的函数参数的析构；
3. 调用`operator delete`来释放协程栈使用的内存；
4. 转移执行回到调用者/恢复者

当在`co_await`表达式中的执行第一次遇到`<return-to-caller-or-resumer>`点时，或者协程执行到完成而没有遇到`<return-to-caller-or-resumer>`，然后协程要么挂起要么销毁，并且之前通过调用`promise.get_return_object()`的返回对象就返回给协程的调用者。

# 分配一个协程栈

首先编译器生成`operator new`的调用来给协程栈分配内存。

如果promise类型`P`自定义了`operator new`方法，那么调用自定义的，否则调用全局的`operator new`。

有一些需要注意的重点：

传入`operator new`的大小不是`sizeof(P)`，而是包括了整个协程栈的大小以及编译器根据参数个数和大小、promise对象大小、局部变量的个数与大小和编译器特定的用于管理携程状态的存储开销一起自动决定的。

编译器可以根据如下情形来优化掉`operator new`:

* 可以确定协程栈的生命周期严格嵌套在调用者内；
* 编译器可以在调用端看到协程栈的大小；

在这些情况中，编译器可以再调用者的栈帧上（要么是栈空间要么是协程栈内）为协程栈分配存储空间。

协程提案并没有指定任何情形分配内存的优化时一定会存在，所以仍然需要对协程栈可能的`std::bad_alloc`失败编写一些代码。这也意味着，通常不应该声明一个协程函数为`noexcept`，除非你能接受在协程栈分配内存失败时会被调用`std::terminate()`。

然而，有一类退化方法可以用来替代处理分配协程栈失败的异常。这在不允许异常的环境中非常有必要，例如嵌入式环境或者高性能环境，这些环境对于异常的开销是无法忍受的。

如果promise类型提供了一个静态的成员函数`P::get_return_object_on_allocation_failure()`，那么编译器就会生成重载`operator new(size_t, nothrow_t)`的调用。如果这个调用返回了`nullptr`然后协程会立即调用`P::get_return_object_on_allocation_failure()`，然后返回该结果给协程的调用者而不是抛出异常。

## 自定义协程栈的内存分配器

promise类型可以重载`operator new()`，在编译器需要为使用了你的promise类型的协程分配内存时，就会使用自定义的分配器来替代全局的`operator new`。

例如：
{% codeblock lang:c++ line_number:false %}
struct my_promise_type
{
  void* operator new(std::size_t size)
  {
    void* ptr = my_custom_allocate(size);
    if (!ptr) throw std::bad_alloc{};
    return ptr;
  }

  void operator delete(void* ptr, std::size_t size)
  {
    my_custom_free(ptr, size);
  }

  ...
};
{% endcodeblock %}

“这哪里是自定义了分配器？”，我听到了你的疑问。

你同时可以提供包括额外参数的`P::operator new()`重载，如果有合适的重载，就会被调用并传入协程函数参数的左值引用。这可被用于替代`operator new`来调用分配器上的`allocate()`方法，来作为参数传递给协程函数。

你需要做一些额外的工作在分配的内存中拷贝分配器，由此你可以在对应的`operator delete`调用中引用，因为并没有参数传递到对应的`operator delete`调用。这么做是因为这些参数存储在协程栈上，所以调用`operator delete`时他，他们已经被析构了。

例如，你可以实现`operator new`来在协程栈之后分配一些额外的空间，使用这些空间来暂存分配器的备份，最后可以用来释放协程栈的内存。

例如：

{% codeblock lang:c++ line_number:false %}
template<typename ALLOCATOR>
struct my_promise_type
{
  template<typename... ARGS>
  void* operator new(std::size_t sz, std::allocator_arg_t, ALLOCATOR& allocator, ARGS&... args)
  {
    // Round up sz to next multiple of ALLOCATOR alignment
    std::size_t allocatorOffset =
      (sz + alignof(ALLOCATOR) - 1u) & ~(alignof(ALLOCATOR) - 1u);

    // Call onto allocator to allocate space for coroutine frame.
    void* ptr = allocator.allocate(allocatorOffset + sizeof(ALLOCATOR));

    // Take a copy of the allocator (assuming noexcept copy constructor here)
    new (((char*)ptr) + allocatorOffset) ALLOCATOR(allocator);

    return ptr;
  }

  void operator delete(void* ptr, std::size_t sz)
  {
    std::size_t allocatorOffset =
      (sz + alignof(ALLOCATOR) - 1u) & ~(alignof(ALLOCATOR) - 1u);

    ALLOCATOR& allocator = *reinterpret_cast<ALLOCATOR*>(
      ((char*)ptr) + allocatorOffset);

    // Move allocator to local variable first so it isn't freeing its
    // own memory from underneath itself.
    // Assuming allocator move-constructor is noexcept here.
    ALLOCATOR allocatorCopy = std::move(allocator);

    // But don't forget to destruct allocator object in coroutine frame
    allocator.~ALLOCATOR();

    // Finally, free the memory using the allocator.
    allocatorCopy.deallocate(ptr, allocatorOffset + sizeof(ALLOCATOR));
  }
}
{% endcodeblock %}

要使用把`std::allocator_arg`作为第一个参数的自定义`my_promise_type`用于协程，需要特例化`coroutine_traits`类（下文会对`coroutine_traits`做更详细的解释）。

例如：

{% codeblock lang:c++ line_number:false %}
namespace std::experimental
{
  template<typename ALLOCATOR, typename... ARGS>
  struct coroutine_traits<my_return_type, std::allocator_arg_t, ALLOCATOR, ARGS...>
  {
    using promise_type = my_promise_type<ALLOCATOR>;
  };
}
{% endcodeblock %}

注意到，即使自定义了协程的内存分配策略，**编译仍然被允许忽略你的内存分配器**。

# 拷贝参数到协程栈

协程需要拷贝原始调用者传递给协程函数的任何参数到协程栈，由此在协程挂起后仍然是有效的。

如果参数是通过值传递到协程的，那么这些参数会通过调用类型的move构造函数来拷贝到协程栈。

如果参数通过引用传递到协程（包括左值和右值引用），那么只有引用拷贝到了协程栈，而不是指向的值。

注意到对于有trivial析构的类型，如果参数在`<return-to-caller-or-resumer>`点之后不再被引用，编译器完全可以跳过拷贝。

这里有很多陷进，当通过引用来传递参数给协程时，你不能在协程生命周期内完全依赖引用依旧有效。很多用于普通函数的常用技术，例如完美转发和全局引用，可能会在协程中导致为定义行为。Toby Allsopp写了一篇很棒的{% link 文章 https://toby-allsopp.github.io/2017/04/22/coroutines-reference-params.html %}来阐述这个话题。

如果在调用参数的拷贝/移动构造函数是抛出了异常，那么任何已经构造的参数就会被析构，协程栈会被释放，且该异常会被传回给调用者。

# 构造promise对象

一旦所有的参数已经被拷贝到了协程栈，然后协程就会构造promise对象。

参数的拷贝优先于promise对象构造的是为了允许promise对象可以在构造函数中访问被拷贝的参数。

首先编译器会检查promise对象是否有可以接收已拷贝参数左值引用的重载构造函数。如果有发现，那么编译器会生成该重载构造函数的调用。如果没找到，那么编译器会退化生成promise类型默认构造函数的调用。

注意到，promise构造函数可以“挑选”参数的能力和协程提案最近的修改有关，是在Jacksonville 2018 meeting中采用到{% link N4723 http://wg21.link/N4723 %}。可以在{% link P0914R1 http://wg21.link/P0914R1 %}中找到该提案。因此这个特性可能还没有被老版本的Clang后者MSVC支持。

如果promise在构造函数中抛出了异常，那么已拷贝的参数会被析构，且在异常传递给调用者之前，协程栈会在stack的退栈中被释放。

# 获取返回对象

对于promise对象协程首先要做的是通过调用`promise.get_return_object()`来获取`return-object`。

在协程第一次挂起或者在协程运行到完成且执行返回到调用者时，`return-object`是作为返回给协程函数调用者的值。

你可以参考控制流是（大体上）如下流转的：

{% codeblock lang:c++ line_number:false %}
// Pretend there's a compiler-generated structure called 'coroutine_frame'
// that holds all of the state needed for the coroutine. It's constructor
// takes a copy of parameters and default-constructs a promise object.
struct coroutine_frame { ... };

T some_coroutine(P param)
{
  auto* f = new coroutine_frame(std::forward<P>(param));

  auto returnObject = f->promise.get_return_object();

  // Start execution of the coroutine body by resuming it.
  // This call will return when the coroutine gets to the first
  // suspend-point or when the coroutine runs to completion.
  coroutine_handle<decltype(f->promise)>::from_promise(f->promise).resume();

  // Then the return object is returned to the caller.
  return returnObject;
}
{% endcodeblock %}

注意到我们需要在开始协程体之前获取返回对象，因为协程栈（也即promise对象）可能在调用`coroutine_handle::resume()`返回前已经被析构了，要么在当前线程要么在另外的线程，所以在开始协程体执行后调用`get_return_object()`是不安全的。

# 初始挂起点

一旦协程栈被初始化完以及返回对象已经被获取后，协程下一步要做的是执行`co_await promise.initial_suspend();`。

这允许`promise_type`的作者可以控制协程在执行协程体代码之前是应该挂起，还是立即执行。

如果协程在最初的挂起点挂起，那么在之后某刻就可以通过对协程的句柄(`coroutine_handle`)调用`resume()`或者`destroy()`来恢复或者销毁。

`co_await promise.initial_suspend()`表达式的返回结果会被丢弃，所以通常在等待者的`await_resume()`方法实现中返回`void`。

重要的是，`try/catch`块之外的声明保护了协程剩余的部分（如果忘了是如何定义的可以放回到前文协程体的定义）。这意味着在`co_await promise.initial_suspend()`求值中抛出的任何异常都早于`<return-to-caller-or-resumer>`点，会在销毁协程栈和返回对象之后抛回给协程的调用者。

如果你的`return-object`有在析构时销毁协程栈的RAII语义时需要特别注意。这种情况下你需要确保`co_await promise.initial_suspend()`是`noexcept`来避免协程栈的double-free。

> 注意到之前有一个提议来修复这样的语义，要求协程体中必须全部而不是部分的`co_await promise.initial_suspend()`表达式都位于`try/catch`块之内，因此在协程最终确定前这里的确切语义可能会发生变化。

对于很多协程类型，`initial_suspend()`方法要么返回`std::experimental::suspend_always`（在操作是惰性启动时），或者`std::experimental::suspend_never`（在操作是立即启动时），这两类都是`noexcept`的可等待类型所以通常情况这不是问题。

# 返回调用者

当协程函数运行到第一个`<return-to-caller-or-resumer>`点（或者没有这样的点那么是运行到完成），然后从`get_return_object()`调用中返回的`return-object`会被返回被协程的调用者。

注意`return-object`的类型并不一定需要和协程函数的返回类型一致。如果有必要，`return-obejct`会被隐式转换为协程的返回类型。

> 注意Clang(5.0)的协程实现中把该转换推迟到了return-object已经从协程调用中返回的时候，而MSVC(2017 Update 3)的实现中该转换会在调用`get_return_object()`时立即执行。虽然协程提案中并没有明确预期的行为，我相信MSVC已经计划改变他们的实现，而更接近Clang的版本，因为这种设计可以实现一些{% link 有趣的用例 https://github.com/toby-allsopp/coroutine_monad %}。

# 用`co_return`从协程中返回

当协程运行到`co_return`语句，会转义为要么调用`promise.return_void()`或者`promise.return_value(<expr>)`，接着是`goto FinalSuspend;`。

转义遵循如下规则：

* `co_return;`
  -> `promise.return_void();`
* `co_return <expr>;`
  -> `<expr>; promise.return_void();` if `<expr>` has type `void`
  -> `promise.return_value(<expr>);` if `<expr>` dose not have type `void`

随后的`goto FinalSuspend;`会首先引发自动存储周期的所有局部变量以构造顺序的逆序来析构，然后求值`co_await promise.final_suspend();`。

注意如果运行到协程结束也没有`co_return`语句那么就等价于在函数体最后是`co_return;`。在这种情况下，如果`promise_type`没有`return_void()`方法，那么行为就是未定义的。

如果`<expr>`的求值或者对`promise.return_void()`的调用，或者`promise.return_value()`，中抛出了异常，那么异常仍然会传递给`promise.unhandled_exception()`(参考下文)。

# 处理协程内透出的异常

如果异常从协程体内传播出，那么异常会被捕获，且`catch`块内的`promise.unhandled_exception()`方法会被调用。

该方法的实现中通常会调用`std::current_exception()`来捕获异常的一个副本来保存起来，用于之后在不同的上下文中重新抛出。

可选的是，实现中可以通过运行`throw;`语句来立即重新抛出异常。例如参考{% link folly::Optional https://github.com/facebook/folly/blob/4af3040b4c2192818a413bad35f7a6cc5846ed0b/folly/Optional.h#L587 %}。然而这么做（有可能 - 参考下文）会导致协程栈被立即析构并把异常传递给调用者/恢复者。这对于一些假设/要求调用`coroutine_handle::resume()`为`noexcept`的抽象会带来问题，所以你应该大体上只能在完全掌控谁或者什么来调用`resume()`时使用该方法。

注意当前的{% link Coroutines TS http://wg21.link/N4736 %}中对于调用`unhandled_exception()`重新抛出异常（或者任何try块之外的任何逻辑抛出一个异常）后预期行为的语法{% link 有一点不明确 https://github.com/GorNishanov/CoroutineWording/issues/17 %}。

我当前对该语法的解释是如果在协程体中有控制，要么通过把异常抛出`co_await promise.initial_suspend()`，`promise.unhandled_exception()`或者`co_await promise.final_suspend()`或者当协程运行到完成来通过`co_await p.final_suspend()`来完成同步，然后在返回调用者/恢复者之前协程栈会自动析构。然而这样的解释也有问题。

未来版本的协程说明有希望会清晰的阐述该场景。然而目前为止，我会说保持避免在`initial_suspend()`, `final_suspend()` 或者 `unhandled_exception()`中抛出异常。敬请期待！

# 最后的挂起点

一旦运行从用户定义的协程体中退出，且结果已经通过调用`return_void()`，`return_value()`或者`unhandled_exception()`捕获，以及所有的局部变量已经被析构，这时候在返回到调用者/恢复者之前协程有一个机会来运行一些额外的逻辑。

协程会运行`co_await promise.final_suspend();`语句。

这允许了协程来运行一些逻辑，例如通知结果，发送完成的信号或者恢复继续执行。这同样允许了协程可以在运行到结束前且协程栈被销毁前立即挂起。

注意`resume()`一个在`final_suspend`点挂起的协程行为是未定义的。在这个点挂起的协程你只能对其执行`destroy()`。

根据Gor Nishanov，这个限制的理由是，编译器可以有机会做一些优化，因为协程对应的挂起状态的数量变少了，以及潜在的所需的分支也变少了。

注意虽然一个协程不能在`final_suspend`点挂起，但是**推荐对协程结构化进而在可能的时候在`final_suspend`点挂起**。因为这样强制了需要在协程之外调用`.destroy()`（一般是通过RAII对象的析构函数）且这会让编译器更容易发现协程栈的生命周期是否嵌套在调用者。这进一步使得编译器可以省略协程栈的内存分配。

# 编译器如何选择promise类型

让我们来看下编译器如何决定一个给定协程的promise对象类型。

promise对象的类型可以通过使用`std::experimental::coroutine_traits`类来从协程的签名中确定。

如果有一个协程函数的签名如下：

{% codeblock lang:c++ line_number:false %}
task<float> foo(std::string x, bool flag);
{% endcodeblock %}

然后编译器会通过把返回类型和参数类型作为模板参数传递给`coroutine_traits`来推导协程的promise类型。

{% codeblock lang:c++ line_number:false %}
typename coroutine_traits<task<float>, std::string, bool>::promise_type;
{% endcodeblock %}

如果函数是非静态的成员函数，那么类的类型会作为第二个模板参数传递给`coroutine_traits`。注意如果你的方法是右值引用的重载，那么第二个模板参数就是右值引用的。

例如，如果有如下类型：

{% codeblock lang:c++ line_number:false %}
task<void> my_class::method1(int x) const;
task<foo> my_class::method2() &&;
{% endcodeblock %}

然后编译器会使用如下promise类型：

{% codeblock lang:c++ line_number:false %}
// method1 promise type
typename coroutine_traits<task<void>, const my_class&, int>::promise_type;

// method2 promise type
typename coroutine_traits<task<foo>, my_class&&>::promise_type;
{% endcodeblock %}

`coroutine_traits`模板的默认通过查找返回类型上嵌套的`promise_type`typedef来定义`promise_type`。如下面的例子（但是在一些特殊的SFINAE魔法中如果`RET::promise_type`没有定义那么`promise_type`也是未定义的）。

{% codeblock lang:c++ line_number:false %}
namespace std::experimental
{
  template<typename RET, typename... ARGS>
  struct coroutine_traits<RET, ARGS...>
  {
    using promise_type = typename RET::promise_type;
  };
}
{% endcodeblock %}

对于可以控制协程的返回类型的情况，所以可以通过在类内定义一个嵌套的`promise_type`，来使编译器把这个类型作为协程的promise对象的类型。

例如：

{% codeblock lang:c++ line_number:false %}
template<typename T>
struct task
{
  using promise_type = task_promise<T>;
  ...
};
{% endcodeblock %}

然而对于不能控制协程返回类型时，你可以特例化`coroutine_traits`来定义promise类型而不用修改该类型。

例如，定义使用`std::optional<T>`作为promise类型的协程：

{% codeblock lang:c++ line_number:false %}
namespace std::experimental
{
  template<typename T, typename... ARGS>
  struct coroutine_traits<std::optional<T>, ARGS...>
  {
    using promise_type = optional_promise<T>;
  };
}
{% endcodeblock %}

# 识别特定协程的栈帧

当调用一个协程函数时，协程栈会被创建。为了恢复对应的协程或者销毁协程栈，你需要能识别或者找到特定的协程栈。

协程提案中提供的机制是`coroutine_handle`类型。

该类型的精简接口如下：

{% codeblock lang:c++ line_number:false %}
namespace std::experimental
{
  template<typename Promise = void>
  struct coroutine_handle;

  // Type-erased coroutine handle. Can refer to any kind of coroutine.
  // Doesn't allow access to the promise object.
  template<>
  struct coroutine_handle<void>
  {
    // Constructs to the null handle.
    constexpr coroutine_handle();

    // Convert to/from a void* for passing into C-style interop functions.
    constexpr void* address() const noexcept;
    static constexpr coroutine_handle from_address(void* addr);

    // Query if the handle is non-null.
    constexpr explicit operator bool() const noexcept;

    // Query if the coroutine is suspended at the final_suspend point.
    // Undefined behaviour if coroutine is not currently suspended.
    bool done() const;

    // Resume/Destroy the suspended coroutine
    void resume();
    void destroy();
  };

  // Coroutine handle for coroutines with a known promise type.
  // Template argument must exactly match coroutine's promise type.
  template<typename Promise>
  struct coroutine_handle : coroutine_handle<>
  {
    using coroutine_handle<>::coroutine_handle;

    static constexpr coroutine_handle from_address(void* addr);

    // Access to the coroutine's promise object.
    Promise& promise() const;

    // You can reconstruct the coroutine handle from the promise object.
    static coroutine_handle from_promise(Promise& promise);
  };
}
{% endcodeblock %}

你可以有两种方法来获取一个协程的`coroutine_handle`:

1. 在`co_await`表达式中它会被传递给`await_suspend()`方法；
2. 如果你有协程promise类型的引用，你可以通过`coroutine_handle<Promise>::from_promise()`来重新构建出`coroutine_handle`;

协程在`co_await`表达式中的`<suspend-point>`挂起后，把`coroutine_handle`会被传递到等待着的`await_suspend()`方法。你可以把`coroutine_handle`作为继续协程的一个{% link continuation-passing style https://en.wikipedia.org/wiki/Continuation-passing_style %}调用。

注意`coroutine_handle`并不是RAII对象。你可以手工调用`.destroy()`来销毁协程栈并释放资源。可以把他作为管理内存的一个`void*`指针。这是为了性能考虑：如果做成RAII对象，会给协程增加额外的开销，例如需要维护引用计数。

你通常需要尝试使用高层了类型来给协程提供RAII语义，例如使用{% link coppcoro https://github.com/lewissbaker/cppcoro %}中提供的类型，或者自己编写高层类型囊括协程栈的整个生命周期。

# 自定义`co_await`的行为

promise类型可选的可以自定义协程体中每一个`co_await`表达式的行为。

可以很简单的通过为promise类型定义`await_transform()`，编译器就会把协程体中每一个`co_await <expr>`转义为`co_await promise.await_transform(<expr>)`。

这可以实现很多重要且有用的场景：

**可以实现等待类型不像通常的变为可等待的。**

例如协程的promise类型拥有`std::optional<T>`返回类型，可以提供一个`await_transform()`重载并接收`std::optional<U>`，那么就会返回一个可等待类型，要么返回类型为`U`的值，或者在等待值为`nullopt`时挂起协程。

{% codeblock lang:c++ line_number:false %}
template<typename T>
class optional_promise
{
  ...

  template<typename U>
  auto await_transform(std::optional<U>& value)
  {
    class awaiter
    {
      std::optional<U>& value;
    public:
      explicit awaiter(std::optional<U>& x) noexcept : value(x) {}
      bool await_ready() noexcept { return value.has_value(); }
      void await_suspend(std::experimental::coroutine_handle<>) noexcept {}
      U& await_resume() noexcept { return *value; }
    };
    return awaiter{ value };
  }
};
{% endcodeblock %}

**可以通过声明`await_transform`重载为`deleted`来阻止等待特定类型**

例如返回类型为`std::generator<T>`的promise类型可以声明一个接收任何类型的deleted的`await_transform()`模板成员函数。这就阻止了对在该协程内使用`co_await`。

{% codeblock lang:c++ line_number:false %}
template<typename T>
class generator_promise
{
  ...

  // Disable any use of co_await within this type of coroutine.
  template<typename U>
  std::experimental::suspend_never await_transform(U&&) = delete;

};
{% endcodeblock %}

**可以实现适配且改变通常可等待值的行为。**

例如，你可以通过包装`resume_on()`操作符内的可等待实现，在定义一个协程类型确保协程总是会在一个相关的执行者上从`co_await`表达式恢复（参考`cppcoro::resume_on()`）。

{% codeblock lang:c++ line_number:false %}
template<typename T, typename Executor>
class executor_task_promise
{
  Executor executor;

public:

  template<typename Awaitable>
  auto await_transform(Awaitable&& awaitable)
  {
    using cppcoro::resume_on;
    return resume_on(this->executor, std::forward<Awaitable>(awaitable));
  }
};
{% endcodeblock %}

最后对`await_transform()`提一点，需要特别注意的是如果promise类型定义了任何`await_transform()`成员，那么就会触发编译器对所有的`co_await`表达式转义为调用`promise.await_transform()`。这意味着如果你只想自定义一部分`co_await`的行为，你需要提供一个退化版本的`await_transform()`来直接转发这些参数。

# 自定义`co_yield`的行为

最后一个可以通过promise类型自定义的是`co_yield`关键字的行为。

如果`co_yield`关键字出现在协程中，那么编译器会把`co_yield <expr>`转义为`co_await promise.yield_value(<expr>)`。promise类型因此可以通过对promise对象定义一个或者多个`yield_value()`方法来自定义`co_yield`关键字的行为。

注意，不像`await_transform`，promise类型在没有定义`yield_value()`方法时对`co_yield`没有默认行为。所以promise类型可以通过声明一个`deleted`的`await_transform()`来明确不允许`co_await`，相反promise类型需要特地的声明来支持`co_yield`。

promise类型定义`yield_value()`方法的典型例子为`generator<T>`类型：

{% codeblock lang:c++ line_number:false %}
template<typename T>
class generator_promise
{
  T* valuePtr;
public:
  ...

  std::experimental::suspend_always yield_value(T& value) noexcept
  {
    // Stash the address of the yielded value and then return an awaitable
    // that will cause the coroutine to suspend at the co_yield expression.
    // Execution will then return from the call to coroutine_handle<>::resume()
    // inside either generator<T>::begin() or generator<T>::iterator::operator++().
    valuePtr = std::addressof(value);
    return {};
  }
};
{% endcodeblock %}

# 总结

在本篇中，我描述了编译器在编译协程时要做的每一个独立转换。

希望本篇有助于你理解如何通过定义不同的promise类型来自定义不同协程类型的行为。协程机制中有很多可以灵活配置的，所以你可以有很多方法来自定义他们的行为。

然而还有一个编译器会做的更重要的转换还没涉及，如何把协程体转义为状态机。本篇已经足够长了所以我会放在下一篇中讲解。保持期待！