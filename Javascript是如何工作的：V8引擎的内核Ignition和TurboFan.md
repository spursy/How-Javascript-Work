>文章翻译自V8团队系列博客中[Launching Ignition and TurboFan](https://v8project.blogspot.al/2017/05/launching-ignition-and-turbofan.html)

今天我怀着无比激动的心情宣布V8执行Javascript的管道5.9版将会与Chrome的M59同步。最新版的执行管道将会为Javascript带来显著的性能和内存占用的优化。关于更多性能优化数据我们会在文末有大量的论述，首先让我们认识下Ignition和TurboFan。

![Ignition.png](https://upload-images.jianshu.io/upload_images/704770-69a5273f9e9ad303.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![TurboFan.png](https://upload-images.jianshu.io/upload_images/704770-d0146def9e6ab951.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

新的管道是由解释器(Ignition)和编译器(TurboFan)组成。这些技术对于在这些年一直专注V8博客的开发者并不感到陌生，但是在V8的技术演变过程中却有着里程碑的意义。

Ignition和TurboFan第一次用来执行Javascript是从V8 5.9开始的。这也意味着自2010后一直服务V8引擎的Full-codegen和Crankshaft将不在使用。它们不能满足Javascript每年推陈出新变化趋势和优化这些新特性的需求。我们也将计划将它们完全从V8技术体系中清除，V8引擎将会朝着越来越简单、易控制方向发展。

### 漫长的旅途
V8引擎团队在优化真实生产环境下Javascript的性能和评估Full-codegen、Crankshaft的优劣做了大量细致的工作。融合Ignition和TurboFan的开发工作已经进行了三年半，这将是我们在未来继续优化执行Javascript性能的基础。

TurboFan在2013发起初只是为了弥补Crankshaft仅仅可以优化Javascript一部分语言的短板。例如，它并没有通过结构化的异常处理来设计代码，即代码块不能通过try、catch、finally等关键字划分。另外由于为每一个新的特性Crankshaft都将要做九套不同的框架代码适应不同的平台，因此在适配新的Javascript语言特性也很困难。还有Crankshaft框架代码的设计也限制优化机器码的扩展。尽管V8引擎团队为每一套芯片架构维护超过一万行代码，Crankshaft也不过为Javascript挤出一点点性能。


TurboFan在起初设计的时不仅是为了适配ES5标准的特性，同时也是为了优化ES2015及以后的规范的新语言特性。在TurboFan通过清楚区分高质量和低质量优化编译的分层编译器，实现在不修改架构代码的情况下优化新的语言特性。同时由于在TurboFan增加明确的指令选择编译阶段，可以为每个支持的平台编写少得多的体系结构代码。在这个新的阶段中，体系结构代码只需要编写一次，而且很少需要更改。这些变化为V8提供了更容易维护和可扩展的特性。

Ignition设计之初是为了减少移动设备的内存占用。以前通过Full-codegen基准编译器生成的代码几乎要占用Chrome浏览器三分之一的堆内存。这样为应用的实际数据留下的内存空间就很少。当在限制RAM的Android设备上启用Chrome M53的Ignition时，未优化的基准Javascript代码在ARM64的移动设备上的内存占用下降了九倍。

后来V8团队利用Ignition和TurboFan合作生成并优化的机器码而不是像Crankshaft重复编译源代码，来提高V8性能。Ignition的字节码为V8引擎提供了一种更简洁、更少错误的基准执行模型，大大简化V8的自适应优化的去优化机制这一关键特性。最终，由于产生字节码的速度要远远快于产生Full-codegen的基准编译代码，激活Ignition通常可以改善代码的启动速度和网页的加载速度。

从整体架构上Ignition与TurboFan在一起配合工作有许多的好处。例如，不需要在手工编写高效的程序集处理Ignition生成的字节码，而是使用TurboFan中间件来处理，并让TurboFan为V8所支持的平台进行优化和生成最终的执行代码。

### 运行表现
抛开历史，让我们看看真实生产环境下新管道技术的表现。

V8团队过去一直在使用[Telemetry-Catapult](https://github.com/catapult-project/catapult)作为测试性能的框架。在[上一篇文章](https://v8project.blogspot.al/2016/12/how-v8-measures-real-world-performance.html)中我们已经讨论关于通过真实生产环境的测试数据驱动性能优化的重要性和如何使用[WebPageReplay](https://github.com/chromium/web-page-replay)与Telemetry结合采集测试。

![Reduction in time spent in V8 for user interaction benchmarks.png](https://upload-images.jianshu.io/upload_images/704770-ce725c2f58b0695b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

虽然Speedometer是一个综合标准，我们之前也没发现其它综合测试标准可以比它更准确描述现代Javascript在真实环境中性能。在管道技术切换到Ignition和TurboFan后，Speedometer的综合指标在不同的平台略微有些差异，但是整体上提高了5-10%。

![benchmarkscores.Improvements on Web and Node.js benchmarks .png](https://upload-images.jianshu.io/upload_images/704770-71fdff4b109f12cf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

与此同时Ignition和TurboFan也减少了V8整体的内存占用。在M59版的Chrome浏览器中，新的管道技术可以为桌面端和移动端减少5-10%不等的内存占用。Ignition内存占用的减少最总导致V8整体内存占用的优化，这一方面我在[之前的文章](https://v8project.blogspot.al/2016/08/firing-up-ignition-interpreter.html)中有所涉猎。

这些性能上的改善都只是开始，Ignition和TurboFan组成的新的管道技术为执行Javascript的性能优化普平了道路，V8的内存无论在Chrome还是在Node.js都将会大幅降低。我们期待与你分享这些改进。
