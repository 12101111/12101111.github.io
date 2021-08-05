+++
title = "lld为什么在多核CPU上这么慢"
date = 2021-08-04
[taxonomies]
categories = ["Linux"]
tags = ["gentoo","Linux", "LLVM", "C++"]

+++

最近在一台48核96线程的服务器上编译chromium, 使用LLVM12全家桶(Clang, lld), 开启了lto, 发现在链接最终的二进制文件chromium时, lld虽然占用了所有的CPU资源, 但是大量CPU占用是发生在内核态的(使用htop可以看到CPU占用条一半以上是红色的), 因此对lld的性能进行了一些分析, 发现了lld为何在在内核态占用大量的CPU资源.

<!-- more -->

## perf top

`perf top` 命令可以显示一个进程各个函数占用CPU的百分比, `-p`或`--pid=`可以指定要分析的进程, 不加该参数则表示分析系统所有线程(包括内核线程).

不指定进程pid直接运行`perf top`,发现最占用CPU的是内核的`native_queued_spin_lock_slowpath`函数, 位于内核源码的`kernel/locking/qspinlock.c`文件中.

其余的函数有内核的`futex_wake`, `_raw_spin_lock`, `futex_wait_setup`, libc的`__pthread_mutex_lock`, `__pthread_mutex_timedlock`, `__pthread_mutex_unlock`, `malloc` `memcpy`, `memcmp`, `free` 等.

通过对内核的挖掘以及perf top提供的信息, 可以得到内核(Linux 5.13)部分的调用栈

```
native_queued_spin_lock_slowpath
_raw_spin_lock
futex_wake/futex_wait
do_futex
__x64_sys_futex
do_syscall_64
entry_SYSCALL_64_after_hwframe
```

libc(musl libc 1.2.2)的调用栈(注:部分是宏或内联函数)

```
__syscall(SYS_futex,...)
__futexwait/__wake
__lock/__unlock
malloc/free
```

```
__syscall(SYS_futex,...)
pthread_mutex_lock/pthread_mutex_timedlock/pthread_mutex_unlock
```

## 使用jemalloc替代musl的mallocng

musl 1.2.0引入的`mallocng`虽然性能较之前的老`malloc`实现性能高了不少, 但是查看其源码可知, 在多线程的情况下, 其`malloc/free`操作仍需要无条件上锁/解锁, 导致多线程场景下的内存分配性能降低. jemalloc是一个著名的高性能`malloc`库,使用jemalloc替代musl自带的实现可以解决lld因内存大量分配导致的性能下降.

使用该命令使得lld自动使用jemalloc:

```shell
sudo patchelf --add-needed libjemalloc.so.2 /usr/bin/lld
```

## perf record

替换`malloc`库后, 仍然有大量的内核自旋锁的调用, 因此需要进一步调查.

`perf top`的动态界面得到的数据并不精确和直观, 可以使用[`FlameGraph`](https://github.com/brendangregg/FlameGraph)绘制更加直观的图形

```shell
perf record -F 200 -p $PID --call-graph lbr
perf script -i perf.data > out.perf
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > out.svg
```

最终得到如下的图形

![flamegraph](/image/lld.svg)

<a href="/image/lld.svg" target="_blank">flamegraph生成的图片是可交互的，点击此在新标签页打开该svg图片</a>

最终我们可以确定llvm内的调用栈为

* [std::shared_timed_mutex::lock_shared](https://en.cppreference.com/w/cpp/thread/shared_timed_mutex/lock_shared): [头文件](https://github.com/llvm/llvm-project/blob/00809c8889ed34a5fe014167aa473216dcb63a47/libcxx/src/shared_mutex.cpp#L61) [实现](https://github.com/llvm/llvm-project/blob/00809c8889ed34a5fe014167aa473216dcb63a47/libcxx/src/shared_mutex.cpp#L61)
* [llvm::Pass::getPassName](https://llvm.org/doxygen/classllvm_1_1Pass.html#ad729b39eacf070a9bca84533b3c743bf): [头文件](https://github.com/llvm/llvm-project/blob/39fa96a4906934774ba20bcb0cd5f808f619f3a6/llvm/include/llvm/Pass.h#L107) [实现](https://github.com/llvm/llvm-project/blob/39fa96a4906934774ba20bcb0cd5f808f619f3a6/llvm/lib/IR/Pass.cpp#L76)
* [llvm::FPPassManager::runOnFunction](https://llvm.org/doxygen/classllvm_1_1FPPassManager.html#a0dec4e6b40dec12d8c6a17040ee73021): [实现](https://github.com/llvm/llvm-project/blob/46020f6f0c8aa134002208b2ecf0593b04c46d08/llvm/lib/IR/LegacyPassManager.cpp#L1402)

## 所以为什么lld这里这么慢

在`llvm::FPPassManager::runOnFunction`中, 只有这一行调用了`getPassName()`

```cpp
llvm::TimeTraceScope PassScope("RunPass", FP->getPassName());
```

根据[`llvm::TimeTraceScope`的文档](https://llvm.org/doxygen/structllvm_1_1TimeTraceScope.html), The [TimeTraceScope](https://llvm.org/doxygen/structllvm_1_1TimeTraceScope.html) is a helper class to call the begin and end functions of the time trace profiler. When the object is constructed, it begins the section; and when it is destroyed, it stops it. If the time profiler is not initialized, the overhead is a single branch.

虽然这个class的overhead很小, 但是为了计算它的构造函数的参数, overhead很大, 而且这个参数的内容在profiler没有启用时直接就扔掉了. 显然这里应该使用一个闭包或者别的进行lazy evaluation.

继续挖掘为什么`getPassName()`会用到锁.

`llvm::Pass`这个类的子类大多重写了`getPassName()`方法, 例如:

```cpp
  StringRef getPassName() const override { return "Loop Pass Manager"; }
```

但是没有重写的子类就会使用默认的实现:

```cpp
StringRef Pass::getPassName() const {
  AnalysisID AID =  getPassID();
  const PassInfo *PI = PassRegistry::getPassRegistry()->getPassInfo(AID);
  if (PI)
    return PI->getPassName();
  return "Unnamed pass: implement Pass::getPassName()";
}
```

注意到这里的`PassRegistry::getPassRegistry()`调用, 其[代码](https://github.com/llvm/llvm-project/blob/39fa96a4906934774ba20bcb0cd5f808f619f3a6/llvm/lib/IR/PassRegistry.cpp#L30)为:

```cpp
static ManagedStatic<PassRegistry> PassRegistryObj;
PassRegistry *PassRegistry::getPassRegistry() {
  return &*PassRegistryObj;
}
```

PassRegistry是一个加了锁的全局变量, 其[定义](https://github.com/llvm/llvm-project/blob/39fa96a4906934774ba20bcb0cd5f808f619f3a6/llvm/include/llvm/PassRegistry.h#L38)为:

```cpp
/// PassRegistry - This class manages the registration and intitialization of
/// the pass subsystem as application startup, and assists the PassManager
/// in resolving pass dependencies.
/// NOTE: PassRegistry is NOT thread-safe.  If you want to use LLVM on multiple
/// threads simultaneously, you will need to use a separate PassRegistry on
/// each thread.
class PassRegistry {
  mutable sys::SmartRWMutex<true> Lock;

  /// PassInfoMap - Keep track of the PassInfo object for each registered pass.
  using MapType = DenseMap<const void *, const PassInfo *>;
  MapType PassInfoMap;

  using StringMapType = StringMap<const PassInfo *>;
  StringMapType PassInfoStringMap;

  std::vector<std::unique_ptr<const PassInfo>> ToFree;
  std::vector<PassRegistrationListener *> Listeners;

public:
  PassRegistry() = default;
  ~PassRegistry();

  /// getPassRegistry - Access the global registry object, which is
  /// automatically initialized at application launch and destroyed by
  /// llvm_shutdown.
  static PassRegistry *getPassRegistry();

  /// getPassInfo - Look up a pass' corresponding PassInfo, indexed by the pass'
  /// type identifier (&MyPass::ID).
  const PassInfo *getPassInfo(const void *TI) const;

  /// getPassInfo - Look up a pass' corresponding PassInfo, indexed by the pass'
  /// argument string.
  const PassInfo *getPassInfo(StringRef Arg) const;
```

PassRegistry使用的Map并非线程安全的, 因此使用sys::SmartRWMutex进行保护. 而后面的`getPassInfo()`则需要加锁解锁的操作:

```cpp
const PassInfo *PassRegistry::getPassInfo(const void *TI) const {
  sys::SmartScopedReader<true> Guard(Lock);
  return PassInfoMap.lookup(TI);
}
```

而`SmartRWMutex`最终则是对`std::shared_timed_mutex`以及其他的一些锁的封装.

## ## Fix

查看`llvm::TimeTraceScope`的代码, 其构造函数有:

```cpp
/// The TimeTraceScope is a helper class to call the begin and end functions
/// of the time trace profiler.  When the object is constructed, it begins
/// the section; and when it is destroyed, it stops it. If the time profiler
/// is not initialized, the overhead is a single branch.
struct TimeTraceScope {

  TimeTraceScope() = delete;
  TimeTraceScope(const TimeTraceScope &) = delete;
  TimeTraceScope &operator=(const TimeTraceScope &) = delete;
  TimeTraceScope(TimeTraceScope &&) = delete;
  TimeTraceScope &operator=(TimeTraceScope &&) = delete;

  TimeTraceScope(StringRef Name) {
    if (getTimeTraceProfilerInstance() != nullptr)
      timeTraceProfilerBegin(Name, StringRef(""));
  }
  TimeTraceScope(StringRef Name, StringRef Detail) {
    if (getTimeTraceProfilerInstance() != nullptr)
      timeTraceProfilerBegin(Name, Detail);
  }
  TimeTraceScope(StringRef Name, llvm::function_ref<std::string()> Detail) {
    if (getTimeTraceProfilerInstance() != nullptr)
      timeTraceProfilerBegin(Name, Detail);
  }
  ~TimeTraceScope() {
    if (getTimeTraceProfilerInstance() != nullptr)
      timeTraceProfilerEnd();
  }
};
```

显然我们可以在第二个参数Detail的地方传入闭包而不是`StringRef`

进一步查看`timeTraceProfilerBegin`函数的源码

```cpp
struct llvm::TimeTraceProfiler {
  void begin(std::string Name, llvm::function_ref<std::string()> Detail) {
    Stack.emplace_back(steady_clock::now(), TimePointType(), std::move(Name),
                       Detail());
  }
}

void llvm::timeTraceProfilerBegin(StringRef Name, StringRef Detail) {
  if (TimeTraceProfilerInstance != nullptr)
    TimeTraceProfilerInstance->begin(std::string(Name),
                                     [&]() { return std::string(Detail); });
}

void llvm::timeTraceProfilerBegin(StringRef Name,
                                  llvm::function_ref<std::string()> Detail) {
  if (TimeTraceProfilerInstance != nullptr)
    TimeTraceProfilerInstance->begin(std::string(Name), Detail);
}

void llvm::timeTraceProfilerEnd() {
  if (TimeTraceProfilerInstance != nullptr)
    TimeTraceProfilerInstance->end();
}
```

显然传入`StringRef`也会立刻复制一遍然后放进闭包里, 所以修复方案就是直接把所有`TimeTraceScope`的构造换成传入闭包的形式, 这样不使用llvm自身的profiler时就不需要计算这个string了.

