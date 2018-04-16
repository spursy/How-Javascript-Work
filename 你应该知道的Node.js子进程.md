文章翻译自[Node.js Child Processes: Everything you need to know](https://medium.freecodecamp.org/node-js-child-processes-everything-you-need-to-know-e69498fe970a)

**如何使用spawn函数、exec函数、execFile函数和for函数**

![child-process.png](https://upload-images.jianshu.io/upload_images/704770-f0db94353574b23e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Node.js中的单线程和非阻塞的特性对单进程非常有用。但是事实上，面对程序中日益增加的工作量，单个cpu中的单进程显然是不足的。

无论服务器如何强大，单线程只可以利用有限的资源。

事实上，Node.js运行在单线程上，并不意味着开发者不能利用多进程，当然还有多台服务器。

使用多进程是扩展Node.js程序最佳的方式。Node.js就是为在多个节点，创建分布式应用而设计的。这也是取名Node的原因。可伸缩性已经渗透到平台中，因此开发不能等到应用程序运行到生命周期后期，在开始思考这个问题。

请注意，在阅读本篇文章前你应该理解Node.js事件和Node.js流的相关知识。如果你还没准备好，我推荐你阅读下面两篇文章：

[Node.js事件驱动](https://www.jianshu.com/p/7622d4305c4d)
[你应该知道的Node.js流](https://www.jianshu.com/p/50bf611b53ca)

## 子进程模块

开发者通过Node的child_process模块，可以很容易spin子进程。这些子系统可以通过消息系统相互间通信。

开发者可以通过child_process模块的内部命令，来访问操作系统。

开发者可以控制子进程的输入流，监听其输出流。开发者同样可以控制输入底层操作系统命令的参数、并且对命令的输出做任何改动。由于命令的输入与输出数据都可以被Node.js流处理，因此开发者可以将一个命令的输出作为（就像linux命令那样），作为另一个命令源数据。

*注意本文中所有的例子都是基于linux系统，如果你使用的系统时windows系统，你需要将对应的linux命令换成windows命令。*

在Node.js中有四种函数创建子进程：spawn()、fork()、exec()和execFile()。

接下来，我们将会讨论这四种函数间的不同以及不同函数的应用场景。

## Spawn(衍生)子进程

Spawan(衍生)函数可以在新的进程中启动命令，并通过Spwan(衍生)函数向启动的命令传递参数。例如，通过衍生的子进程，执行"pwd"命令：

```
const { spawn } = require('child_process');
const child = spawn('pwd');
```

Node.js程序从child_process模块结构出spawn函数，并把它作为OS命令的第一参数去执行。

执行spawn函数的结果仅仅是继承事件接口的子进程的实例对象，开发者可以对它直接注册事件处理函数。例如开发者可以注册子进程执行结果和子进程退出的事件:
```
child.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});
child.stderr.on('data', (data) => {
  console.log(`stderr: ${data}`);
});

child.on('exit', function (code, signal) {
    console.log('child process exited with ' +
                `code ${code} and signal ${signal}`);
  });
```
开发者还可以注册子进程的处理事件有：disconnect、error和message。

- disconnect事件：当父进程调用child.disconnect函数时触发
- error事件：当进程不能衍生或者进程被杀死时触发
- close事件：当子进程的stdio流关闭时触发
- message事件：当子进程使用process.send()函数时触发，这个函数主要用于父子进程间的通信。

每个子进程都具有标准的stdio流，开发者可以通过child.stdin、child.stdout和child.stderr操作stdio流。

在子进程中的stdio流关闭时，子进程会触发close事件。close事件并不完全等同于exist事件，主要在于子进程可以共享相同的stdio流，当一个子进程并不会导致流关闭。

由于流是事件的触发者，开发可以监听子进程stdio流中的事件。
与普通进程不同，在子进程中，stdout/stderr是可读流、stdin是可写流。从根本上讲，这些流在子进程与主进程的特性是相反的。最为重要的，通过监听data事件，程序可以获得命令的输出或执行命令时产生的异常信息。

```
child.stdout.on('data', (data) => {
  console.log(`child stdout:\n${data}`);
});

child.stderr.on('data', (data) => {
  console.error(`child stderr:\n${data}`);
});
```
上面的监听函数将会打印出主进程和stdout和stderr的输出。当程序执行上面spawn函数，"pwd"命令的输出将会打印出来。子进程将会退出，并返回0，这说明没有异常发生。

开发者可以向spawn函数衍生出的子进程传递参数，这个参数的格式要求是数组。例如下面的find命令：

```
const child = spawn('find', ['.', '-type', 'f']);
```

如果命令执行的过程中出现异常，child.stderr的data事件被触发该事件获得程序退出code是1（意味着程序出现异常），异常的信息通常是根据异常的类型和OS系统有所不同。

由于子进程的stdin是可写流，开发者可以通过它向子进程写入数据。就像其它的可写流一样，pipe方法是使用可写流最简单的方式，程序可以将可读流写入到可写流中。由于主进程的stdin是可读流，程序就可以将它写入到子进程的可写流中。例如：
```
const { spawn } = require('child_process');

const child = spawn('wc');

process.stdin.pipe(child.stdin)

child.stdout.on('data', (data) => {
  console.log(`child stdout:\n${data}`);
});
```
在上面的例子中，子进程启动wc命令来计算输入数据的行数、字符数。让后将主进程的stdin(可读流)传输给子进程的stdin(可写流)中。执行上面的程序后，命令行工具将会开启输入模式。当输入组合键Ctrl＋D后，终止输入。已经输入的数据将会作为wc命令的输入数据源。

##Noted

开发者将进程的输出作为另一个进程的输入数据源，实现像linux命令那样的管道命令。例如开发者将find命令的stdout流，做为wc命令的输入数据源，实现计量文件夹中的文件数量：
```
const { spawn } = require('child_process');

const find = spawn('find', ['.', '-type', 'f']);
const wc = spawn('wc', ['-l']);

find.stdout.pipe(wc.stdin);

wc.stdout.on('data', (data) => {
  console.log(`Number of files ${data}`);
});
```
在wc命令后添加参数-l，实现计算文件的行数。上面的程序将会对当前项下所有目录中所有文件进行计数。

#### Shell语法和exec函数

默认情况下，spawn函数并不会创建新的shell，来执行通过参数传递进来的命令。由于不会创建新的shell，这是spawn函数比exec函数高效的主要原因。exec函数与spawn函数还有一点主要的区别，spawn函数通过流操作命令执行的结果，而exec函数则将程序执行的结果缓存起来，最后将缓存的结果传给回调函数中。

下面通过exec函数实现find|wx命令的例子:
```
const { exec } = require('child_process');

exec('find . -type f | wc -l', (err, stdout, stderr) => {
  if (err) {
    console.error(`exec error: ${err}`);
    return;
  }

  console.log(`Number of files ${stdout}`);
});
```
因为exec函数使用shell执行命令，因此开发者可以直接通过shell句法使用shell管道的特性。

值得注意，如果程序正在执行外部提供的任何动态输入将会存在安全隐患。用户只要输入特定命令就可以实现命令的注入攻击，如：*rm -rf ~~*。

exec函数缓存命令的输出，并将输出的结果作为回调函数的参数，传递给回调函数。

如果你需要使用shell句法，如果你期望命令操作的文件比较小，使用shell句法是一项不错的选择。注意，exec函数先将所要返回的数据缓存在内存中，然后返回。

如果执行命令后得到的数据太大，spawn函数将是很不错的选择，因为使用spawn函数会标准的IO对象转换为流。








  





