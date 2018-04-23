文章翻译子[Scaling Node.js Applications](https://medium.freecodecamp.org/scaling-node-js-applications-8492bd8afadc)

*你应该知道所有关于Node.js的可伸缩性*

![scalability.png](https://user-gold-cdn.xitu.io/2018/4/19/162dce6d989ae69c?w=1480&h=668&f=png&s=145092)

可伸缩性并不是扩展Node.js应用的第三方包的性能，它是Node.js(Javascript运行时环境)的核心功能。Node.js取名为节点(Node)，这强调Node.js应用可以通过相互通信的分布式节点向外提供服务。

你是否将你的Node.js应用部署在多个微服务上？你是否为生产环境cpu的每个内核启动一个Node.js进程？你是否对已经启动的Node.js进程做负载均衡？你是否知道Node.js的内置模块可以帮助你实现上述功能？

Node.js集群(cluster)模块不仅为充分利用服务器cpu性能提供一种开箱即用的解决方案，而且可以提升Node.js进程的性能，还可以在零停机的情况下重启服务器。这边文章不仅涵盖上述所有内容，而且还有更多鲜为人知的知识。


## 可伸缩策略

对应用做负载均衡，可以增强应用的可伸缩性，但这并不是唯一的原因。负载均衡还可以增强应用的可用性，提高对Node.js应用的生命力(不会因为单个Node.js进程阻塞，而导致Node.js应用死亡)。

#### 克隆

提高应用可伸缩性的最简单方式是将应用克隆很多次，与克隆后的应用一起分担外部的数据请求(负载均衡)。这种策略不会增加开发的时间，但是很高效。使用Node.js的cluster(集群)模块，可以使开发者通过最小的开发量对单进程的服务实现克隆策略。

#### 分解

根据应用的功能或者服务对程序进行分解，从而提供应用的伸缩性。这就意味着将会有多个应用程序，这些应用程序可能由不同的代码构成、连接不同数据库和对外提供不同API接口。

这个策略通常是将多个微服务联合在一起，其中“微”的字面意思是服务尽可能的小。在现实场景下，服务的大小并不是最重要的。但是各个服务必须是高内聚、低耦合的。

这种策略实施起来并不容易，可能长期存在不可预测的风险，但是在开发中充分运用这一特性依旧是非常必要的。

#### 切割

将应用根据切割成多个实例，每个实例仅仅负责一部分应用的数据。这种策略也叫做数据库的水平分区或水平分片。数据分区在操作前需要进行一次查表，通过查询的结果，调用相应的分区或分片。例如：根据用户的国家或语言切割用户，在每次调用数居前，要去查询用户的国家或者用户的语言。

实现可伸缩性的大型应用，最终都会使用上述三种策略。Node.js可以很容易实现上面三种策略，但是在这篇文章中仅仅集中在克隆策略以及对克隆应用的Node.js内置工具做一些探索。


**请注意**你在阅读本篇文章前需要对Node.js的子进程有一些必要的了解。如果还没有深入理解，我推荐你阅读我另外一篇文章：

[你应该知道的Node.js子进程](https://juejin.im/post/5ad6b1686fb9a028ce7c1f80)

## 集群模块

集群模块充分挖掘服务器多核cpu的物理性能，实现负载均衡。它使用子进程模块的fork方法，衍生出与服务器内核数量的子进程。当有外部向主进程多个请求时，集群模块将请求均匀分配给衍生子进程。

集群模块是Node.js为开发者提供的在单服务器增强应用伸缩性的的“帮助器”。如果你的服务器有足够的物理资源？如果对服务器增添物理资源的成本小于添加多台服务器？集群模块将是快速克隆应用的最好选择。

即便小服务器也会有多核cpu？即使你不担心你的Node.js服务的负载？你都应该使用集群模块来提高服务的伸缩性以及增强服务的容错能力。对于进程管理工具PM2来说，只要在PM2命令后添加一个参数，就可以上述的功能。

本文将着重介绍如何使用Node.js原生模块实现负载均衡：

集群模块的工作原理很简单。开发者先创建一个主进程，然后通过fork方法衍生出多个工作进程，当请求数据时主进程控制子进程的调度。每个工作进程都是应用的一个实例，所有的请求都由主进程分配给子进程处理。


![master.png](https://user-gold-cdn.xitu.io/2018/4/20/162e0de336ab5947?w=2000&h=1125&f=png&s=192886)

主进程使用轮询调度算法(roud-robin algorithm)，对子进程分配请求的任务。除了Windows，所有平台都支持集群模块。开发者还可以在全局自定义的调度算法。

轮询调度算法(roud-robin-algorithm)虚拟所有可用进程首尾相接形成圆，第一个请求分配给圆上第一个子进程，第二个请求分配给圆上第二个子进程，以此类推。当圆上最后一个子进程分配请求后，调度算法将从圆上第一个进程开始分配请求任务。

轮询调度算法是最简单和最实用的调度算法，但是还有其它选择。最有特色的算法是可以根据任务的优先权选择负载最小的进程或响应最快的进程。

#### 对HTTP服务做负载均衡

下面是对Node.js的hello-word代码做一些简单修改，在响应请求前做大量计算的代码：

```
// server.js
const http = require('http');
const pid = process.pid;

http.createServer((req, res) => {
  for (let i=0; i<1e7; i++); // simulate CPU work
  res.end(`Handled by process ${pid}`);
}).listen(8080, () => {
  console.log(`Started process ${pid}`);
});
```

为了验证我们创建的平衡器是否可以工作，将进程pid放到HTTP响应对象中，根据进程的pid决定哪个子进程处理请求。

使用cluster模块将主进程克隆成多个子进程前，先测算Node.js主进程服务每秒钟可以处理的请求数量并将测算的值作为性能基准。我们可以使用[Apache benchmarking tool](https://httpd.apache.org/docs/2.4/programs/ab.html)。当启动上面的node.js服务后，执行下面的ab命令：

`
ab -c200 -t10 http://localhost:8080/
`

这条命令在10秒钟向服务器并发请求200次。


![benchmark.png](https://user-gold-cdn.xitu.io/2018/4/20/162e1bb4a07c8df5?w=2000&h=1125&f=png&s=743033)

在我的设备上，单节点服务器每秒处理51次请求。由于不同设备的性能表现并不一样，这也仅仅是个简单的测试，并不是百分之百正确。但是作为性能基准可以与对服务做集群后的性能作对比。

保留上面的server.js文件，我们创建新的文件cluster.js作为主进程：

```
// cluster.js
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
  const cpus = os.cpus().length;

  console.log(`Forking for ${cpus} CPUs`);
  for (let i = 0; i<cpus; i++) {
    cluster.fork();
  }
} else {
  require('./server');
}
```

在cluster.js中，我们首先引用了cluster模块和os模块。使用os.cpus()获取运行服务器的cpu数量。

集群模块提供一个布尔类型的isMaster变量确定cluster.js文件是不是作为主进程。程序第一次执行这个文件时，isMaster变量是true，cluster.js文件将会作为主进程。这种情况下，主进程将会衍生出与cpu数量相等的子进程数。

当在主进程中执行cluster.fork函数后，在子进程中将会再次执行当前文件(cluster.js)。在子进程中执行cluster.js时，变量isMaster的值为false，此时子进程中存在另外一个值为true的变量isWorker。

子进程运行的应用是真正向外提供服务的程序。在那里需要我们写真正的服务逻辑，例如上面的例子中，我引用的server.js是真正响应请求的代码。

Node.js集群模块实现应用的伸缩性的代码基本就是这样。通过集群模块开发者可以充分利用服务器的物理性能。要测试集群模块，可以运行cluster.js：


![cluster.png](https://user-gold-cdn.xitu.io/2018/4/20/162e21c3bde47e6b?w=708&h=548&f=png&s=212474)

我的机器的cpu是8核的，因此Node.js开启了8个进程。**注意这里的进程与普通Node.js进程是不完全相同的，**每个进程都有独立的事件循环机制和内存空间。

当我们多次请求服务时，这些请求将会被分配给不同的进程进行处理。由于集群模块在选择子进程处理请求时会做一些优化，因此主进程并不会严格按照顺序轮询子进程响应请求，但是请求的负载将会被分配给不同的子进程上。

我们可以使用与上面一样的ab命令，测试集群模块负载均衡的性能：


![advanced-performance](https://user-gold-cdn.xitu.io/2018/4/20/162e224de9600af9?w=2000&h=1125&f=png&s=792428)

通过集群模块优化后，部署在我机器上的服务每秒钟可以处理181次请求。而只使用Node.js主进程的服务每秒钟仅仅可以处理51次请求。我们仅仅对这个简单应用修改了几行代码就让性能翻了很多倍。

#### 向所有子应用广播消息

*注意这里所说的子应用是指子进程中的应用*

由于集群模块是使用child_process.fork衍生子进程，这样就可以通过主进程与子进程之间的通信管道，实现主应用与子应用的通信。

根据上面server.js／cluster.js例子，使用cluster.workers获取子应用的集合。这是指向所有子应用的引用，可以获取所有子应用的信息，只要使用for循环子应用就可以向所有的子应用广播消息。例如：

```
Object.values(cluster.workers).forEach(worker => {
  worker.send(`Hello Worker ${worker.id}`);
});
```

通过Object.values获取cluster.workers中的所有子应用对象，然后for each便利这些子应用，最后使用send函数向所有子应用广播消息。

在子应用中(例子指的是server.js)，对全局进程对象注册message事件，可以获取来自主进程发送的消息。例如：

```
process.on('message', msg => {
  console.log(`Message from master: ${msg}`);
});
```

下面是对cluster/server做两个额外测试的结果：

![aditional-performance](https://user-gold-cdn.xitu.io/2018/4/20/162e24472b808bd9?w=2000&h=1125&f=png&s=700334)

可以看出两点：
- 每个子应用都从主应用那里获取了信息
- 子应用并不是按顺序分配外部请求的

接下来我们对示例代码做更接近实际应用的修改：请求服务获取数据库中user表中的数据。通过mock函数返回数据表中用户数，每次调用mock函数都会返回当前cout变量的平方值：

```
// **** Mock DB Call
const numberOfUsersInDB = function() {
  this.count = this.count || 5;
  this.count = this.count * this.count;
  return this.count;
}
// ****
```

为了避免多次请求数据库，我们每个隔一段时间做一次数据库缓存，例如10秒钟。然而我并不想衍生的8个子应用每隔10秒钟分别向数据库请求一次。我们可以在主应用中向数据库发起请求，然后将请求得到的数据通过通信接口传递给8个子应用。

在主应用中，我们可以像下面使用forEach函数向8个应用广播主应用请求的数据：

```
// Right after the fork loop within the isMaster=true block
const updateWorkers = () => {
  const usersCount = numberOfUsersInDB();
  Object.values(cluster.workers).forEach(worker => {
    worker.send({ usersCount });
  });
};

updateWorkers();
setInterval(updateWorkers, 10000);
```

当第一次调用updateWorkers函数后，setInternval函数每隔10秒调用一次updateWorkers函数。这样主应用就可以每隔10秒都会访问一次数据库，然后通过通信管道传输向子应用广播访问请求数据库的数据。

在服务端的代码中，我们通过注册message事件获取主应用传输的usersCount值。使用全局变量缓存usersCount数据，这样就可以随时使用usesCount变量。

例如下面代码：

```
const http = require('http');
const pid = process.pid;

let usersCount;

http.createServer((req, res) => {
  for (let i=0; i<1e7; i++); // simulate CPU work
  res.write(`Handled by process ${pid}\n`);
  res.end(`Users: ${usersCount}`);
}).listen(8080, () => {
  console.log(`Started process ${pid}`);
});

process.on('message', msg => {
  usersCount = msg.usersCount;
});
```

当有外部请求时，将usersCount作为响应对象。如果现在要测试集群，在刚开始的10秒内你获得的users count数据为25.下一个10秒内你获得的users count数据为625。

因此非常感谢主进程与子进程的信息管道，让集群有了通信基础。

#### 提升服务的可用性

在服务器上仅仅部署一个实例服务对象会存在下面的缺点：如果服务的实例对象崩溃了，服务必须要在重启后才能继续对外提供服务。即便进程可以自动重启服务，这也意味着在服务崩溃后和重启前存在一个时间段。

另外重启服务部署新的代码也会存在同样的问题。只要是仅仅通过一个实例(节点)，服务停机的时间就会影响应用的可用性。

如果服务有多个实例(节点)，程序就可以通过简单几行代码增强服务的可用性。

在setTimeout函数中设置随机的时间后调用process.exit函数，模拟服务进程随机崩溃：

```
// In server.js
setTimeout(() => {
  process.exit(1) // death by random timeout
}, Math.random() * 10000);
```

如果作为服务的子应用崩溃了，主应用通过在cluster对象上注册exit事件获取子应用退出的信息。当子应用退出程序时，主应用在注册事件的回调函数中重新衍生出一个新的子应用。例如：

```
// Right after the fork loop within the isMaster=true block
cluster.on('exit', (worker, code, signal) => {
  if (code !== 0 && !worker.exitedAfterDisconnect) {
    console.log(`Worker ${worker.id} crashed. ` +
                'Starting a new worker...');
    cluster.fork();
  }
});
```

最好在上面代码的基础上加上一个条件，在子进程在开发者手动断开连接或是被主进程故意杀死的情况下，主进程不会重新衍生新的进程。例如，主进程根据负载模式发现应用使用太多的资源，它可能会主动杀死一些子进程。在这种场景下，变量existedAfterDisconnect的值是true。如下面的程序：

```
const cluster = require('cluster');
const os = require('os');

if (cluster.isMaster) {
    const spus = os.cpus().length;
    for (let i=0; i< cpus; i++) {
        cluster.fork();
    }
    
    cluster.on('exit', (worker, code, signal) => {
        if (code !== 0 && !worker.existedAfterDisconnect) {
            console.log('Worker ${worker.id} crashed. ' + 'Starting a new worker ....' );
        
            cluster.fork();
        }
    });
} else {
    require('./server')
}
```

部署上面的代码，子进程在随机时间内会崩溃，主进程立即衍生出新的子进程以提高应用的可用性。由于一些请求不可避免的要面对子进程的崩溃，因此程序不可能对所有的请求做响应。开发者可以通过ab命令测量应用的可用性：


![availability.png](https://user-gold-cdn.xitu.io/2018/4/21/162e66a42abd1408?w=2000&h=1125&f=png&s=840140)

通过测试百分之九十九的请求都可以得到响应。仅仅通过简单的几行代码，开发者就不在担心程序的崩溃。主进程就像眼睛一样替开发者盯住进程的运行状况。

#### 重启零停机

当我们需要部署新的代码，如何实现零停机重启服务？

当很多子进程正在运行，不是将它们全部重启，一次仅仅重启一个子进程。这样就可以保证在子进程重启时，其它的子进程仍然可以处理外部请求。

使用Node.js内置模块cluster可以很容易实现上述示例。如果主进程一旦启动，我们不想在此重启主进程。在这种情况下，我们需要向主进程发送命令，让这条命令指挥它重启子进程。在linux系统上可以使用下面的方式实现：先在主进程上监听SIGUSR2事件，开发者可以使用"kill  进程的pid"命令触发主进程监听的SIGUSR2事件。实现如下：

```
// In Node
process.on('SIGUSR2', () => { ... });


// To trigger that
$ kill -SIGUSR2 PID
```

通过上述方式，可以在不杀死主进程的情况下，通过命令引导主进程工作。由于SIGUSR2信号是用户命令，因此这条命令非常适合向主进程传递信号。如果你对为什么不使用SIGUSR1信号有疑问？这是因为Node.js使用SIGUSR1信号做debugger调试，不是使用它主要是避免发生冲突。

然而不幸的是在windows系统上并不支持上述的进程信号，我们必须通过其它方式引导主进程。我这里有一些替代方案。例如，1.使用标准的输入或套接字输入。 2. 监听进程的pid文件的存在和删除事件。这里为了让示例更简单，我仅仅假设Node.js服务是部署在Linux系统上。

Node.js服务在Windows系统上可以很好的工作，但是我认为将服务部署在Linux系统上是更安全的选择。这不但是由于Node.js自身的原因，而且许多生产环境的工具在Linux系统上更加稳定。**以上仅仅是一家之言，你可以完全忽略**

*顺便说一下，在最近的Windows系统上可以安装Linux系统。我在Windows的子linux系统上测试过，并没有明显的性能改进。如果正在使用的生产环境是Windows系统，你可以查一下[Bash on Windows](https://docs.microsoft.com/zh-cn/windows/wsl/about)，然后试一试Windows中子Linux的系统表现。*

让我们再次回到最上面的例子上，当主进程接收到SIGUSR2的信号时，这就意味着是时候重启子进程了，并且要求每次仅仅重启一个子进程。

在开始任务前，需要使用cluster.workers函数获取当前子进程的引用，并将它保存在数组中：

`
const workers = Object.values(cluster.workers);
`

然后，向restartWorker函数中传递将要重启的子进程在子、进程数组中的序号。然后在函数中递归调用restartWorker函数，传递的参数是当前进程的序号加一，这样就可以实现按顺序重启子进程。下面是我使用的restartWorker函数代码：

```
const restartWorker = (workerIndex) => {
  const worker = workers[workerIndex];
  if (!worker) return;

  worker.on('exit', () => {
    if (!worker.exitedAfterDisconnect) return;
    console.log(`Exited process ${worker.process.pid}`);
    
    cluster.fork().on('listening', () => {
      restartWorker(workerIndex + 1);
    });
  });

  worker.disconnect();
};

restartWorker(0);
```

在restartWorker函数中，我们获取子进程的引用后重启子进程。由于程序需要按照子进程在子进程数组中的位置递归调用restartWorker函数，因此程序需要一个终止递归的条件。当程序所有子进程都已经重启后，调用return结束函数。使用worker.disconnect函数终止子进程，但是在重启下一个子进程前需要衍生出新的子进程代替正在终止的子进程。

对当前进程注册exit事件，当前进程退出时，触发该事件。但是开发者必须确保退出子进程的行为是由于调用disconnect函数。如果exitedAfetrDisconnect变量是false，说明子进程不是由于调用disconnect函数导致的。这是直接调用return，不再继续往下处理。如果exitedAfetrDisconnect变量是true，程序继续往下执行并且衍生出新的子进程代替正在退出的进程。

当衍生新的子进程后，程序继续重启子进程数组中下一个进程。但是衍生的子进程函数并不是同步的，因此不能在调用衍生函数后直接重启下一个子进程。然而，程序可以在衍生函数后注册listening事件。当新的衍生进程正常工作后触发listening事件，然后程序就可以按顺序重启子进程数组中下一个进程。

为了测试上述代码，我们应该先获取主进程的pid，然后将它作为SIGUSR2信号的参数。

`
console.log(`Master PID: ${process.pid}`);
`

启动集群服务，获取主进程的PID，然后使用kill -SIGUSR2 PID命令重启子进程。在重启子进程期间，使用ab命令测试程序的性能表现。测试结果发现没有丢失任何一个请求：

![reatart-process](https://user-gold-cdn.xitu.io/2018/4/22/162ec3b03ae62990?w=2000&h=1125&f=png&s=721430)

在生产环境下，我通常使用进程检测器(PM2)。PM2提供许多监测Node.js应用的命令，使用PM2处理各种任务都非常简单。例如：只要在命令参数后面添加-i，就可以对应用使用集群功能：

`
pm2 start server.js -i max
`

如果需要实现零停机重启子进程，可以使用下面命令：

`
pm2 reload all
`

#### 共享状态和粘滞负载均衡

事物总是有利有弊。对Node.js应用做负载均衡时，应用必然会失去单进程所具有的许多特性。就像其它开发语言都要面对如进程间共享数据这样的进程安全问题。在我们这里，就是多个子进程共享数据的问题。

例如启动集群后，由于每个子应用都有自己的内存，因此不能将应用数据缓存在子应用的内存中。如果开发者将数据缓存在子应用的内存中，其它子应用将没有权限访问这些数据。

如果需要缓存集群应用的数据，开发者需要使用独立的实体，这个实体可以向所有子应用提供读／写数据的API。这个实体可以是数据服务、如果你喜欢使用内存缓存，可以使用redis数据服务或者创建一个提供读／写API的Node.js进程，帮助子进程相互通信。

尽管如此，也不要认为使用独立的数据服务是集群的弊端。使用独立的缓存服务是通过分解策略增强Node.js应用的可伸缩性的一种方式。即便Node.js应用部署在单核服务器上，也推荐开发者使用这种方式实现数据分离。

除了数据缓存，状态通信也会存在问题。例如在集群应用中，某个子服务对外部请求做状态标记后，并不能确保当该外部客户端再次发送请求时，该请求可以分配给与上次相同的子服务上。因此在子服务上创建程序的状态标记并不是好选择。

最常见的问题就是用户的认证：


![](https://user-gold-cdn.xitu.io/2018/4/22/162ec94d5b905909?w=2000&h=1125&f=png&s=142839)

假设集群服务将外部请求分配给子应用A：


![](https://user-gold-cdn.xitu.io/2018/4/22/162ec969da72605e?w=2000&h=1125&f=png&s=159979)

在子应用A上标记了该用户的状态。然而，当同一个用户再次请求服务时，主进程可能会将请求任务分配给没有标记用户状态的子进程。这时将用户认证的对话保存在一个子进程中，程序将不能工作。

这个问题其实有很多中解决方式：可能将用户的认证状态保存在共享数据库(如Redis)。然而使用这种策略需要对代码做出改动，因此并不总会选择这种方案。

**注意：**如果开发者不想通过修改代码在共享数据库中存储请求信息，其它的方式会很低效。这里可以考虑**粘滞负载均衡**方案。由于许多负载均衡器都支持这种策略，因此可以很轻松完成对应的代码。原理其实很简单：如果用户的会话信息保存在子进程A上，在主进程会存储会话与进程的信息。

![](https://user-gold-cdn.xitu.io/2018/4/22/162eca9208e162a7?w=2000&h=1125&f=png&s=169554)

当相同用户再次请求服务时，程序会在主进程的索引表上查询存储该用户会话信息的进程，然后将请求任务分配给该子进程。这里我们并没有对请求实现负载均衡，如果修改程序受限的场景下，这也是不错的选择。

Node.js集群模块并不支持粘滞负载均衡，但是许多负载均衡器的默认配置是支持它的。

这就是关于这个主题的全部内容，感谢您的阅读。




