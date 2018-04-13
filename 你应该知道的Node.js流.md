文章翻译自：[Node.js Streams: Everything you need to know](https://medium.freecodecamp.org/node-js-streams-everything-you-need-to-know-c9141306be93)

![streams.jpeg](https://upload-images.jianshu.io/upload_images/704770-8d3d386be3ecf252.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在开发者中普遍认为Node.js流不但难以应用，而且难以理解。现在有一个好消息，Node.js流将不在难以处理。过去几年，为了方便操作Node.js流，开发者开发了许多第三方Node.js包。但是在这篇文章中，我将集中在Node.js原生的流接口应用的介绍。


**“Streams are Node’s best and most misunderstood idea.”**

**— Dominic Tarr**

## 什么是流

流就是数据集合----诸如数组或是字符串。不同之处在于流不必一次全部使用，它们也不必适应内存。这两个特点使流在处理大量数据或一次向外部返回一大块数据时非常高效。

流代码的组合性，为流处理大量数据，提供了新的力量。就像把微小linux命令组合成功能丰富的组合命令一样，Node.js流通过同样的方式实现数据通道的功能。

![linux-command.png](https://upload-images.jianshu.io/upload_images/704770-c4b2f1a0ccce57dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


```
const grep = ... // A stream for the grep output
const wc = ... // A stream for the wc input
grep.pipe(wc)
```

许多Node.js内置模块都实现流接口：

![native-module.png](https://upload-images.jianshu.io/upload_images/704770-4bc04e09d1f1d2b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面展示的API中，一部分原生Node.js对象既是可读又是可写流，诸如TCP Sockets,Zlib和Crypto流。

值得注意的是，对象的部分行为是密切相关的。例如：在客户端HTTP对象是可读流，在服务端HTTP对象是可写流。这是因为在HTTP上，程序从一个对象上读取数据（http.IncomingMessage），然后将读取的数据写到另外一个对象上（http.ServerResponse）。

## 一个关于流实际用例

理论听起来美妙，但并不能完全传递流的精妙。让我们看一个例子，通过这个例子，可以看出是否使用流对于内存占用的不同影响。

*让我们先创建一个大的文件：*

```
const fs = require('fs');
const file = fs.createWriteStream('./big.file');

for(let i=0; i<= 1e6; i++) {
  file.write('Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.\n');
}

file.end();
```

在上面的示例代码中，fs模块可以通过流接口实现对文件的读和写。通过循环一百万次可写流，将数据写入到big.file文件中。

执行相应的代码，生成大约400兆的文件。

下面是一个专门用来操作这个大文件的Node服务器代码：

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

当服务接收请求，程序将会通过异步fs.readFile函数向请求者发送数据报文。表面上看这样的代码并不会阻塞程序的事件循环，真的是这样吗？

好，当我们启动这段服务，然后请求这个服务后，让我们看看内存占用将会有怎样的变化。

当启动服务时，服务占用的内存量是8.7兆。

![server-memory.png](https://upload-images.jianshu.io/upload_images/704770-75f79b2920a5a671.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


然后请求这个服务，注意内存占用的情况：

![server-memory.gif](https://upload-images.jianshu.io/upload_images/704770-a3f39f43cd535c64.gif?imageMogr2/auto-orient/strip)


哇 ---- 内存占用突然间跳到434.8兆。

本质上讲，程序在将大数据文件写入到http响应对象前，会将所有的数据写入内存中。这种代码的效率是非常低效的。

HTTP响应对象也是一个可写流，如果我们将代表big.file内容可读流与HTTP相应对象的可写流在管道中连接，程序就可以通过两个流管道，在不产生近400兆内存占用的情况下，达到相同的结果。

Node.js中的fs模块通过createReadStream方法，生成读取文件的可读流。然后程序可以通过管道将可读流传到http响应对象中：

```
const fs = require('fs');
const server = require('http').createServer();

server.on('request', (req, res) => {
  const src = fs.createReadStream('./big.file');
  src.pipe(res);
});

server.listen(8000);
```

当再次请求服务时，一个神奇的事情发生了（注意内存占用）：

![server-memory-opt.gif](https://upload-images.jianshu.io/upload_images/704770-e6ae3cd36931864e.gif?imageMogr2/auto-orient/strip)
 
 #### 发生了什么

当客户端请求大文件时，程序一次将一块数据生成流文件，这就意味着我们不需要将数据缓存到内存中。内存的占用也仅仅上升了25兆。

我们可以将这个测试用例推到极限。重新生成五百万行而不是一百万行的big.file文件，重新生成的文件将会达到2GB，这将大于Node.js默认的缓存量。

使用fs.readFile实现大内存文件的读取，最好不要修改程序默认的缓存空间。但是如果使用fs.createReadStream,即便请求2GB的数据流也不会有问题。使用第二种方式作为服务程序的内存占用几乎不发生变化。

## 流

流在Node.js中有四种：Readable、Writable、Duplex和Transform。

- 可读流（Readable Stream）： 可被消费的资源的抽象，如 fs.createReadStream方法
- 可写流（Writable Stream）：数据可被写入目的地的抽象，如 fs.createWriteStream方法
- 双工流（Duplex Stream）：既是可读流，又是可写流， 如 TCP socket
- 转换流（Transform Stream）：以双工流为基础，把读取数据或者写入数据进行修改或者转换。如 zlib.createGzip函数使用gzip方法实现数据的压缩。我们可以认为**转换流的输入是可写流、输出是可读流**。这就是听说过的"通过流"的转换流。

所有流都是EventEmitter模块的实例，触发可读和可写数据的事件。但是，程序可以使用pipe函数消费流数据。

#### 管道（pipe）函数

*下面是一段你值得记忆的魔法代码：*

`
readableSrc.pipe(writableDest)
`

在这一简单的代码中,**将可读流的输出 (数据源) 作为可写流的输入 (目标) 进行管道化**。源数据必须是可读流，目标必须是可写流。它们也可以同时是双工流或者转换流。事实上, 如果开发者将双工流传入管道中, 我们就可以像Linux那样链接到管道调用:
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

管道（pipe）方法是实现流消费的最简单方式。通常建议使用管道函数（pipe）或者事件消费流，但是避免将它们混合使用。通常当你使用管道（pipe）函数时，你就不需要使用事件。但是如果程序需要定制流的消费，事件可以是一个不错的选择。

## 流事件
除了读取可读流源，并把读取的数据写入到可写的目的地上。管道（pipe）还可以自动管理一些事情。例如：它可以处理异常，当一个流比其它流更快或更慢时结束文件。

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


这些事件或函数通常以某种方式相关联，关于可读流的事件有：
- data事件，当流传递给消费者时触发
- end事件，当流中没有数据被消费时触发

关于可写流的重要事件有：
-drain事件，可写流接受数据时的信号
-finish事件，所有的数据已经刷新到系统底层时触发

通过事件和函数可以组合在一起后自定义流或流的优化。为了消费可读流，程序可以使用pipe/unpipe方法，或是read/unshift/resume方法。为了消费可写流，程序可以将它作为pipe/unpipe的目的地，或是通过write方法写入数据，在写入完成后调用end方法。

#### 可读流中的暂停（Paused）和流动（Flowing）模式

可读流中存在两种模式影响程序对可读流的使用：
- 可读要么处在暂停（paused）模式
- 要么处在流动（flowing）模式

这些模式又是被认为是拉和推模式。

所有的可读流在默认情况下都是从暂停模式开始，在程序需要时，转换成流动模式或者暂停模式。有时这种转换是自发的。

当可读流在暂停（paused）模式时，我们可以使用read方法按需读取流数据。但是，对于处在流动（flowing）模式下的可读流，我们必须通过监听事件来消费数据。

在流动（flowing）模式下，如果数据没有被消费，数据可能会丢失。这就是当程序中有流动的可读流时，需要data事件处理数据的原因。事实上，只需要添加data事件就可以将流从暂停转换为流动模式和解除程序与事件监听器的绑定、将流从流动模式转换为暂停模式。其中的一些是为了向后兼容老版本Node流的接口。

开发者可以使用resume方法和pause方法，手动实现两种流模式的转换。

![mode-transform.png](https://upload-images.jianshu.io/upload_images/704770-b284f1099d410a3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当程序使用管道（pipe）方法消费可读流时，开发这就不必关心流模式的转换了，因为管道（pipe）会自动实现。

##实现流

当我们Node.js中的流时，有两种不同的任务：
- 继承流的任务
- 消费流的任务

到目前为止，我们仅仅讨论着消费流。让我们实现一些例子吧！

*实现流需要我们在程序中引入流模块*

 #### 实现可写流

开发者可以使用流模块中的Writeable构造器，实现可写流。

`
const { Writable } = require('stream');
`

开发者实现可写流有很多种方式。例如：通过继承writable构造器

```
class myWritableStream extends Writable {
}
```

但是，我更喜欢使用构造器的实现方式。仅仅通过writable接口创建对象并传递一些选项：一个必须函数选项是write函数，传入要写入的数据块。

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

write函数有三个参数：

- chunk通常是buffer数组，除非开发者对流做了自定义的配置
- encoding参数在测试用例中是需要的，但是开发者通常可以忽略
-  callback是程序处理数据块后，开发者调用的回调函数。通常是写入操作成功与否的信号。如果是写入异常的信号，调用出现异常的回调函数。

在outStream类中，程序仅仅将数据转换为字符串类型打印出来，并在没有出现异常时调用回调函数，以此来标志程序的成功执行。这是一个简单但不是特别有效的回声流，程序会输出任何输入的数据。

要使用这个流，我们可以将它与process.stdin一起使用，这是一个可读的流，将process.stdin传输到outStream。

当程序执行时，任何通过process.stdin输入的数据都会被outStream中的console.log函数打印出来。

但是这个功能可以通过Node.js内置模块实现，因此这并不是一个非常实用的流。它与process.stdout的功能非常类似，我们使用下面的代码可以实现相同的功能：

`
process.stdin.pipe(process.stdout);
`

#### 实现可读流

为了实现一个可读流，开发者需要引入Readable的接口后通过这个接口构建对象：

```
const { Readable } = require('stream');
const inStream = new Readable({});
```

这是实现可读流的最简单方式，开发者可以直接推送数据以供消费使用。

```
const { Readable } = require('stream'); 
const inStream = new Readable();
inStream.push('ABCDEFGHIJKLM');
inStream.push('NOPQRSTUVWXYZ');
inStream.push(null); // No more data
inStream.pipe(process.stdout);
```
当程序中推送一个空对象时，这就意味着不再有数据供给可读流。

开发者可以将可读流通过管道传给process.stdout的方式，供其消费可读流。

执行这段代码，程序可以读取来自可读流的数据，并将数据打印出来。非常简单，但是并不高效。

上段代码的本质是：把数据推送给流，然后将流通过管道传给process.stdout消费。其实程序可以在消费者请求流时，按需推送数据，这种方式比上一种更高效。通过实现readable流中的read函数实现：

```
const inStream = new Readable({
  read(size) {
    // there is a demand on the data... Someone wants to read it.
  }
});
```

在readable流中调用read函数，程序可以将部分数据传输到队列上。例如：每次向队列中推送一个字母，字母的的序号从65（代表A）开始，每次推送的字母序号都自增1：
```
const inStream = new Readable({
  read(size) {
    this.push(String.fromCharCode(this.currentCharCode++));
    if (this.currentCharCode > 90) {
      this.push(null);
    }
  }
});
inStream.currentCharCode = 65;
inStream.pipe(process.stdout);
```

当消费者正在消费可读流时，read函数就会被激活，程序就会推送更多的字母。通过向队列些推送空对象，终止循环。如上代码中，当字母的序号超过90时，终止循环。

这段代码的功能与之前实现的代码是等效的，但是当消费者要读流时，程序可以按需推送数据的效率更优。因此建议使用这种实现方式。

#### 实现双工、转换流

双工流：在同一对象上分别实现可读流和可写流，就像对象继承了两个可读流和可写流接口。

下面是一个双工流，它结合了上面已经实现的可读流、可写流的例子：

```
const { Duplex } = require('stream');

const inoutStream = new Duplex({
  write(chunk, encoding, callback) {
    console.log(chunk.toString());
    callback();
  },

  read(size) {
    this.push(String.fromCharCode(this.currentCharCode++));
    if (this.currentCharCode > 90) {
      this.push(null);
    }
  }
});

inoutStream.currentCharCode = 65;
process.stdin.pipe(inoutStream).pipe(process.stdout);
```

通过实现双工流的对象，程序可以读取A－Z的字母，然后按顺序打印出来。开发者将stdin可读流传输到双工流，然后将双工流传输到stdout可写流中，打印出A－Z字母。

**在双工流中的可读流与可写流是完全独立的，双工流仅仅是一个对象同时具有可读流和可写流的功能。理解这一点至关重要。**

转换流比双工流更有趣，因为它的结果是根据输入流计算出来的。

对于双工流，并不需要实现read和write函数，开发者仅仅需要实现transform函数，因为transform函数已经实现了read函数和write函数。

下面是将输入的字母转换为大写格式后，然后把转换后的数据传给可写流：

```
const { Transform } = require('stream');

const upperCaseTr = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
});

process.stdin.pipe(upperCaseTr).pipe(process.stdout);
```
在这个例子中，开发者仅仅通过transform函数，就实现了向上面双工流的功能。在transform函数中，程序将数据转换为大写后推送到可写流中。


## 流的对象模式

默认情况下，流只接受Buffer和String的数据。但是开发者可以通过设置objectMode标识的值，可以使流接受任何Javascript数据。

下面的例子可以证明这一点。通过一组流将以逗号分隔的字符串转换为Javscript对象，于是"a,b,c,d"转换成｛a: b, c : d｝。

```
const { Transform } = require('stream');
const commaSplitter = new Transform({
  readableObjectMode: true,
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().trim().split(','));
    callback();
  }
});

const arrayToObject = new Transform({
  readableObjectMode: true,
  writableObjectMode: true,
  transform(chunk, encoding, callback) {
    const obj = {};
    for(let i=0; i < chunk.length; i+=2) {
      obj[chunk[i]] = chunk[i+1];
    }
    this.push(obj);
    callback();
  }
});

const objectToString = new Transform({
  writableObjectMode: true,
  transform(chunk, encoding, callback) {
    this.push(JSON.stringify(chunk) + '\n');
    callback();
  }
});
process.stdin
  .pipe(commaSplitter)
  .pipe(arrayToObject)
  .pipe(objectToString)
  .pipe(process.stdout)
```

commaSplitter转换流将输入的字符串（例如：“a，b，c，d”）转换为数组（［“a”， “b”， “c”， “d”］）。设置writeObjectMode标识，因为在transform函数中推的数据是对象而不是字符串。

然后将commaSplitter输出的可读流传输到转换流arrayToObject中。由于接收的是对象，同样需要在arrayToObject中需要设置writableObjectMode标识。由于需要在程序中推送对象（将传入的数组转换为对象），这也是程序中设置readableObjectMode标识的原因。最后，转换流objectToString接收对象，但是输出字符串。这就是程序中只设置writableObjectModel标识的原因。输出的可读流时正常的字符串（序列化后的数组）。

![transform-result.png](https://upload-images.jianshu.io/upload_images/704770-bafa5fb8d6b0b25a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**Node的内置转换流**

Node有许多内置转换流，如：lib和crypto流。

下面的代码是使用zlib.createGzip()流与fs模块的可读和可写流相结合，实现压缩文件的代码：

```
const fs = require('fs');
const zlib = require('zlib');
const file = process.argv[2];

fs.createReadStream(file)
  .pipe(zlib.createGzip())
  .pipe(fs.createWriteStream(file + '.gz'));
```

程序将读取文件的可读流传输进Node内置的转换流zlib中，最后传输到创建压缩文件的可写流中。因此开发者只要将需要压缩的文件路径作为参数传进程序中，就可以实现任何文件的压缩。

开发者可以将管道函数与事件结合使用，这是选择管道函数另一个原因。例如：开发者让程序通过打印出标记符显示脚本正在执行，并在脚本执行完毕后打印出"Done"信息。pipe函数返回的是目标流，程序可以在获取目标流后注册事件链：

```
const fs = require('fs');
const zlib = require('zlib');
const file = process.argv[2];

fs.createReadStream(file)
  .pipe(zlib.createGzip())
  .on('data', () => process.stdout.write('.'))
  .pipe(fs.createWriteStream(file + '.zz'))
  .on('finish', () => console.log('Done'));
```

开发者通过pipe函数可以很容易操作流，甚至在需要时，通过事件对经过pipe函数处理后的目标流做一些定制交互。

管道函数的强大之处在于，使用易理解的方式将多个管道函数联合在一起。例如：不同于上个示例，开发者可以通过传入一个转换流，标识脚本正在执行。

```
const fs = require('fs');
const zlib = require('zlib');
const file = process.argv[2];

const { Transform } = require('stream');

const reportProgress = new Transform({
  transform(chunk, encoding, callback) {
    process.stdout.write('.');
    callback(null, chunk);
  }
});

fs.createReadStream(file)
  .pipe(zlib.createGzip())
  .pipe(reportProgress)
  .pipe(fs.createWriteStream(file + '.zz'))
  .on('finish', () => console.log('Done'));
```

reportProgress只是一个转换流，在这个流中标识脚本正在执行。值得注意的是，代码中使用callback函数推送transform中的数据。这与先前示例中this.push()的功能是等效的。

组合流的应用场景还有很多。例如：开发者要先加密文件，然后压缩文件或是先压缩后加密。如果要完成这个功能，程序只要将文件按照顺序传入流中，使用crypto模块实现：

```
const crypto = require('crypto');
// ...
fs.createReadStream(file)
  .pipe(zlib.createGzip())
  .pipe(crypto.createCipher('aes192', 'a_secret'))
  .pipe(reportProgress)
  .pipe(fs.createWriteStream(file + '.zz'))
  .on('finish', () => console.log('Done'));
```

上面的代码实现了压缩、加密文件，只有知道密码的用户才可以使用加密后的文件。因为开发者不能按照普通解压的工具对加密后压缩文件，进行解压。

对于任何通过上面代码压缩的文件，开发者只需要以相反的顺序使用crypto和zlib流，代码如下：

```
fs.createReadStream(file)
  .pipe(crypto.createDecipher('aes192', 'a_secret'))
  .pipe(zlib.createGunzip())
  .pipe(reportProgress)
  .pipe(fs.createWriteStream(file.slice(0, -3)))
  .on('finish', () => console.log('Done'));
```

假设传输进去的文件是压缩后的文件，上面的程序首先会生成可读流，然后传输到crypto的createDecipher()流中，接着将输出的流文件传输到zlib的createGunzip()流中，最后写入到文件中。

上面就是我对这个主题的总结，感谢您的阅读，期待下次与你相遇。
































