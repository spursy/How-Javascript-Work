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
