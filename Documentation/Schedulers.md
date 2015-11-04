Schedulers
调度器
==========

1. [Serial vs Concurrent Schedulers](#serial-vs-concurrent-schedulers)
1. [Custom schedulers](#custom-schedulers)
1. [Builtin schedulers](#builtin-schedulers)

Schedulers abstract away mechanism for performing work.
调度器是抽象于 mechanism ，用于进行工作的一个概念。
Different mechanisms for performing work include, current thread, dispatch queues, operation queues, new threads, thread pools, run loops ...
用于执行工作的不同 mechanism 包括有：当前线程，派遣队列，操作队列，新线程，线程池，运行循环等等。
There are two main operators that work with schedulers. `observeOn` and `subscribeOn`.
和调度器一起使用的操作符主要有两个： `observeOn` 和 `subscribeOn`.
If you want to perform work on different scheduler just use `observeOn(scheduler)` operator.
如果需要在不同的调度器中执行工作，使用 `observeOn(scheduler)` 操作符。
You would usually use `observeOn` a lot more often then `subscribeOn`.
通常情况下更多的是使用 `observeOn`，较少使用 `subscribeOn`.
In case `observeOn` isn't explicitly specified, work will be performed on which ever thread/scheduler elements are generated.
如果 `observeOn` 没有特别明确的指定调度器，那么就是默认在创建调度器的线程中执行。
Example of using `observeOn` operator

```
sequence1
  .observeOn(backgroundScheduler)
  .map { n in
      println("This is performed on background scheduler")
  }
  .observeOn(MainScheduler.sharedInstance)
  .map { n in
      println("This is performed on main scheduler")
  }
```

If you want to start sequence generation (`subscribe` method) and call dispose on a specific scheduler, use `subscribeOn(scheduler)`.
如果需要通过 `subscribe` 方法来开始一个序列并且需要在一个指定的调度器中调用 dispose 对象，使用方法 `subscribeOn(scheduler)`.
In case `subscribeOn` isn't explicitly specified, `subscribe` method will be called on the same thread/scheduler that `subscribeNext` or `subscribe` is called.
如果没有特别明确的指定调度器，`subscribe`方法会在调用 `subscribeNext` or `subscribe` 的线程中执行。
In case `subscribeOn` isn't explicitly specified, `dispose` method will be called on the same thread/scheduler that initiated disposing.
如果没有特别明确的指定调度器，`dispose` 方法会在初始化 disposing 相同的线程中进行处理。
In short, if no explicit scheduler is chosen, those methods will be called on current thread/scheduler.
也就是说，如果没有明确指定调度器，这些方法都会在当前的线程以及使用默认的调度器。
# Serial vs Concurrent Schedulers
连续调度器和并发调度器
Since schedulers can really be anything, and all operators that transform sequences need to preserve additional [implicit guarantees](GettingStarted.md#implicit-observable-guarantees), it is important what kind of schedulers are you creating.

In case scheduler is concurrent, Rx's `observeOn` and `subscribeOn` operators will make sure everything works perfect.

If you use some scheduler that for which Rx can prove that it's serial, it will able to perform additional optimizations.

So far it only performing those optimizations for dispatch queue schedulers.

In case of serial dispatch queue schedulers `observeOn` is optimized to just a simple `dispatch_async` call.

# Custom schedulers

Besides current schedulers, you can write your own schedulers.

If you just want to describe who needs to perform work immediately, you can create your own scheduler by implementing `ImmediateScheduler` protocol.

```swift
public protocol ImmediateScheduler {
    func schedule<StateType>(state: StateType, action: (/*ImmediateScheduler,*/ StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

If you want to create new scheduler that supports time based operations, then you'll need to implement.

```swift
public protocol Scheduler: ImmediateScheduler {
    typealias TimeInterval
    typealias Time

    var now : Time {
        get
    }

    func scheduleRelative<StateType>(state: StateType, dueTime: TimeInterval, action: (StateType) -> RxResult<Disposable>) -> RxResult<Disposable>
}
```

In case scheduler only has periodic scheduling capabilities, you can inform Rx by implementing `PeriodicScheduler` protocol

```swift
public protocol PeriodicScheduler : Scheduler {
    func schedulePeriodic<StateType>(state: StateType, startAfter: TimeInterval, period: TimeInterval, action: (StateType) -> StateType) -> RxResult<Disposable>
}
```

In case scheduler doesn't support `PeriodicScheduling` capabilities, Rx will emulate periodic scheduling transparently.

# Builtin schedulers

Rx can use all types of schedulers, but it can also perform some additional optimizations if it has proof that scheduler is serial.

These are currently supported schedulers

## MainScheduler (Serial scheduler)

Abstracts work that needs to be performed on `MainThread`. In case `schedule` methods are called from main thread, it will perform action immediately without scheduling.

This scheduler is usually used to perform UI work.

## SerialDispatchQueueScheduler (Serial scheduler)

Abstracts the work that needs to be performed on a specific `dispatch_queue_t`. It will make sure that even if concurrent dispatch queue is passed, it's transformed into a serial one.

Serial schedulers enable certain optimizations for `observeOn`.

Main scheduler is an instance of `SerialDispatchQueueScheduler`.

## ConcurrentDispatchQueueScheduler (Concurrent scheduler)

Abstracts the work that needs to be peformed on a specific `dispatch_queue_t`. You can also pass a serial dispatch queue, it shouldn't cause any problems.

This scheduler is suitable when some work needs to be performed in background.

## OperationQueueScheduler (Concurrent scheduler)

Abstracts the work that needs to be peformed on a specific `NSOperationQueue`.

This scheduler is suitable for cases when there is some bigger chunk of work that needs to be performed in background and you want to fine tune concurrent processing using `maxConcurrentOperationCount`.
