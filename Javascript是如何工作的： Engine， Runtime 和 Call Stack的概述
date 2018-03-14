>文章翻译自How Javascript work系列的第一篇[How JavaScript works: an overview of the engine, the runtime, and the call stack](https://blog.sessionstack.com/how-does-javascript-actually-work-part-1-b0bacc073cf)

Javascript语言越来越流行，无论个人还是团队可以把它应用在前端、后端、混合app、嵌入式等越来越多的技术方向上。

本篇将是我关于**Javascript是如何工作的**系列中的第一篇文章，深度理解Javascript中building blocks*（保证文章的严谨，Javascript中的专有名词我会保留翻译）*和如何组织building blocks工作的原理将会有利于程序员写出更加健壮的代码。我也会在这个文章系列中分享一些在我们构建SessionStack时，为保证轻量级的Javascript应用更健壮、更高的性能表现所使用到的一些不被大众熟知的特性。

通过[GitHut Stats](http://githut.info)的排名，我们可以看出Javascript在Github仓库活跃度和提交代码总量两个维度上都是排在第一名，在其它衡量标准的维度上Javascript的表现也是可圈可点。
![GitHut.png](https://upload-images.jianshu.io/upload_images/704770-31910f7a48ff14e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果项目对Javascript的依赖强度很大，为了构建一个稳定的程序每位开发者都必须了解Javascript的特性和Javascript的内部原理。

但现实状况却非如此，尽管大批的开发者中每天都在使用Javascript作为日常基础开发语言，对Javascript是如何工作的原理却知之甚少。

###概述
几乎每位开发者都听说过V8引擎的概念，许多开发者知道Javascript是单线程或Javascript使用callback队列。

在这篇文章中，我将详细阐述这些概念以及Javascript是如何工作的。这些细节将有助你写出更健壮、非阻塞的应用程序。

如果你是一位Javascript新手，这一系列文章将会让你知道为什么Javascript相较其它开发语言是如此惊艳。

如果你是一位资深Javascript开发者，我多希望这一系列的文章可以让你对每天使用的Javascript Runtime有一些新的洞见。

###Javascript引擎
 Google’s V8是流行的Javascript引擎之一，它使用在Chrome浏览器和Node.js中。下面是V8引擎一个简化的视图：

![V8引擎.png](https://upload-images.jianshu.io/upload_images/704770-05a9dc6ffe51f669.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

V8引擎主要包含两个部分：
* Memory Heap — 分配内存将会在这里发生
* Call Stack — 回调函数将会在这里执行

###Runtime
有一些APIs被开发者在浏览器中经常使用到（如：“setTimeout”），然而这些APIs也许并不是由Javascript引擎提供的。

*它们来自于哪里呢？*
*它们的来历有些复杂。*
![Runtime.png](https://upload-images.jianshu.io/upload_images/704770-274c37d32c2791cd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

诸如DOM、AJAX、setTimeout等其它是由浏览器提供的，我们称之为WEB APIs。

接下来，我们将谈谈非常流行的**callback queue**和**event loop**。

###Call Stack
Javascript是一种单线程的编程语言，这导致它只有单一的Call Stack。因此在某一时刻，他只能做一件事。

Call Stack是一种数据结构，他主要是记录Javascript整个执行过程。当Javascript的虚拟机执行一个函数，就会把这个函数推送到Call Stack中。当这个函数返回值或是执行完毕后，这个函数就会从Call Stack删除。

*让我们一起看一个例子，注意下面的代码：*
```
function multiply(x, y) {
    return x * y;
}
function printSquare(x) {
    var s = multiply(x, x);
    console.log(s);
}
printSquare(5);
```
当Javascript引擎在执行这段代码的前一刻，Call Stack是空的。然后Call Stack将会按照下图发生变化。
![CallStack.png](https://upload-images.jianshu.io/upload_images/704770-c9e7af4528082633.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**看下面的代码**，这段代码模拟在Call Stack中出现异常后的全过程。

```
function foo() {
    throw new Error('SessionStack will help you resolve crashes :)');
}
function bar() {
    foo();
}
function start() {
    bar();
}
start();
```
假设这段代码在foo.js中，foo.js在chrome浏览器执行后将会出现下面的堆栈追踪记录。
![StackTrace.png](https://upload-images.jianshu.io/upload_images/704770-dca648e475214650.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**堆栈溢出：**Javascript引擎产生的堆栈超过Javascript运行环境所提供的最大数量。这种异常在代码中存在递归但没有设置递归结束的条件时，尤其容易产生。
*下面就是这种类型的代码：*
```
function foo() {
    foo();
}
foo();
```
Javascript引擎执行这段代码是从foo函数开始，在这个函数中不断调用自己并没有设置终止条件，从而产生无限循环。每一次执行foo，Call Stack都会添加一次函数。*这就像下面显示的那样：*
![Overflowing.png](https://upload-images.jianshu.io/upload_images/704770-a3c71cbd76c4ed3b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当Javascript引擎中的Call Stack的长度，超过Javascript执行环境中Call Stack的实际长度时，Javascript执行环境（Chrome浏览器或Node）就会抛出下面的异常。
![Exception.png](https://upload-images.jianshu.io/upload_images/704770-74c6f4e82f16aadb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


在多线程环境中，要考虑诸如死锁等复杂执行过程。单线程的环境中相比较要简单很多，但是单线程同样有它的限制。Javascript单线程的执行环境中，如何应对复杂的调用，单线程会不会限制程序的性能。

###并发（concurrence）与时间循环（Event Loop）
当在你的Call Stack中存在一个需要占用相当大执行时间的函数时，将会发生什么。例如在浏览器中通过Javascript传输一个比较大的image文件时，你会怎么做？

你也许会问这怎么也算是一个问题。当Call Stack有待执行的函数时，浏览器会阻塞在这里，并不做其它的任务。这也意味着你不可能在app中呈现流畅复杂的UI。

问题不仅仅如此，一旦Call Stack中等待执行的任务很多时，浏览器要在很长的时间内都不能回应其它事件。许多浏览器这时都会抛出一个提示信息，征求你是否要关闭页面。
![PopError.jpeg](https://upload-images.jianshu.io/upload_images/704770-4bee70decaa730be.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这样必然将导致非常差的用户体验。

我们如何在复杂的环境下既不阻塞UI同时也不使浏览器长时间没有回应？好吧，这里我就不卖关子了，可以使用异步回调。
这将会是我在**Javascript是如何工作的**系列文章的第二篇《V8引擎的内核和5个写出高质量代码的技巧》






