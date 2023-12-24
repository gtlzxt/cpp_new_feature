## C++新标准分享_管窥协程
---
### Part 1. 标准发展简述

* **C++98/03:**
    引入了STL标准库
* **C++11:**
    语言级别的线程支持
    lambda表达式
    智能指针
    右值引用
* **C++14:**
    shared_lock 共享锁
    \[\[deprecated]]等Attribute 
* **C++17:**
    结构化绑定
    optional
    variant
* **C++20:**
    modules 模块
    coroutines 协程

[编译器对C++标准支持情况概览](https://en.cppreference.com/w/cpp/compiler_support)

简单总结：
  1. C++支持的特性越来越多，但是只要没有使用这些特性，是不需要额外的负担的，保证了在效率方面的优势。
  2. C++在发展过程中也会逐渐借鉴其他语言中成熟的特性，以保证C++的竞争力。
  3. 特性在增加，以前一些需要Hack方式的丑陋解决方案，会越来越多的被语言层面支持
   
---
### Part 2. 协程简述

**协程简述：**

协程是一种特殊的函数，它的执行可以被暂停或恢复。要定义协程，关键字co_return ，co_await，或co_yield 必须出现在函数体中。c++ 20的协程是无栈的;除非编译器进行了优化，否则它们的状态是在堆上分配的。

**协程违反直觉之处**
函数返回到调用者的方法：
1. 函数调用return
2. 使用goto
3. 使用exception

协程是第4种方法。
一个典型的协程函数调用顺序：
![普通函数调用和协程函数调用区别](/coroutine_call.webp)

**协程的使用场景：**
更高效的任务切换。

进程，线程和协程常见的切换时间对比

  空| 进程| 线程|协程
 -|-|-|-
 切换者|操作系统|操作系统|程序本身
 切换时机|线程调度|进程调度|程序本身决定
 切换内容|页目录，内核栈，上下文信息|内核栈，上下文信息|上下文信息
 切换过程|用户态->内核态->用户态|用户态->内核态->用户态|始终在用户态
 切换耗时|典型10微秒级别|典型微秒级别|典型10纳秒级别

 **使用基础函数创建和使用协程**

 ```
 #include <coroutine>
#include <iostream>
#include <thread>

namespace Coroutine {
  struct task {
    struct promise_type {
      promise_type() {
        std::cout << "1.create promie object\n";
      }
      task get_return_object() {
        std::cout << "2.create coroutine return object, and the coroutine is created now\n";
        return {std::coroutine_handle<task::promise_type>::from_promise(*this)};
      }
      std::suspend_never initial_suspend() {
        std::cout << "3.do you want to susupend the current coroutine?\n";
        std::cout << "4.don't suspend because return std::suspend_never, so continue to execute coroutine body\n";
        return {};
      }
      std::suspend_never final_suspend() noexcept {
        std::cout << "13.coroutine body finished, do you want to susupend the current coroutine?\n";
        std::cout << "14.don't suspend because return std::suspend_never, and the continue will be automatically destroyed, bye\n";
        return {};
      }
      void return_void() {
        std::cout << "12.coroutine don't return value, so return_void is called\n";
      }
      void unhandled_exception() {}
    };

    std::coroutine_handle<task::promise_type> handle_;
  };

  struct awaiter {
    bool await_ready() {
      std::cout << "6.do you want to suspend current coroutine?\n";
      std::cout << "7.yes, suspend becase awaiter.await_ready() return false\n";
      return false;
    }
    void await_suspend(
      std::coroutine_handle<task::promise_type> handle) {
      std::cout << "8.execute awaiter.await_suspend()\n";
      std::thread([handle]() mutable { handle(); }).detach();
      std::cout << "9.a new thread lauched, and will return back to caller\n";
    }
    void await_resume() {}
  };

  task test() {
    std::cout << "5.begin to execute coroutine body, the thread id=" << std::this_thread::get_id() << "\n";//#1
    co_await awaiter{};
    std::cout << "11.coroutine resumed, continue execcute coroutine body now, the thread id=" << std::this_thread::get_id() << "\n";//#3
  }
}// namespace Coroutine

int main() {
  Coroutine::test();
  std::cout << "10.come back to caller becuase of co_await awaiter\n";
  std::this_thread::sleep_for(std::chrono::seconds(1));

  return 0;
}
 ```
输出结果：
```
1.create promie object
2.create coroutine return object, and the coroutine is created now
3.do you want to susupend the current coroutine?
4.don't suspend because return std::suspend_never, so continue to execute coroutine body
5.begin to execute coroutine body, the thread id=0x10e1c1dc0
6.do you want to suspend current coroutine?
7.yes, suspend becase awaiter.await_ready() return false
8.execute awaiter.await_suspend()
9.a new thread lauched, and will return back to caller
10.come back to caller becuase of co_await awaiter
11.coroutine resumed, continue execcute coroutine body now, the thread id=0x700001dc7000
12.coroutine don't return value, so return_void is called
13.coroutine body finished, do you want to susupend the current coroutine?
14.don't suspend because return std::suspend_never, and the continue will be automatically destroyed, bye
```
---
### Part 3. 风控项目的应用展望

* 一个小问题
  当前所有界面的请求均为客户端发起一个请求，服务器端将整个请求处理完毕之后，将结果分为一个或者多个结果包，发送给客户端。当客户端请求的是比较重的业务逻辑时，服务器端需要进行复杂的业务处理，最终将一个巨大的结果返回给客户端，当预结算的需求出现任何的改动，就会影响到整个预结算服务。
* 思考，是否能将功能细分，降低功能之间的耦合度。
  将不同粒度的业务划分为不同层级的服务，数据和业务分割，基础服务和业务逻辑分割。不同层级的需求变更修改对应层级的代码，减少耦合。

![一个可能的架构图](/new_arch_for_risk.png)

按照上述架构图，预结算部分伪代码

```

void send_calc_margin_req() {
  send_req;
}

void on_calc_margin_rsp() {
  update_result_with_margin_rsp;
}

void send_calc_fee_req() {
  send_req;
}
void on_calc_fee_rsp() {
  update_result_with_fee_rsp;
}

```

使用协程的方式的伪代码：
```
  void calc_margin()
  {
    send_calc_margin_req;
    co_wait margin_rsp;
    update_result_with_margin_rsp;
  }

  void calc_fee()
  {
    send_calc_fee_req;
    co_wait fee_rsp;
    update_result_with_fee_rsp;
  }
```
**协程提供了一种使用同步的代码风格，编写异步代码的方式**


  
---
### Part 4. 封装好的协程库

[libco](https://github.com/Tencent/libco)
libco是微信后台大规模使用的c/c++协程库，2013年至今稳定运行在微信后台的数万台机器上。

[雅兰亭库 ](https://github.com/alibaba/yalantinglibs)
 是一个现代C++基础工具库的集合, 现在它包括 struct_pack, struct_json, struct_xml, struct_yaml, struct_pb, easylog, coro_rpc, coro_io, coro_http 和 async_simple.

 [libaco](https://github.com/hnes/libaco)
 一个极速的轻量级 C 非对称协程库, 核心实现不超过 700 行代码，但包含了一个协程库应该有的全部功能。

[libgo](https://gitcode.com/yyzybb537/libgo/tree/master/libgo)
libgo 是一个使用 C++ 编写的协作式调度的stackful有栈协程库, 同时也是一个强大的并行编程库。