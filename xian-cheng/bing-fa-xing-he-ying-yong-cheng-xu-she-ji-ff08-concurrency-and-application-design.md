# 并发性和应用程序设计（Concurrency and Application Design）

在计算的早期，单位时间计算机可以执行的最大工作量取决于CPU的时钟速度。但是随着技术进步以及处理器的设计变得更紧凑，热量和其他的物理限制开始限制处理器的最大时钟速度。所以，芯片制造商寻找其他方式来提升他们的芯片的总的性能。他们找到的解决方案是增加每个芯片上的处理器核心个数。通过提升核心个数，一个单独的芯片每秒可以执行更多的指令，而不用提升CPU速度或改变芯片的尺寸或热特性。唯一的问题是如何利用额外的核心。

为了利用多核，计算机需要软件可以同时地做多件事。对于像OS X或iOS这样的现代多任务处理操作系统，在任何给定的时间都可以有上百或者更多的程序运行，所以在不同的核心间调度每个程序是可能的。然而，这些程序中的大部分都是系统守护进程或后台应用程序，这些应用程序消耗很少的真实处理时间。相反，真正需要的是个别应用更有效的利用额外的核心的方式。

应用程序使用多个内核的传统方式是创建多个线程。然而，随着核心数量的增加，线程解决方案会产生一些问题。最大的问题是线程代码不能很好地扩展到任意数量的内核。你不能创建核心一样多的线程，并期望程序运行良好。你需要知道的是可以有效利用的核心数量，这对于应用程序自己计算是一个挑战。尽管你设法正确获得了数量，编程这么多线程，使它们高效运行以及避免互相干扰，依然是一个挑战。

所以，总结问题，应用程序需要一种方式以利用可变数量的计算机核心。单个应用程序执行的工作数量也需要能够动态地扩展以适应不断变化的系统条件。而且解决方案必须足够简单以免增加利用这些核心所需的工作数量。好消息是苹果的操作系统提供了所有这些问题的解决方案，这个章节将介绍构成这些解决方案的技术以及为了利用它们你可以对你的代码进行的设计调整。

## 从线程迁移（The Move Away from Threads）

尽管线程已经存在很多年且一直有它们的使用，但是它们没有解决以一种可扩展的方式执行多任务的一般性问题。使用线程，创建可扩展的解决方案的负担正好落在了你（开发者）的肩上。你必须决定创建多少线程并且需要根据系统条件的改变动态地调整数量。另外一个问题是你的应用程序承担与创建和维护它使用的任何线程相关的大部分成本。

代替对线程的依赖，OS X和iOS采用一个异步设计途径来解决并行问题。异步函数出现在操作系统里已经很多年了并且经常被用作启动可能消耗长时间的任务，比如从磁盘读取数据。被调用时，异步函数在后台做一些工作来开始一个任务运行但是在任务可能真正完成前返回。通常，这个工作包括请求一个后台线程，在该线程上开始期望的任务，然后当任务完成时发送一个通知（一般通过一个回调函数）给调用者。过去，如果你想做事的异步函数不存在，你必须写你自己的异步函数并创建自己的线程。但是现在，OS X和iOS提供技术允许你异步执行任何任务，而不必自己管理线程。

异步启动任务的一种技术是GCD\(Grand Central Dispatch\)。这个技术负责你通常在你的应用程序中会写的线程管理代码并且移动这些代码到系统级别。所有你必须做的是定义你想执行的任务然后添加它们到合适的调度队列。GCD负责创建需要的线程并且调度你的任务在这些线程上运行。由于线程管理现在是系统的一部分，GCD提供对于任务管理和执行的一种全面的方法，提供比传统线程更好的效率。

操作队列（Operation queue）是非常像调度队列的Objective-C对象。你定义期望执行的任务然后添加它们到一个操作队列，该操作队列处理这些任务的调度和执行。像GCD一样，操作队列为你处理所有的线程管理，确保任务在系统上执行的尽可能快和高效。

以下部分提供关于调度队列、操作队列以及一些其他你可以在你的应用程序中使用相关的异步技术。

### 调度队列（Dispatch Queues）

调度队列是执行自定义任务的基于C的机制。调度队列串行或并行执行任务，但总是先进先出的顺序。（换个说法，调度队列总是以任务加入队列的顺序出列并开始任务。）串行调度队列一次只运行一个任务，知道等到任务完成才出列并开始一个新任务。相反，并行调度队列开始尽可能多的任务，而不等待已经开始的任务结束。

调度队列有其他好处：

* 它们提供直接且简单的编程接口。
* 它们提供自动且全面的线程池管理。
* 它们提供调整部件的速度。（They provide the speed of tuned assembly.）
* 它们内存更高效（因为线程栈不占用应用程序内存）。
* 它们不会再负载下陷入内核。
* 异步调度任务到调度队列不能使队列死锁。
* 它们在竞争中优雅地缩放。（They scale gracefully under contention.）
* 串行调度队列为锁和其他同步原语（primitives）提供更高效的替代方案。

你提交到调度队列的任务必须封装在函数或块对象中。块对象是OS X10.6和iOS4.0发布的C语言特性，概念上跟函数指针接近，但是有一些额外的益处。通常在其他函数或方法中定义块以至于它们可以从该函数或方法中获取其他变量，而不是在块自己的词法范围内定义它们。块也可以被移出它们的原始范围并且拷贝到堆中，这就是当你提交它们到调度队列时发生的。所有这些语义使用相对少的代码实现非常动态的任务变得可能。

调度队列是GCD技术和C运行时的一部分。更多关于在应用程序中使用调度队列的信息，参[Dispatch Queues](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW1)。更多关于块和它们的益处的信息，参阅[_Blocks Programming Topics_](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Blocks/Articles/00_Introduction.html#//apple_ref/doc/uid/TP40007502)。

### 调度源（Dispatch Sources）

调度源是异步处理特定类型的系统事件的一个基于C的机制。调度源封装关于特定类型系统事件的信息并提交一个特定的块对象或函数到事件出现的调度队列。你可以使用调度源观察下列类型的系统事件：

* 定时器
* 信号处理器
* 描述相关事件
* 处理相关事件
* Mach port events
* 你触发的自定义事件

调度源是GCD技术的一部分。关于在应用程序中使用调度源接收事件的更多信心，参阅[Dispatch Sources](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/GCDWorkQueues/GCDWorkQueues.html#//apple_ref/doc/uid/TP40008091-CH103-SW1)。

### 操作队列（Operation Queues）

操作队列是并发调度队列的Cocoa等价物，通过NSOperationQueue类实现。调度队列总是以先进先出的顺序执行任务，而操作队列在决定任务的执行顺序时把其他的因素考虑在内。这些因素中最主要的是任务是否依赖于其他任务的完成。定义你的任务时设置依赖，你可以用它们为你的任务创建复杂的执行顺序图。

提交给操作队列的任务必须是NSOperation类的实例。操作对象是一个Objective-C对象，它封装了你想执行的工作和执行它所需的任何数据。由于NSOperation类本质上是抽象基类，你通常定义自定义子类来执行你的任务。然而，Foundation框架确实包含一些可以创建并使用的具体子类来执行任务。

操作对象生成键值观察（key-value observing）通知，这可以是检测任务进度的非常有用的方法。尽管操作队列总是并发地执行操作，当需要的时候你可以使用依赖来确保它们串行地执行。

更多关于如何使用操作队列以及如何定义自定义的操作对象的信息，参阅[Operation Queues](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html#//apple_ref/doc/uid/TP40008091-CH101-SW1)。

## 异步设计技术（Asynchronous Design Techniques）

在你曾经考虑重新设计你的代码以支持并发之前，你应该问下自己什么时候有必要这样做。并发可以通过确保主线程可以自由地响应用户事件来提升代码的响应速度。它甚至可以通过借由更多的核心在同样的时间内做更多的工作来提升代码的效率。然而，它也增加了开支和代码的总体复杂度，使代码难以编写和调试。

由于它增加了复杂度，并发不是你在产品周期的末期可以移植到应用程序的特性。做对这个需要小心的考虑应用程序执行的任务以及用来执行这些任务的数据结构。做的不对，你可能发现你的代码运行的更慢而且更不能响应用户。因此，在设计的开始花一些时间设定一些目标并且需要采用的方法是值得的。

每个应用程序有不同的需求和它执行的不同任务集合。一个文档不可能确切地告诉你该如何设计你的应用以及它相关的任务。然而，下列部分试图提供一些指导帮助你在设计过程中做出好的选择。

### 定义应用程序的预期行为（Define Your Application’s Expected Behavior）

甚至在你考虑添加并发性到你的应用程序之前，你应该总是以定义你认为正确的应用程序行为作为开始。理解应用程序的期望行为给你一种后期证实设计的方法。它还应该让你了解一下通过引入并发可能会带来的预期性能优势。

你需要做的第一件事是遍历应用程序执行的任务以及每个任务相关的对象和数据结构。起初，你可能想从用户选择一个菜单项或者点击按钮时执行的任务开始。这些任务提供分离的行为并且有明确界定的开始和结束点。你也应该遍历应用程序可能执行的没有用户交互的其他类型的任务，比如基于计时器的任务。

在你有了高级任务列表之后，开始把每个任务拆分成一系列成功完整任务所必须采取的步骤。这个层面，你应该主要考虑你需要对任何数据结构和对象的修改以及这些修改将如何影响应用程序的整体状态。你也需要注意对象和数据结构间的任何依赖。例如，一个任务涉及对一个对象数组做同样的改变，值得注意的是对一个对象的更改是否影响任何其它对象。如果这些对象可以彼此独立地进行修改，这可能就是你可以把这些修改并行的地方。

### 分解可执行的工作单元（Factor Out Executable Units of Work）

根据你对应用程序任务的理解，你应该已经能够辨别哪些代码将从并发中受益。如果改变任务中一个或多个步骤的顺序会改变结果，你很可能需要继续串行执行这些步骤。如果尽管改变了顺序也不会影响输出，你可以考虑并行执行这些任务。在这两种情况下，你定义将要执行的步骤等价的可执行工作单元。这些工作单元然后变成使用块或操作对象的封装，并调度到合适的队列。

对于你确定的每一个可执行工作单元，至少在开始时不用太担心正在执行的工作量。尽管转换线程总是一个开销，但调度队列和操作队列的一大好处是在很多情况下这些开销比传统线程要小很多。因此，使用队列执行更小的工作单元比使用线程更高效是可能的。当然，你应该总是测量你的实际性能，并按需调整任务的大小，但是开始时，任务不需要考虑的太小。

### 确定你需要的队列（Identify the Queues You Need）

现在你的任务被拆分成不同的工作单元并使用块对象或操作对象封装，你需要定义将要用来执行这些代码的队列。对于给定的任务，检查你创建的块或操作对象以及正确执行任务它们必须被执行的顺序。

如果你使用块实现任务，你可以添加你的块到一个串行或并行调度队列。如果需要特定的顺序，你应该总是添加你的块到一个串行调度队列。如果没有特定顺序的需要，你可以根据你的需要添加这些块到一个串行调度队列或添加它们到几个不同的调度队列。

如果你使用操作对象实现任务，队列的选择往往不如对象的配置有趣。串行执行操作对象，你必须在相关的对象之间配置依赖。依赖会阻止操作执行直到它依赖的对象完成它们的工作。

### 提高效率的指示（Tips for Improving Efficiency）

除了简单把你的代码分解成更小的任务并添加它们到队列，还有其他方法可以使用队列提升代码的整体性能：

* **如果内存使用是一个因素，考虑直接在任务内计算值。**如果你的应用程序已经绑定了内存，现在直接计算值可能比从主内存加载缓存值更快。计算值直接使用给定处理器内核的寄存器和高速缓存，这比主内存快得多。当然，如果测试表明这是一场性能胜利，那么你只应该这样做。
* **尽早确认连续任务并且尽可能使它们更并发。**如果一个任务必须串行执行，因为它依赖一些共享资源，考虑改变你的架构移除该共享资源。你可能考虑为每个需要的客户拷贝一份资源或者完全排除该资源。
* **避免使用锁。**调度队列和操作队列提供的支持使锁在大部分情况下不再需要。设计串行队列（或使用操作队列依赖）来以正确的顺序执行任务，替代使用锁来保护一些共享资源。
* **尽可能依靠系统框架。**实现并发性的最好方法是利用系统框架提供的内置并发。许多框架内部使用线程和其他技术实现并发行为。当定义你的任务时，查看是否现有框架定义函数或方法做了你正要做的事并且并发地做。使用这些API可以节省你的努力且更能给你最大的并行可能。

## 性能影响（Performance Implications）

操作队列、调度队列和调度源被提供以使并发地执行更多代码变得简单。然而，这些技术不保证对应用程序的效率或灵敏度的提升。你仍然有责任以既满足你需求的方式使用队列，也不会给应用程序的其他资源强加过多的负担。例如，机关你可以创建10000个操作对象并提交它们到一个操作队列，但这样做将会导致你的应用程序分配一个潜在的不重要的内存量，这可能会导致分页并降低性能。

不管是使用队列还是线程，在引入任何数量的并发到你的代码之前，你应该总是收集一组反映你的应用程序当前性能的基准指标。引入你的改变之后，你应该随后收集额外的指标并比较它们和你的基准看是否你的应用程序的总体性能获得了提升。如果并发的引入是应用程序性能更差或更不灵敏，你应该使用可用的性能工具检查潜在的原因。

对于性能和可用的性能工具的介绍，以及更先进的性能相关的话题的链接，参阅[_Performance Overview_](https://developer.apple.com/library/content/documentation/Performance/Conceptual/PerformanceOverview/Introduction/Introduction.html#//apple_ref/doc/uid/TP40001410)。

## 并发和其他技术（Concurrency and Other Technologies）

分解你的代码到模块化的任务是尝试以及提升应用程序并发量的最好方法。然而，这种设计方法可能不能满足每一个场景下的每一个应用的需求。可能有可以提供额外应用程序整体性能提升的其他选择，这取决于你的任务。本节概述了作为设计一部分考虑使用的其他一些技术。

### OpenCL和并发（OpenCL and Concurrency）

在OS X中，Open Computing Language（OpenCL）是一个在计算机的图形处理器上执行通用计算的基于标准的技术。如果您想要应用于大型数据集的计划明确，则OpenCL是一种很好的技术。例如，你可以使用OpenCLass对图像的像素执行滤波计算，或使用它对多个值一次执行复杂的数学计算。换句话说，OpenGL更适合于可以并行操作数据的问题集。

尽管OpenCL适合执行大规模数据并行操作，但不适合更通用的计算。 准备数据和将所需的工作内核传输到图形卡以便可以通过GPU对其进行操作需要花费大量精力。 同样，检索OpenCL生成的任何结果也需要花费大量精力。 因此，与系统交互的任何任务通常都不推荐用于OpenCL。 例如，您不会使用OpenCL处理来自文件或网络流的数据。 相反，使用OpenCL执行的工作必须更加独立，才能将其转移到图形处理器并独立计算。

更多关于OpenCL以及如何使用它的信息，参阅[_OpenCL Programming Guide for Mac_](https://developer.apple.com/library/content/documentation/Performance/Conceptual/OpenCL_MacProgGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008312)。

### 什么时候使用线程（When to Use Threads）

尽管操作队列和调度队列是并发执行任务优先选择的方法，但它们不是万能药。根据您的应用程序，您可能仍然有时需要创建自定义线程。如果你创建自定义线程，你应该努力创建尽量少的线程并且你应该仅为不能用其他方式实现的特定任务使用这些线程。

线程仍然是实现必须实时运行的代码的好方法。调度队列尽可能快地执行它们的任务，但它们不解决实时限制。如果您需要在后台运行的代码具有更多可预测的行为，那么线程仍然可以提供更好的选择。

与任何线程编程一样，你应该总是审慎地使用线程并且仅在绝对需要的时候。有关线程包的更多信息以及如何使用它们，参阅[_Threading Programming Guide_](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/Introduction/Introduction.html#//apple_ref/doc/uid/10000057i)。

