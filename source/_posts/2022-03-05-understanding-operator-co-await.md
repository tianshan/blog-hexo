---
title: "【翻译】C++ Coroutines: understanding-operator-co-await"
date: 2022-03-05 16:08:30
tags: coroutines
---

这是*Lewis Baker*讲解协程的第二篇，主要介绍
* awaite，awaiter相关概念
* awaite细节机制，包括c++
* 实现awaite对象的例子
原文：https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await

* {% post_link 2022-02-07-coroutine-theory "第一篇 Coroutine Theory" %}

在上一篇中，我描述了函数和协程的高层区别，但是没有深入{% link C++协程提案 http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4680.pdf %}中描述的语法和语义细节。

该协程提案中给C++添加的关键点是挂起一个协程的能力，并允许稍后被恢复。提案中提供的机制是新的操作`co_await`。

理解`co_await`是如何工作的，有助于弄清楚协程的行为，以及是如何挂起以及恢复的。本篇中会解释`co_await`操作的机制，并介绍相关的`Awaitable`和`Awaiter`类型概念。

<!-- more -->

在深入`co_await`前，我想通过对协程提案的概要介绍来提供一些上下文。

# 协程提案提供了哪些？

* 3个语言关键字，`co_await`, `co_yield`, `co_return`
* 一些新的类型：
    * coroutine_handle<P>
    * coroutine_traits<Ts...>
    * suspend_always
    * suspend_never
* 库编写者可以和协程交互以及定义行为的通用机制
* 编写异步代码非常容易的语言层工具

C++协程提案提供的语言层工具，可以认为是协程的底层汇编语言。这些工具很难安全的直接使用，主要是提供给库编写者来构建更高层的抽象，来使应用开发者更安全的使用。

这些基础工具计划在即将到来的语言标准中发表（协程定义已包含在C++20标准中），包括标准库中的一些高层类型，封装了底层构建块以及可以让应用开发者更安全编写协程。

# 编译器和库之间的交互

有趣的是，协程提案并没有准确的定义协程的语义。并没有定义如何使用返回给调用者的值。没有定义传递给`co_return`语句的返回值该怎么处理，或者如何处理携程中异常的传播。没有定义应该在什么线程上恢复协程。

相反，定义了库代码定制协程行为的一个通用机制，具体通过实现遵从特定接口的类型。编译器就可以对调用库提供的类型实例来生成代码。这个方法类似于库编码者可以通过实现`iterator`类型的`begin()/end()`方法来定制循环范围的行为。

事实上，协程提案没有明确协程的特定语义使得其成为一个强有力的工具。允许了库编写者可以定义很多支持不同目的，不同类型的协程。

例如，可以定义一个协程，用于异步的产生一个单一值，或者用于惰性产生一组值，或者用于简化消费`optional<T>`类型值的控制流，通过返回`nullopt`来提前返回。

协程提案中定义了两类接口：`Promise`接口和`Awaitable`接口。

`Promise`接口指定了定制协程本身行为的方法。库编写者可以定制协程被调用时如何工作，以及协程返回（通过正常方法或者未处理的异常）时如何工作，并且定制协程中任何`co_await`或者`co_yield`表达式的行为。

`Awaitable`接口指定了控制`co_await`表达式语义的方法。当一个值通过`co_await`返回，代码会翻译为一系列对awaitable对象的调用，并允许指定：是否挂起当前协程，在协程挂起后执行一些逻辑，以及在协程恢复处理`co_await`表达式返回值后执行一些逻辑。

我会在下一篇中细探`Promise`接口，当前来深入探索下`Awaitable`接口。

# Awaiters和Awaitables：解释`operator co_await`

`co_await`操作符是可以应用到一个值上的新一元操作符。例如`co_await someValue`。

`co_await`操作符尽可以在协程中使用。这虽然是一个赘述，因为任何一个函数包含了`co_await`操作符，从定义上来说，就会被编译为协程。

一个支持`co_await`操作符的类型就称为`Awaitable`类型。

注意到`co_await`操作符是否能应用到一个类型，取决于`co_await`表达式出现的上下文。协程中`promise`类型通过`await_transform`方法就可以用来修改协程中`co_await`表达式的含义（后续会更多的解释）。

为了更具体的解释，我用术语`Normally Awaitable`来描述一个在协程上下文可以支持`co_await`操作符的类型，这类类型的promise type不包含`await_transform`成员。同时，我用术语`Contextually Awaitable`来描述仅支持在协程的promise type中包含`await_transform`方法的上下文中使用`co_await`操作符的类型。

`Awaiter`类型是一类实现了3个特殊方法的类型，`await_ready`, `await_suspend`和`await_resume`，这3个方法被称为称为`co_await`表达式的组成。

注意到，我从C#的`async`关键字机制中借用了`Awaiter`术语，机制中通过`GetAwaiter()`方法返回了一个对接，拥有非常类似C++ Awaiter概念的接口。参考{% link 这篇文章 https://weblogs.asp.net/dixin/understanding-c-sharp-async-await-2-awaitable-awaiter-pattern %}获得更多C# awaiters的细节。

注意一个类型既可以是`Awaitable`又可以是`Awaiter`。

当编译器遇到`co_await <expr>`表达式，根据具体的类型会转换为很多可能的结果。

## 获取Awaiter

编译器首先做的事情就是给等待变量(awaited value)生成代码来获取`Awaiter`对象。协程提案的5.3.8(3)章节对获取等待对象设置了多个步骤。

让我们假设等待协程(awaiting coroutine)的promise对象(promise object)的类型为`P`，且promise是当前协程promise对象的一个左值引用。

如果promise类型`P`有一个成员`await_transform`，那么`<expr>`首先会调用为`promise.await_transform(<expr>)`来获取**可等待**变量`awaitable`。否则，如果promise类型不包含`await_transform`成员，那么我们将`<expr>`直接作为**可等待**对象`awaitable`。

然后，如果**可等待**对象`awaitable`有可应用的重载`operator co_await()`，然后就会被调用来获取**可等待**对象。否则对象`awaitable`就作为等待对象。

如果我们把这些函数封装到`get_awaitable()`和`get_awaiter()`中，可能会如下所示。

{% codeblock lang:c++ line_number:false %}
template<typename P, typename T>
decltype(auto) get_awaitable(P& promise, T&& expr)
{
  if constexpr (has_any_await_transform_member_v<P>)
    return promise.await_transform(static_cast<T&&>(expr));
  else
    return static_cast<T&&>(expr);
}

template<typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable)
{
  if constexpr (has_member_operator_co_await_v<Awaitable>)
    return static_cast<Awaitable&&>(awaitable).operator co_await();
  else if constexpr (has_non_member_operator_co_await_v<Awaitable&&>)
    return operator co_await(static_cast<Awaitable&&>(awaitable));
  else
    return static_cast<Awaitable&&>(awaitable);
}
{% endcodeblock %}

## Awaiting the Awaiter

所以，假定我们把转换`<expr>`结果为**等待**对象的逻辑囊括在了上述函数中，那么`co_await <expr>`的语义可以大致被翻译如下：

{% codeblock lang:c++ line_number:false %}
{
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready())
  {
    using handle_t = std::experimental::coroutine_handle<P>;

    using await_suspend_result_t =
      decltype(awaiter.await_suspend(handle_t::from_promise(p)));

    <suspend-coroutine>

    if constexpr (std::is_void_v<await_suspend_result_t>)
    {
      awaiter.await_suspend(handle_t::from_promise(p));
      <return-to-caller-or-resumer>
    }
    else
    {
      static_assert(
         std::is_same_v<await_suspend_result_t, bool>,
         "await_suspend() must return 'void' or 'bool'.");

      if (awaiter.await_suspend(handle_t::from_promise(p)))
      {
        <return-to-caller-or-resumer>
      }
    }

    <resume-point>
  }

  return awaiter.await_resume();
}
{% endcodeblock %}

当调用返回值是`void`版本的`await_suspend()`返回时，即会无条件转移执行权回到协程的调用者/恢复者；而返回`bool`的版本允许等待对象有条件的立即恢复协程而不用返回调用者/恢复者。

当返回`bool`版本的`await_suspend()`在有些场景比较有用，例如等待者开始一个异步操作后有时候可以直接同步返回。当同步返回时，`await_suspend()`方法可以返回`false`来指示协程被立即恢复了且继续执行。

在代码中`<suspend-coroutine>`点，编译器生成了当前协程的状态，并为恢复做准备。包括存储`<resume-point>`的位置，以及保存当前寄存器中值到协程栈内存中。

当前协程在`<suspend-coroutine>`操作完成后即认为是挂起状态。观察挂起协程的第一个位置可以在调用`await_suspend()`之内。一旦协程挂起后即可被恢复或者销毁。

在后续一旦操作完成后，`await_suspend()`方法负责调用协程来恢复（或者销毁）。注意到`await_suspend()`返回`false`时意味着在当前线程中立即调度恢复协程。

`await_ready()`方法的目的是为允许在某些场景避免`<suspend-coroutine>`操作的开销，如上操作会同步完成而不需要挂起的时候。

在代码`<return-to-caller-or-resumer>`中，执行权被转移回调用者/恢复者，弹出本地栈帧但是协程栈依旧保留存活。

在挂起中的协程最终恢复，然后执行点会恢复到`<resume-point>`。例如就在`await_resume()`方法被调用获取操作结果之前。

`await_resume()`方法的调用得到的返回值会变成`co_await`表达式的结果。`await_resume()`方法同样也会把异常抛出，这种情况异常会从`co_await`表达式传播出。

注意到如果异常从`await_suspend()`中抛出，协程会自动恢复，且不需要调用`await_resume()`异常就可以从`co_await`表达式传出。

# 协程句柄

可能已经注意到`coroutine_handle<P>`类型是通过传递给`co_await`表达式的`await_suspend()`调用中来使用的。

该类型代表了协程栈的一个非拥有句柄，可以用来恢复协程的执行或者销毁协程栈。并且可以用来访问协程的promise对象。

`coroutine_handle`类型有如下（最小）接口：

{% codeblock lang:c++ line_number:false %}
namespace std::experimental
{
  template<typename Promise>
  struct coroutine_handle;

  template<>
  struct coroutine_handle<void>
  {
    bool done() const;

    void resume();
    void destroy();

    void* address() const;
    static coroutine_handle from_address(void* address);
  };

  template<typename Promise>
  struct coroutine_handle : coroutine_handle<void>
  {
    Promise& promise() const;
    static coroutine_handle from_promise(Promise& promise);

    static coroutine_handle from_address(void* address);
  };
}
{% endcodeblock %}

当实现**可等待**类型时，`coroutine_handle`上使用的关键方法是`.resume()`，在操作完成且需要恢复等待中的协程时调用。在`coroutine_handle`上调用`.resume()`会重新激活一个在`<resume-point>`挂起的协程。`.resume()`的调用会在协程下一次执行到`<return-to-caller-or-resumer>`时返回。

`.destroy()`方法用于销毁协程栈，调用所有存活变量的析构方法，并释放协程栈使用的内存。一般来说你不需要（确切来说是避免）调用`.destroy()`，除非你是在编写库实现协程的promise对象。通常的，协程栈属于调用协程时返回的某类RAII类型。所以直接调用`.destroy()`而不和RAII对象配合会导致两次析构问题。

`.promise()`方法返回协程的promise对象的一个引用。然而，像`.destroy()`一样，通常仅在拥有协程promise类型时有效。你需要把协程的promise对象当做为协程的内部协程实现细节。对于大部分**通常可等待**类型，需要使用`coroutine_handle<void>`作为参数类型用在`await_suspend()`方法而不是`coroutine_handle<Promise>`。

`coroutine_handle<P>::from_promise(P& promise)`函数允许基于协程的promise对象引用来重新重新构造协程句柄。注意到需要你确保类型`P`和协程栈的promise类型实体完全匹配；如果用`coroutine_handle<Base>`来构造基于Base派生的`Derived`promise类型实体会导致未知行为。

`.address()`/`from_address()`函数允许转换协程句柄为一个`void*`指针。主要是用来支持传递上下文参数到现有的C-style APIs，所以你可以发现在一些环境中对于实现**可等待**对象很有用。然而，大部分情况我发现这对于在回调的上下文参数中传递额外信息非常有用，所以我总是最终把`coroutine_handle`保存在结构中并以一个指针传递给上下文参数而不是使用`.address()`返回值。

# 无需同步动作的异步代码

`co_await`操作的一个强有力设计特点是在协程挂起后及返回调用者/恢复者之前有能力执行代码。

这允许了等待对象在协程挂起后初始化一个异步操作，传递挂起协程的`coroutine_handle`给操作，就可以在操作完成时不需要任何额外同步动作就安全的恢复（也有可能在另外的线程）

例如，当协程已经挂起后，在`await_suspend()`中发起一个异步读操作，我们可以在操作完成时即可恢复协程，而不需要任何线程同步动作来协同发起操作的线程和完成操作的协程。

{% codeblock lang:c++ line_number:false %}
Time     Thread 1                           Thread 2
  |      --------                           --------
  |      ....                               Call OS - Wait for I/O event
  |      Call await_ready()                    |
  |      <supend-point>                        |
  |      Call await_suspend(handle)            |
  |        Store handle in operation           |
  V        Start AsyncFileRead ---+            V
                                  +----->   <AsyncFileRead Completion Event>
                                            Load coroutine_handle from operation
                                            Call handle.resume()
                                              <resume-point>
                                              Call to await_resume()
                                              execution continues....
           Call to AsyncFileRead returns
         Call to await_suspend() returns
         <return-to-caller/resumer>
{% endcodeblock %}

需要特别注意的是，当利用这种方法时，只要开始了这个操作即传递协程句柄给其他的线程，那么另外线程可以在`await_suspend()`返回之前恢复协程，并且可以继续并发的继续执行  `await_suspend()`方法剩余部分。

协程恢复时首先要做的是调用`await_resume()`来获得结果，然后通常会立即销毁**等待**对象（例如`await_suspend()`调用的`this`指针）。在`await_suspend()`返回之前，协程然后可能会运行至结束，销毁协程和promise对象。

所以在`await_suspend()`方法之内，一旦协程可以在另外线程并发地恢复时，你需要确保避免访问`this`或者该协程的`.promise()`对象，因为这两个可能已经被销毁了。总的来说，在操作开始后且协程已经被调用要恢复时，唯一可以安全访问的是`await_suspend()`内的本地变量。

# 对比Stackful协程

我会花点时间来通过协程挂起后执行逻辑的能力对比，来比较C++协程提案stackless协程和其他现有的stackful协程，比如Win32 fibers或者boost::context。

对于很多stackful协程框架，协程的挂起和另外协程的恢复是结合在'上下文切换'操作中的。在该'上下文切换'中，通常没有机会在当前协程挂起后以及转移执行到另外协程前执行逻辑。

这意味着，如果我们要在stackful协程上实现一个类似的异步文件读操作，需要在挂起协程之前开始这个操作。因此在协程挂起且准备好恢复前，这个操作可能已经在其他线程上完成了。另外线程上的操作完成以及协程挂起之间潜在的竞争，就需要一些线程同步操作来做协调。

可能有很多方法来实现这样的需求，例如通过使用一个临时上下文（trampoline context）代表起初的上下文在它被挂起后再发起操作。然后这就要求有额外的基础设施和一个额外的上下文切换来使其工作，并且有可能引入的开销要比想要避免的同步开销更大。

# 避免内存分配

异步操作通常需要按操作保存一些状态来跟踪进度。这样的状态通常需要在整个操作周期内维持，且只能在操作完成后释放。

例如，调用异步Win32 I/O函数，需要分配并传递一个`OVERLAPPED`结构的指针。调用者负责确保指针在操作完成前都是有效的。

对于传统的回调类APIs，这类状态通常需要在堆上分配来确保有合适的生命周期。如果你需要执行很多操作，可能需要对每个操作都分配以及释放这个状态。如果需要考虑性能，那么就需要定制分配器从内存池中分配这些状态对象。

然而当我们使用协程时，通过利用协程栈内的临时变量会在协程挂起时依旧保持存活的特性，我们可以避免从堆上为操作分配状态。

通过将操作的状态放在**等待**对象中，我们可以有效的从协程栈上”借用“内存，来保存操作状态且持续到`co_await`表达式的周期。

一旦操作完成，协程会被恢复，**等待**对象会被销毁，以及释放协程栈上其他本地变量使用的内存。

最终，协程栈可能还是在堆上分配的。但是一旦分配后，协程栈可以用这唯一的堆内存来执行很多异步操作。

如果你仔细想下，协程栈就像一种高性能内存分配器。编译器会在编译阶段就计算好所有局部变量需要的内存大小，然后就可以零开销来给局部变量分配内存！可以尝试用自定义分配器来打败他:)

# 样例：实现一个线程同步原语

现在我们已经介绍了`co_await`操作符的很多机制，我想通过应用一些知识到实践来实现一个基础的可等待同步原语：一个异步手工重置的事件。

事件实现的基础要求是可以被多个并发执行协程**等待**，以及当处于等待状态时需要挂起等待协程，直到一些线程调用`.set()`方法，这时任意等待协程都需要被恢复。如果一些线程调用了`.set()`，那么协程应该继续执行而不用挂起。

理想情况下，我们需要`noexcept`，也就要求没有堆分配，以及是无锁实现。

参考用法如下：

{% codeblock lang:c++ line_number:false %}
T value;
async_manual_reset_event event;

// A single call to produce a value
void producer()
{
  value = some_long_running_computation();

  // Publish the value by setting the event.
  event.set();
}

// Supports multiple concurrent consumers
task<> consumer()
{
  // Wait until the event is signalled by call to event.set()
  // in the producer() function.
  co_await event;

  // Now it's safe to consume 'value'
  // This is guaranteed to 'happen after' assignment to 'value'
  std::cout << value << std::endl;
}
{% endcodeblock %}

我们首先考虑下event可能的状态：'not set' 以及 'set'。

当处于'not set'状态时，会有一组（也可能是空的）等待中的协程在等待变为'not'。

当处于'set'状态时，不会有任何等待协程，如果有协程在该状态下需要`co_await`事件那么就会继续执行而不需要挂起。

这个状态事实上可以用一个`std::atomic<void*>`来表示：

* 保存一个特殊指针用于'set'状态。本例中我们会使用事件的`this`指针，因为我们知道和列表中项目的地址都不重复
* 事件处于'not set'状态时，该值代表指向一个等待协程linked-list结构的指针；

我们可以通过将nodes存储在‘等待者’对象的协程栈中来为堆上的linked-list避免额外的nodes分配。

那么我们就会得到一个如下类接口：

{% codeblock lang:c++ line_number:false %}
class async_manual_reset_event
{
public:

  async_manual_reset_event(bool initiallySet = false) noexcept;

  // No copying/moving
  async_manual_reset_event(const async_manual_reset_event&) = delete;
  async_manual_reset_event(async_manual_reset_event&&) = delete;
  async_manual_reset_event& operator=(const async_manual_reset_event&) = delete;
  async_manual_reset_event& operator=(async_manual_reset_event&&) = delete;

  bool is_set() const noexcept;

  struct awaiter;
  awaiter operator co_await() const noexcept;

  void set() noexcept;
  void reset() noexcept;

private:

  friend struct awaiter;

  // - 'this' => set state
  // - otherwise => not set, head of linked list of awaiter*.
  mutable std::atomic<void*> m_state;

};
{% endcodeblock %}

这里我们给了一个非常直观的接口。最需要注意的是，这里包含了`operator co_await()`方法，会返回一个还没定义类型的`awaiter`。

让我们现在来定义`awaiter`类型。

## 定义等待者

首先，需要知道哪一个`async_manual_reset_event`对象即将变成等待中，所以需要一个事件的引用以及构造函数来初始化。

同时需要拥有持有`awaiter`值的linked-list的node，以此来获得列表中下一个`awaiter`对象的指针。

以及需要存储执行了`co_await`表达式的等待协程的`coroutine_handle`，以便于该事件可以在变成'set'时恢复协程。我们不需要在意协程的promise类型，所以只需要使用`coroutine_handle<>`(是`coroutine_handle<void>`的缩写)。

最终，需要实现**等待者**接口，包括3个特别方法：`await_ready`，`await_suspend`和`await_resume`。我们不需要从`co_await`表达式返回值，所以`await_resume`可以返回`void`。

一旦我们实现了以上全部，`awaiter`的基础类接口就如下：

{% codeblock lang:c++ line_number:false %}
struct async_manual_reset_event::awaiter
{
  awaiter(const async_manual_reset_event& event) noexcept
  : m_event(event)
  {}

  bool await_ready() const noexcept;
  bool await_suspend(std::experimental::coroutine_handle<> awaitingCoroutine) noexcept;
  void await_resume() noexcept {}

private:

  const async_manual_reset_event& m_event;
  std::experimental::coroutine_handle<> m_awaitingCoroutine;
  awaiter* m_next;
};
{% endcodeblock %}

现在当我们`co_await`一个事件时，如果已经处于'set'状态就不需要等待协程挂起。所以我们可以定义`await_ready()`在事件已经'set'时返回`true`。

{% codeblock lang:c++ line_number:false %}
bool async_manual_reset_event::awaiter::await_ready() const noexcept
{
  return m_event.is_set();
}
{% endcodeblock %}

下一步，让我们看向`await_suspend()`方法。通常这里是可等待类型出现魔法的地方。

首先需要暂存等待协程的协程句柄到`m_awaitingCoroutine`成员，用于之后可以调用`.resume()`。

然后一旦我们完成后，需要尝试自动把等待者入队到等待者队列。如果成功入队，则返回`true`来代表我们不需要立即恢复协程，否则当我们发现事件已经被并发的设置为'set'状态就返回`false`来代表协程可以被立即恢复。

{% codeblock lang:c++ line_number:false %}
bool async_manual_reset_event::awaiter::await_suspend(
  std::experimental::coroutine_handle<> awaitingCoroutine) noexcept
{
  // Special m_state value that indicates the event is in the 'set' state.
  const void* const setState = &m_event;

  // Remember the handle of the awaiting coroutine.
  m_awaitingCoroutine = awaitingCoroutine;

  // Try to atomically push this awaiter onto the front of the list.
  void* oldValue = m_event.m_state.load(std::memory_order_acquire);
  do
  {
    // Resume immediately if already in 'set' state.
    if (oldValue == setState) return false; 

    // Update linked list to point at current head.
    m_next = static_cast<awaiter*>(oldValue);

    // Finally, try to swap the old list head, inserting this awaiter
    // as the new list head.
  } while (!m_event.m_state.compare_exchange_weak(
             oldValue,
             this,
             std::memory_order_release,
             std::memory_order_acquire));

  // Successfully enqueued. Remain suspended.
  return true;
}
{% endcodeblock %}

注意到我们在读取旧状态时使用的`acquire`内存序，因此如果读到特殊的'set'值，然后我们可以读到调用'set()'之前发生的写。

我们在比较修改写成功时使用了'release'内存序，因此之后的'set()'调用可以看到我们对`m_awaitingCoroutine`的更新以及之前对协程状态的写入。

## 填充event类的剩余部分

现在我们定义好了`awaiter`类型，再来回看`async_manual_reset_event`方法的实现。

首先是构造函数，需要构造为'not set'状态配上空的等待者队列（例如，nullptr）或者初始化为'set'状态（例如，'this'）。

{% codeblock lang:c++ line_number:false %}
async_manual_reset_event::async_manual_reset_event(
  bool initiallySet) noexcept
: m_state(initiallySet ? this : nullptr)
{}
{% endcodeblock %}

下一步，`is_set()`方法很直接的就判断是否为`this`来判断'set'状态：

{% codeblock lang:c++ line_number:false %}
bool async_manual_reset_event::is_set() const noexcept
{
  return m_state.load(std::memory_order_acquire) == this;
}
{% endcodeblock %}

再下一步是`reset()`方法。如果是'set'状态就需要转换状态为'not set'并清空等待者队列，否则的话不用动。

{% codeblock lang:c++ line_number:false %}
void async_manual_reset_event::reset() noexcept
{
  void* oldValue = this;
  m_state.compare_exchange_strong(oldValue, nullptr, std::memory_order_acquire);
}
{% endcodeblock %}

在`set()`方法内，我们要通过交换当前状态和特殊'set'值`this`来设置转换状态，然后来检查旧值。如果有任何等待协程就会在返回之前按序恢复。

{% codeblock lang:c++ line_number:false %}
void async_manual_reset_event::set() noexcept
{
  // Needs to be 'release' so that subsequent 'co_await' has
  // visibility of our prior writes.
  // Needs to be 'acquire' so that we have visibility of prior
  // writes by awaiting coroutines.
  void* oldValue = m_state.exchange(this, std::memory_order_acq_rel);
  if (oldValue != this)
  {
    // Wasn't already in 'set' state.
    // Treat old value as head of a linked-list of waiters
    // which we have now acquired and need to resume.
    auto* waiters = static_cast<awaiter*>(oldValue);
    while (waiters != nullptr)
    {
      // Read m_next before resuming the coroutine as resuming
      // the coroutine will likely destroy the awaiter object.
      auto* next = waiters->m_next;
      waiters->m_awaitingCoroutine.resume();
      waiters = next;
    }
  }
}
{% endcodeblock %}

最后，我们需要实现`operator co_await()`方法。这仅需要构造一个`awaiter`对象。

{% codeblock lang:c++ line_number:false %}
async_manual_reset_event::awaiter
async_manual_reset_event::operator co_await() const noexcept
{
  return awaiter{ *this };
}
{% endcodeblock %}

至此全部完成。我们拥有了一个可等待的异步手工重置事件，且是无锁的、没有内存分配、以及`noexcept`的实现。

如果想要运行该代码，或者查看在MSVC和Clang中的编译结果，可以参考{% link source on godbolt https://godbolt.org/g/Ad47tH %}。

你同样可以在{% link cppcoro https://github.com/lewissbaker/cppcoro %}库中找到这个类的实现，以及其他一些有用的可等待类型，例如`async_mutex`和`async_auto_reset_event`。

# 总结

本篇讲述了如何实现`operator co_await`，且定义了**可等待**和**等待者**属于的概念。

同时描述了如何实现一个可等待异步线程同步原语，即利用了等待者对象可以在协程栈上分配的特性来避免堆分配。

我希望本篇有助于你理解新的`co_await`操作符。

下一篇中，我会探讨**Promise**概念，以及协程类型作者如何来定制协程的行为。