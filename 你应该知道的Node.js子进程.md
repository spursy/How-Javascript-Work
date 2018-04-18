文章翻译自[Node.js Child Processes: Everything you need to know](https://medium.freecodecamp.org/node-js-child-processes-everything-you-need-to-know-e69498fe970a)

**如何使用spawn函数、exec函数、execFile函数和for函数**

![child-process.png](https://upload-images.jianshu.io/upload_images/704770-f0db94353574b23e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Node.js中的非阻塞单线程的特性对单进程任务是非常有用。但是事实上，面对日益复杂的业务逻辑，单个cpu中的单进程所能提供的计算力显然是不足的。因为无论服务器如何强大，单线程只可以利用有限的资源。

事实上，Node.js运行在单线程上，并不意味着开发者不能利用多进程，当然还有多台服务器。

使用多进程是扩展Node.js程序最佳的方式。Node.js就是为在多个节点，创建分布式应用而设计的。这也是取名Node的原因。可伸缩性已经渗透到平台中，因此开发不能等到应用程序运行到生命周期后期，在开始思考这个问题。

请注意，在阅读本篇文章前你应该理解Node.js事件和Node.js流的相关知识。如果你还没准备好，我推荐你阅读下面两篇文章：

[Node.js事件驱动](https://www.jianshu.com/p/7622d4305c4d)
[你应该知道的Node.js流](https://www.jianshu.com/p/50bf611b53ca)

## 子进程模块

开发者通过Node的child_process模块，可以很容易衍生出子进程。这些子系统可以通过消息系统实现相互通信。

开发者可以通过child_process模块的内部命令，来访问操作系统。

开发者可以控制子进程的输入流，监听其输出流。开发者同样可以控制输入底层操作系统命令的参数、并且对命令的输出做任何所需要的改动。由于命令的输入与输出数据都可以被Node.js流处理，因此开发者可以将一个命令的输出（就像linux命令那样）作为另一个命令源数据。

*注意本文中所有的例子都是基于linux系统，如果你使用的系统时windows系统，你需要将对应的linux命令换成windows命令。*

在Node.js中有四种函数创建子进程：spawn()、fork()、exec()和execFile()。

接下来，我们将会讨论这四种函数间的不同函数的应用场景。

## Spawn(衍生)子进程

Spawan函数可以衍生出新的子进程，并通过Spwan函数向子进程传递命令。例如，通过衍生的子进程，执行"pwd"命令：

```
const { spawn } = require('child_process');
const child = spawn('pwd');
```

Node.js程序从child_process模块析构出spawn函数，向函数传递OS命令，并在子进程中执行OS命令。

执行spawn函数的结果是继承事件接口的子进程实例对象，开发者可以对它直接注册事件处理函数。例如开发者对子进程执行结果和子进程退出行为注册事件:
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
开发者对子进程还可以注册的处理事件有：disconnect、error和message。

- disconnect事件：当父进程调用child.disconnect函数时触发
- error事件：当进程不能衍生或者进程被杀死时触发
- close事件：当子进程的stdio流关闭时触发
- message事件：当子进程使用process.send()函数时触发，这个函数主要用于父子进程间的通信。

每个子进程都具有标准的stdio流，开发者可以通过child.stdin、child.stdout和child.stderr操作stdio流。

在子进程中的stdio流关闭时，子进程会触发close事件。close事件并不完全等同于exist事件，主要在于子进程可以共享相同的stdio流，当一个子进程并不会导致流关闭。

由于流是事件的触发者，开发者可以监听子进程stdio流中的事件。
与普通进程不同，在子进程中，stdout/stderr是可读流、stdin是可写流。从根本上讲，这些流在子进程与主进程的属性是相反的。最为重要的，通过监听data事件，程序可以获得命令的输出或执行命令时产生的异常信息。

```
child.stdout.on('data', (data) => {
  console.log(`child stdout:\n${data}`);
});

child.stderr.on('data', (data) => {
  console.error(`child stderr:\n${data}`);
});
```
当程序执行上面spawn函数，"pwd"命令的输出将会打印出来。子进程将会退出，并返回0，这说明没有异常发生。

除了可以向spawn函数衍生出的子进程传递命令，开发者还可以向它传递命令的参数，这个参数的格式要求是数组。例如下面的find命令：

```
const child = spawn('find', ['.', '-type', 'f']);
```

如果命令执行的过程中出现异常，child.stderr的data事件被触发该事件获得程序退出code是1（意味着程序出现异常），异常的信息通常是根据异常的类型和OS系统有所不同。

由于子进程的stdin是可写流，开发者可以通过它向子进程写入数据。就像其它的可写流一样，pipe方法是使用可写流最简单的方式，程序可以将可读流写入到可写流中。由于主进程的stdin是可读流，因此可以实现主进程向子进程穿数据。例如：
```
const { spawn } = require('child_process');

const child = spawn('wc');

process.stdin.pipe(child.stdin)

child.stdout.on('data', (data) => {
  console.log(`child stdout:\n${data}`);
});
```
在上面的例子中，子进程启动wc命令来计算输入数据的行数、字符数。然后将主进程的stdin(可读流)传输给子进程的stdin(可写流)中。执行上面的程序后，命令行工具将会开启输入模式。当输入组合键Ctrl＋D后，终止输入。已经输入的数据将会作为wc命令的输入数据源。

![stream-pipe.gif](https://upload-images.jianshu.io/upload_images/704770-4f84d6df79f4956b.gif?imageMogr2/auto-orient/strip)


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

默认情况下，spawn函数并不会衍生新的shell，执行通过参数传递进来的命令。由于不会创建新的shell，这是spawn函数比exec函数高效的主要原因。exec函数与spawn函数还有一点主要的区别，spawn函数通过流操作命令执行的结果，而exec函数则将程序执行的结果缓存起来，最后将缓存的结果传给回调函数中。

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

值得注意，要确保向exec函数传递的OS命令是没有安全隐患的。因为用户只要输入一些特定的命令就可以实现命令的注入攻击，如：*rm -rf ~~*。

exec函数缓存命令的输出，并将输出的结果作为回调函数的参数，传递给回调函数。

如果你需要使用shell句法，并且期望命令操作的文件比较小，使用shell句法是一项不错的选择。**注意，exec函数先将所要返回的数据缓存在内存中，然后返回。**

如果执行命令后得到的数据太大，spawn函数将是很不错的选择，因为使用spawn函数会标准的IO对象转换为流。

程序可以通过spawn函数衍生出继承父进程标准I／O对象的子进程，如果需要，可以在子进程中使用shell句法。下面的代码就是实现定制子进程的代码：

```
const child = spawn('find . -type f | wc -l', {
  stdio: 'inherit',
  shell: true
});
```

设置**stdion: 'inherit'**，当执行代码时，子进程将会继承主进程的stdin、stdout和stderr。主进程的process.stdout 流将会触发子进程的事件处理函数，并在事件处理函数中立刻输出结果。

设置**shell: true**，就像exec函数一样，程序可以向衍生函数传递shell句法，作为衍生函数的参数。即便这样，依旧可以利用衍生函数中流的特性。**不得不说这样是非常酷**

除了在spawn衍生函数的option对象中设置shell和stdio，开发者还有设置其它的选项。通过cwd属性设置程序工作的目录。例如下面将程序的工作目录设置为下载文件夹，实现计算对目的文件夹中所有文件计数的代码：

```
const child = spawn('find . -type f | wc -l', {
  stdio: 'inherit',
  shell: true,
  cwd: '/Users/samer/Downloads'
});
```

使用option对象env属性，可以设置对子进程可见的环境变量。process.env是env属性的默认值,提供对当前进程环境的任何命令访问权限。开发者可以设置env属性为空对象或子进程可见的环境变量值，实现定制子进程可见环境变量。

```
const child = spawn('echo $ANSWER', {
  stdio: 'inherit',
  shell: true,
  env: { ANSWER: 42 },
});
```

上面的echo命令并不能访问父进程的环境变量。由于设置env属性值，进程没有访问$HONE的权限但是可以访问$ANSWER。

通过设置spawn函数中option对象的detached属性，可以实现子进程完全独立于父进程的调用。

假设我们有一个让事件循环繁忙的timer.js测试程序：
```
setTimeout(() => {  
  // keep the event loop busy
}, 20000);
```

程序设置spawn函数中option对象的detached属性，实现在后台执行timer.js程序：

```
const { spawn } = require('child_process');

const child = spawn('node', ['timer.js'], {
  detached: true,
  stdio: 'ignore'
});

child.unref();
```

独立子进程运行在不同的系统，有不同的行为。在Windows环境下，独立的子进程有独立的控制台窗口。在Linux环境下，独立的子进程将会成为新的进程组或会话的领导者。

在独立的子进程中调用unref函数，父进程可以可以独立于子进程终止运行。这一特性对于下面的场景很适用：子进程需要在后台运行很长时间、子进程的stdio流也要独立于父进程。

上面的示例代码中，设置option对象的detached属性为true ，独立的子进程在后台执行nodejs代码（timer.js）。设置option对象的option对象的stdio属性为ignore，子进程拥有独立于主进程的stdio流。这样就可以实现在子进程还是后台执行时，终止父进程。

![independent-stdio.gif](https://upload-images.jianshu.io/upload_images/704770-6cd97f4700bd2681.gif?imageMogr2/auto-orient/strip)


#### execFile函数
如果开发者不需要使用shell执行文件，execFile函数是一个不错的选择。execFile函数与exec函数很像，但是由于execFile并不会衍生新的shell，这是execFile函数比exec函数高效的主要原因。在Windows环境下，诸如.bat和.cmd文件并不能独立执行。但是可以通过exec函数或是设置spawn函数的shell特性执行这些文件。

#### ＊Sync函数
子进程模块中的spawn函数，exec函数和execFile函数同样有相应同步、阻塞函数。它们将会等待子进程执行完毕后退出。

```
const { 
  spawnSync, 
  execSync, 
  execFileSync,
} = require('child_process');
```
这些同步的函数对于简化所要执行的脚本或处理程序启动的任务都非常有用，但是在其它方面要避免使用它们。

#### fork函数

fork函数和spawn函数在衍生子进程时并不相同。它们的区别主要在于：通过fork函数衍生的子进程会建立通信管道，衍生的子进程可以通过send函数向主进程发送信息，主进程也可以通过send函数向子进程发送信息。下面是示例代码：

**父进程代码：**
```
const { fork } = require('child_process');

const forked = fork('child.js');

forked.on('message', (msg) => {
  console.log('Message from child', msg);
});

forked.send({ hello: 'world' });
```

**子进程代码：**
```
process.on('message', (msg) => {
  console.log('Message from parent:', msg);
});

let counter = 0;

setInterval(() => {
  process.send({ counter: counter++ });
}, 1000);
```

在父进程的程序中，开发者可以fork文件（这个文件将会通过node命令执行），然后监听message事件。当子进程调用process.send函数的时，父进程的message事件将会被触发。在上面的代码中，子进程每分钟都会调用一次process.send函数。

当从父进程向子进程传递数据时，在父进程中调用send函数后，子进程的message监听事件将会被触发，从而获取到父进程传递的消息。

当执行上面的父进程后，父进程将会向子进程传递对象{hello: 'world'}，然后子进程将会把这些父进程传递的消息打印出来。同时子进程将每隔一分钟向父进程发送一个递增的数字，这些数字将会在父进程控制窗口打印出来。

![fork.gif](https://upload-images.jianshu.io/upload_images/704770-cea7922f93fc8ba2.gif?imageMogr2/auto-orient/strip)


让我们看一个关于fork更实用的例子：

开发者在http服务上开启两个api。其中之一是"/compute"，在这个api上将会做大量的计算，计算过程将会占用很长时间。我们可以用一个for循环模拟上面的场景：
```
const http = require('http');
const longComputation = () => {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) {
    sum += i;
  };
  return sum;
};
const server = http.createServer();
server.on('request', (req, res) => {
  if (req.url === '/compute') {
    const sum = longComputation();
    return res.end(`Sum is ${sum}`);
  } else {
    res.end('Ok')
  }
});

server.listen(3000);
```

上面的程序存在一个问题：当http服务"/compute"被请求时，由于for循环阻塞了http服务的进程，因此http服务将不能再处理其它api请求。

由于请求的程序需要长期运行，因此我们可以设计出很多优化代码性能的方案。其中之一是通过fork函数衍生出新的子进程，然后将计算的代码放在子进程中运行，运行结束后将结果传输给父进程。

首先将longComputation函数封装在一个独立的js文件中，通过父进程的信息指令来执行longComputation函数：
```
const longComputation = () => {
  let sum = 0;
  for (let i = 0; i < 1e9; i++) {
    sum += i;
  };
  return sum;
};

process.on('message', (msg) => {
  const sum = longComputation();
  process.send(sum);
});
```

不需要在主进程中做longComputation函数中的运算，而是通过fork函数衍生出新的子进程，然后在子进程中计算，最后通过fork函数的信息传递管道将运算结果传回父进程中。

```
const http = require('http');
const { fork } = require('child_process');

const server = http.createServer();

server.on('request', (req, res) => {
  if (req.url === '/compute') {
    const compute = fork('compute.js');
    compute.send('start');
    compute.on('message', sum => {
      res.end(`Sum is ${sum}`);
    });
  } else {
    res.end('Ok')
  }
});

server.listen(3000);
```
当请求'/compute'时，子进程通过process.send函数将计算的结果传回给父进程，这样主进程的事件循环将不再发生阻塞。


然而上面代码的性能受限于程序可以通过fork函数衍生的进程数量。但是当通过http请求时，主进程并不会阻塞。

如果服务是通过多个fork函数衍生的子进程，Node.js的cluster模块将会对来自外部的请求，做http请求的负载均衡处理。这就会是我下个主题所要讲述的内容。

以上就是我关于这个主题所有的内容，非常感谢你的阅读，期待下次再见。