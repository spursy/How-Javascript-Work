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







  






