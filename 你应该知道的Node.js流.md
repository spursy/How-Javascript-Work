文章翻译自：[Node.js Streams: Everything you need to know](https://medium.freecodecamp.org/node-js-streams-everything-you-need-to-know-c9141306be93)

![streams.jpeg](https://upload-images.jianshu.io/upload_images/704770-8d3d386be3ecf252.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

开发者中普遍流传着Node.js流难以应用，也难以理解。现在对开发者有一个好消息，Node.js流将不在难以处理。

过去几年，开发者为了更加简单操作Node.js流，开发了许多第三方Node.js包。但是在这篇文章中，我将集中在Node.js原生的流接口。


**“Streams are Node’s best and most misunderstood idea.”**

**— Dominic Tarr**

#### 什么是流

流就是数据的集合----诸如数组或是字符串。不同之处在于流不必一次全部使用，它们也不必适应内存。这两个特点是的流在处理大量数据或从外部一次返回一大块数据时非常高效。

但是，流不仅仅可以处理大量数据。流实现代码中的组合性，提供了新的力量。就像通过把微小linux命令组合成强大的linux命令一样，Node.js流可以通过同样的方式实现功能强大的数据通道。

![linux-command.png](https://upload-images.jianshu.io/upload_images/704770-c4b2f1a0ccce57dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
const grep = ... // A stream for the grep output
const wc = ... // A stream for the wc input
grep.pipe(wc)
```

许多Node.js内置模块都实现流接口：

![native-module.png](https://upload-images.jianshu.io/upload_images/704770-4bc04e09d1f1d2b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面展示的名单中，有些原生Node.js对象既是可读又是可写的流。有些对象既是可读又是可写流，诸如TCP sockets,zlib和crypto流。

值得注意的是对象是密切相关的。例如：在客户端上HTTP响应是可读流，在服务端是可写流。这是因为在HTTP上，程序从一个对象上读取数据（http.IncomingMessage），然后将读取的数据写到另外一个对象上（http.ServerResponse）。


还要注意，在子进程中，stdio流（stdin，stdout，stderr）具有逆流类型。这允许一个真正简单的方法，从主进程stdio流中传出和传入这些流。

#### 一个关于流实际用例

理论很伟大，但并不能百分之百传递。让我们看一个例子，可以看出是否使用流对于内存占用的不同影响。

*让我们先创建一个大的文件：*

```
const fs = require('fs');
const file = fs.createWriteStream('./big.file');

for(let i=0; i<= 1e6; i++) {
  file.write('Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.\n');
}

file.end();
```

看我过去创建大文件的代码，一个可写的流。

fs模块可以通过流接口实现对文件的读和写。在上面的示例代码中，我们通过循环一百万次可写流，将数据写入到big.file文件中。

执行相应的代码，生成一个大约400兆的文件。

下面是一个专门用来服务big.file的Node服务器。

```
const fs = require('fs');
const server = require('http').createServer();

server.on('request', (req, res) => {
  fs.readFile('./big.file', (err, data) => {
    if (err) throw err;
  
    res.end(data);
  });
});

server.listen(8000);
```

当服务接收一个请求，程序将会通过异步fs.readFile方法服务大数据文件。这样的代码并不会阻塞程序的事件循环，真的是这样吗？

好，当我们启动这段服务，然后请求这个服务后，内存占用将会有怎样的变化。

当启动服务时，服务占用的内存量是8.7兆。

![server-memory.png](https://upload-images.jianshu.io/upload_images/704770-75f79b2920a5a671.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后请求这个服务，注意内存占用的情况：

![server-memory.gif](https://upload-images.jianshu.io/upload_images/704770-a3f39f43cd535c64.gif?imageMogr2/auto-orient/strip)


哇 ---- 内存占用突然间跳到434.8兆。

本质上讲，程序在将big.file的内容写到http 响应对象前，将所有的内容写入内存中。这种程序的效率是非常低效的。

HTTP响应对象同样也是一个可写流。这就意味着如果我们有一个代表big.file内容可读流，程序可以通过两个流管道，在不产生近400兆内存占用的情况下，达到相同的结果。

Node.js中的fs模块通过createReadStream方法，生成对所有文件的可读流。程序可以通过管道将可读流传到http响应对象中：

```
const fs = require('fs');
const server = require('http').createServer();

server.on('request', (req, res) => {
  const src = fs.createReadStream('./big.file');
  src.pipe(res);
});

server.listen(8000);
```

现在当程序再次请求服务时，一个神奇的事情发生了（注意内存占用）：

![server-memory-opt.gif](https://upload-images.jianshu.io/upload_images/704770-e6ae3cd36931864e.gif?imageMogr2/auto-orient/strip)

 **发生了什么**

当客户端请求大文件时，程序一次将一块数据生成流文件，这就意味着我们不需要将一点数据内容缓存在内存中。内存的占用也仅仅上升了25兆。

我们可以将这个测试用例推到极限。重新生成五百万行而不是一百万行的big.file文件，重新生成的文件将会达到2GB，这将大于Node.js默认的缓存量。

最好不要通过修改程序默认的缓存空间的方式，使用fs.readFile实现大内存文件的读取。但是如果使用fs.createReadStream,即便请求2GB的数据流也不会有问题。最重要的是，使用第二种方式的内存占用几乎不发生变化。

#### 流101

流在Node.js中有四种：Readable、Writable、Duplex和Transform。

- 可读流（Readable Stream）： 可被消费的资源的抽象，如 fs.createReadStream方法
- 可写流（Writable Stream）：数据可被写入目的地的抽象，如 fs.createWriteStream方法
- 双工流（Duplex Stream）：既是可读流，又是可写流， 如 TCP socket
- 转换流（Transform Stream）：以双工流为基础，可以把读取的数据或者写入的数据进行修改或事转换。如 zlib.createGzip函数使用gzip实现数据的压缩。我们可以认为转换流的输入是可写流，输出是可读流。你也许听说过"通过流"的转换流。

所有流都是EventEmitter模块的实例。它们触发可读和可写数据的事件。但是，程序可以使用pipe方法消费流数据。

**管道（pipe）函数**

*下面是一段你值得记忆的魔法代码：*

`
readableSrc.pipe(writableDest)
`

在上面简短的代码中，我们将可读流的输出，作为数据源，放入管道中、将可写流的输入，作为管道的目的地。源数据必须是可读流，目的地必须是可写流。它们也可以同时是双工流或者转换流。事实上，如果我们将管道化为双工流，我们就可以像linux的方式使用管道链：
```
readableSrc
  .pipe(transformStream1)
  .pipe(transformStream2)
  .pipe(finalWrtitableDest)
 ```

管道函数返回的是目标流，它可以允许程序做上面的链式调用。下面的代码： 流a是可读流、流b与c是双工流、流c是可写流。
```
a.pipe(b).pipe(c).pipe(d)
# Which is equivalent to:
a.pipe(b)
b.pipe(c)
c.pipe(d)
# Which, in Linux, is equivalent to:
$ a | b | c | d
```

管道（pipe）方法是实现消费流中最简单的方式。通常建议使用管道函数（pipe）或者事件消费流，但是避免将它们混合使用。通常当你使用管道（pipe）函数时，你就不需要使用事件。但是如果程序需要定制流的消费，事件可以是一个不错的选择。

**流事件**
除了读取可读流源，并把读取的数据写入到可写的目的地上。管道（pipe）还可以自动管理一些事情。例如：它可以处理异常，结束文件和当一个流比其它流更快或更慢。

但是，流可以通过事件被直接消费。下面是一段等效于管道（pipe）方法的程序，它通过简化的、与事件等效的代码实现数据的读取或写入。
```
# readable.pipe(writable)
readable.on('data', (chunk) => {
  writable.write(chunk);
});
readable.on('end', () => {
  writable.end();
});
```

这里有一系列可用于可读、可写流的事件或函数。

![event-function.png](https://upload-images.jianshu.io/upload_images/704770-6f014fcceaf90969.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


这些事件或函数通常以某种方式相关联，因为它们经常在一起使用。

关于可读流的重要事件有：
- data事件，当流传递给消费者一块数据时触发
- end事件，当流中没有数据被消费时触发

关于可写流的重要事件有：
-drain事件，这是可写流接受数据时的信号
-finish事件，当所有的数据已经刷新到系统底层时触发

事件和函数可以组合在一起，用于自定义流或流的优化。为了消费可读流，程序可以使用pipe/unpipe方法，或是read/unshift/resume方法。为了消费可写流，程序可以将它作为pipe/unpipe的目的地，或是通过write方法写入数据，在写入完成后调用end方法。

**可读流中的暂停（Paused）和流动（Flowing）模式**

可读流中存在两种模式，影响程序对可读流的使用：
- 可读要么处在暂停（paused）模式
- 要么处在流动（flowing）模式

这些模式又是被认为是拉和推模式。

所有的可读流在默认情况下都是从暂停模式开始，但是程序需要时，很容易转换成流动模式或事暂停模式。有时这种转换将是自发的。

当可读流在暂停（paused）模式时，我们可以使用read方法按需读取流数据。但是，对于处在流动（flowing）模式下的可读流，数据一直都处在流动的模式，我们必须通过监听事件来消费数据。

在流动（flowing）模式下，如果没有被消费，数据可能会丢失。这就是在程序中有流动的可读流时，程序需要data事件处理数据的原因。事实上，只需要添加data事件就可以将流从暂停转换为流动模式和解除程序与事件监听器的绑定、将流从流动模式转换为暂停模式。其中的一些事为了向后兼容老版本Node流的接口。

开发者可以使用resume方法和pause方法，手动实现两种流模式的转换。

![mode-transform.png](https://upload-images.jianshu.io/upload_images/704770-b284f1099d410a3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当程序使用管道（pipe）方法消费可读流时，开发这就不必关心流模式的转换了，因为管道（pipe）会自动实现。

#### 实现流

当我们Node.js中的流时，有两种不同的任务：
- 继承流的任务
- 消费流的任务

到目前为止，我们仅仅讨论着消费流。让我们实现一些吧！

*实现流需要我们在程序中引入流模块*

 **实现可写流**

开发者需要使用流模块中的Writeable构造器，实现可写流。

`
const { Writable } = require('stream');
`

开发者实现可写流有很多种方式。例如：通过继承writable构造器

```
class myWritableStream extends Writable {
}
```

但是，我喜欢更简单构造器实现的方式。开发者仅仅通过writable接口创建对象并传递一些选项。一个必须函数选项是写入函数，暴露要写入的数据块。

```
const { Writable } = require('stream');
const outStream = new Writable({
  write(chunk, encoding, callback) {
    console.log(chunk.toString());
    callback();
  }
});

process.stdin.pipe(outStream);
```

写入函数有三个参数：

- chunk通常是buffer数组，除非开发者对流做了自定义的配置
- encoding参数在测试用例中是需要的。但是开发者通常可以忽略
-  callback是在程序处理数据块后，开发者调用的回调函数。这里通常是写入操作成功与否的信号。如果是写入异常的信号，调用出现异常的回调函数。















