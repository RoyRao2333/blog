---
title: 【Swift多线程】GCD的概念和使用
date: 2022-11-30 00:54:52
excerpt: Swift中，在提到并发的时候，一般指的是异步 + 并行...
categories:
  - Swift
---

Swift中，在提到并发的时候，一般指的是**异步 + 并行**。（异步和并行的概念假设大家已有所了解，这里不作详细介绍。如果想进一步了解的，可以参阅喵大的博客[Swift并发初步](https://onevcat.com/2021/07/swift-concurrency/)）

我们一般有两种方式来使我们的任务实现并发：```Grand Central Dispatch```（通常称为**GCD**）和 ```Operation```。两种方式相辅相成。此外，```Operation```其实是在GCD的基础上打造的，这个我会在接下来的博客中介绍，本文暂不提及。下面我会简单介绍一下GCD的基本概念以及使用。

## GCD (Grand Central Dispatch)

GCD是Apple对C语言```libdispatch```库的实现。它的目的是将任务（方法或闭包）进行排队，这些任务可以并行运行（具体取决于资源的可用性），然后让它们在可用的处理器核心上执行。

虽然GCD在其实现中使用了线程，但作为开发人员，你完全不需要担心如何进行线程管理。GCD的任务非常轻量级，以至于Apple在2009年关于GCD的技术简报中指出，部署一个任务只需要15条指令，而创建传统线程可能需要数百条指令。

GCD将你的所有任务都放入它管理的先进先出(FIFO)队列中，你提交给队列的每个任务都会由系统管理的线程池执行，并在完成后销毁线程。

> Note: GCD不能保证你的任务将在哪个线程上执行。


### Dispatch Queue

在Swift中，```DispatchQueue```是GCD管理的一组FIFO队列，我们提交的任务在队列里被加载、调度，然后在要执行的线程池中进一步分发。这些队列可以是串行的，即执行的顺序被严格限制；也可以是并发的，即同时执行。因为并发可以带来更好的性能，因此Apple鼓励我们更频繁地使用并发队列。

#### .main和.global()

这里，```.main```队列是GCD提供的全局可用的串行队列，用于在应用程序的主线程上执行任务。

> Note: 由于主线程用于更新UI和用户交互，因此我们在使用main队列执行任务时要注意，最好不要阻塞主线程。

为了避免阻塞主线程，除了```.main```队列之外，GCD还为我们提供了```.global()```方法，调用时返回一支全局队列，也称为后台队列，用于执行不必要在主线程执行的任务，即任何不属于UI更新的任务。

我们这里举一个例子：假如你需要根据当前状态，通过方法```getTextString(status: OSStatus)```来获取需要显示的文字，并更新到UI上的```NSTextField```上。

```swift
DispatchQueue.global(qos: .userInteractive).async { [unowned self] in
    // 根据当前状态获取要显示的文字
    let text2Show = getTextString(status)
    
    DispatchQueue.main.async {
        // 记得在主线程更新UI
        textfield.stringValue = text2Show
    }
}
```

GCD允许我们以同步或异步的方式分派任务。使用```.sync```（即同步）时，只有在任务执行完成后，才会将控制权返回给调用方函数。而使用```.async```（即异步）时会开始执行任务并立即将控制权返回给调用方，而不会阻塞线程。通常，我们在进行API调用或执行计算密集型任务 (CPU-intensive task) 时使用异步。具体使用什么方式，根据需求选择。

### Quality of Service (QoS)

在以上示例代码中，出现了一个参数```qos```，即Quality of Service。顾名思义，就是定义一个队列的重要性、优先级。

定义QoS有助于我们对```DispatchQueue```的任务进行优先级分类。但这并不意味着我们应该把所有的任务都安排在高优先级上。通过合理的分配，可以使我们的app节能且响应迅速。

QoS的优先级被分为了四个大类：

- **userInteractive**: 用于显示动画，或更新UI。
- **userInitiated**: 用于从API中获取数据，且在过程中不允许用户进行交互。
- **utility**: 用于那些不需要用户关注或在意，可独立完成的任务。
- **background**: 用于任何优先级不高的任务，例如在本地数据库中保存数据。

一旦对任务队列进行分类后，系统就会在可分配的资源内以尽可能好的方式处理它。另一点需要注意的是，.utility和.background比其他分类的优先级更低，使用之前应当对任务类型进行合理的分析，否则你的app可能会出现线程爆炸和死锁。

### Dispatch Source

在Swift中，```DispatchSource```是一种用于处理事件的数据类型，这些被处理的事件为操作系统中的底层级别。GCD支持如下的```DispatchSource```类型：

- **Timer dispatch sources**: 定时器类型，能够产生周期性的通知事件；
- **Signal dispatch sources**: 信号类型，当UNIX信号到底时，能够通知应用程序；
- **Descriptor sources**：文件描述符类型，处理UNIX的文件或socket描述符，如：
	- 数据可读
	- 数据可写
	- 文件被删除、修改或移动
	- 文件的元信息被修改
- **Process dispatch sources**: 进程类型，能够通知一些与进程相关的事件类型，如：
	- 当进程退出
	- 当进程调用了fork或exec
	- 当一个信号传递给了进程
- **Mach port dispatch sources**: 端口匹配类型，能够通知一些端口事件的类型；
- **Custom dispatch sources**: 自定义类型，可以自定义一些事件类型。

在Swift中，```DispatchSource```能够替换一些异步的回调函数，特别是用于处理一些与系统相关的事件。当进行```DispatchSource```配置时，可以指定希望监控的事件类型，且可以指定DispatchQueue来处理上述的事件。当一个受监控的的事件出现时，所指定的block或函数将会被调用执行。

和将任务提交到```DispatchQueue```不同，```DispatchSource```将会持续对所提交的事件进行监控，除非手动取消指定事件的监控。

Signal、Process、File System、Timer等等，所有事情都可以通过```DispatchSource```对象进行处理。你可以使用```DispatchSource```监控信号、进程、文件、计时器等，详见[Apple官方文档](https://developer.apple.com/documentation/dispatch/dispatchsource)。

## GCD API

创建主队列

```swift
DispatchQueue.main.async {}
```

创建全局队列

```swift
DispatchQueue.global().sync {}

DispatchQueue.global(qos: .background).async {}
```

创建自定义队列

```swift
DispatchQueue(label: "com.Roy.workingQueue").async {}

DispatchQueue(
    label: "com.Roy.concurrentQueue",
    qos: .default,
    attributes: .concurrent,
    autoreleaseFrequency: .inherit,
    target: nil
).async {}
```


## References

- 喵大 - [Swift并发初步](https://onevcat.com/2021/07/swift-concurrency/)
- [Guide to Multithreading in iOS](https://blog.devgenius.io/multithreading-in-swift-how-gcd-works-why-do-we-need-operation-queues-high-performing-ios-ddf6ca09583e)

<br/>

> 不足之处，欢迎指正。
