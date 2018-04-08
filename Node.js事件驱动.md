本文翻译自：[Understanding Node.js Event-Driven Architecture](https://medium.freecodecamp.org/understanding-node-js-event-driven-architecture-223292fcbc2d)

![event-drive.jpeg](https://upload-images.jianshu.io/upload_images/704770-ebf164104a4eb4df.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

许多Node.js模块（诸如Http requests、responses、streams等）内置了EventEmitter模块，因此这些模块可以通过emit和listen实现事件的触发和监听。

![event-emmiter.png](https://upload-images.jianshu.io/upload_images/704770-d4592ff3ce92ca87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

事件驱动的本质是：以类似回掉函数的方式，实现流行的Node.js函数的调用（诸如 fs.readFile）。按照这种说法，当Node.js的"callback函数"准备就绪后，事件一旦被处罚，"回调函数"将作为事件的处理程序。

*让我们一起探索最基本的实现形式。*

#### Node当你准备好了，调用我

Node处理异步最原始的方法机会是回调函数，那是在很久以前，Node还没有内置promises和async／await的特性。

回调函数作为就是传递给其它函数的参数。这对Javascript是可行的，因为函数是第一类对象。

回调函数并不意味着异步调用，这对于理解回调函数是至关重要的。在方法体中，调用回调函数既可以是同步也可以是异步调用。

例如下面的函数fileSize，它接收回调函数作为参数并且根据不同的条件以同步或是异步的方式调用该回调函数。


```
const readFileAsArray = function(file, cb) {
  fs.readFile(file, function(err, data) {
    if (err) {
      return cb(err);
    }
    const lines = data.toString().trim().split('\n');
    cb(null, lines);
  });
};
```

**readFileAsArray**有两个参数：文件路径和回调函数。readFileArray读取文件的内容，并把行内容切开成数组，最后把得到的数组传递给回调函数中。

下面将是我们应用回调函数的例子。假设在相同的路径下存在一个numbers.txt文件，文件的内容如下：

```
10
11
12
13
14
15
```

如果我们要计算这个文件有多少奇数，我们可以使用下面readFileAsArray函数中的代码：
```
readFileAsArray('./numbers.txt', (err, lines) => {
  if (err) throw err;
  const numbers = lines.map(Number);
  const oddNumbers = numbers.filter(n => n%2 === 1);
  console.log('Odd numbers count:', oddNumbers.length);
});
```

上面的代码实现了：读取文件中的内容，把内容转换为数组，计算数组中的奇数。

这段代码是典型的Nodejs回调函数。在回调函数中遵循错误优先的原则，错误信息的参数可以为空，回调函数的结果作为回调函数第二个参数。开发者都应该遵循这条原则，因为假定其他的代码都是按照这种原则。

#### 现代Javascript对回调函数的改进

在现代的Javascript中，我们有了promise对象。Promise可以看作是对回调函数作异步调用的优化接口。不再通过将回调函数的结果和回调函数可能出现的错误作为回调函数的参数。Promise对象允许代码分别处理函数成功返回结果和函数出现的错误，通过链条的方式处理多个异步回调函数，避免出现回调嵌套的地狱。

如果readFileAsArray支持promise，我们就可以写出下面的代码：
```
readFileAsArray('./numbers.txt')
  .then(lines => {
    const numbers = lines.map(Number);
    const oddNumbers = numbers.filter(n => n%2 === 1);
    console.log('Odd numbers count:', oddNumbers.length);
  })
  .catch(console.error);
 ```
通过调用.then函数获取异步函数的结果，而不是将结果放进回调函数中。 .catch函数可以获取回调函数的异常信息。

由于新Promise对象的出现，让现代的Javascript代码很方便的支持promise接口。

下面的代码就是对回调函数进行异步调用的另一种封装，通过promise对象实现readFileAsArray函数的一部调用：
```
const readFileAsArray = function(file, cb = () => {}) {
  return new Promise((resolve, reject) => {
    fs.readFile(file, function(err, data) {
      if (err) {
        reject(err);
        return cb(err);
      }
      const lines = data.toString().trim().split('\n');
      resolve(lines);
      cb(null, lines);
    });
  });
};
```

在fs.readFile函数外面包裹这Promise对象。通过Promise对象暴露出的两个参数（一个resolve函数，另一个reject函数）实现函数的封装。

当我们想引用异步函数中的错误信息，可以调用reject函数。当我们想使用异步函数返回的数据，可以调用resolve函数。

#### 通过async／await 调用promise对象

通过Promise对象异步回调的接口，可以使代码在需要异步回调时变的非常简单，但是随着回调的增多，代码也会显得很凌乱。

Promise对象对异步回调优化了一点，Generator函数在Promise对象的基础上又又优化了一点。对异步函数调用最友好的方式还要数async，通过async函数我们可以以同步编程的方式，实现异步调用。这种编程方式的出现使得异步代码的可读性有了质的提升。

下面的代码就是如何使用async／await调用readFileAsArray函数：

```
async function countOdd () {
  try {
    const lines = await readFileAsArray('./numbers');
    const numbers = lines.map(Number);
    const oddCount = numbers.filter(n => n%2 === 1).length;
    console.log('Odd numbers count:', oddCount);
  } catch(err) {
    console.error(err);
  }
}
countOdd();
```

我们先是创建async函数，就是在普通函数前添加async关键字。在async函数内部，我们调用readFileAsArray就像它返回行变量一样。为了实现这个功能，我们使用关键字await。之后，我们继续执行代码，就好像readFileAsArray调用的时同步函数一样。

这样对异步回调的处理，使的代码变的很简单而且易读。为了获取代码中错误信息，我们需要在代码外面包裹一层try／catch。

通过Async／await的新特性，我们不在需要在代码写一些特殊的接口（像 .then 和 .catch）。我们只不过是在一些纯Javascript代码的基础上，使用一些函数标记。

我们可以对任何封装promise对象的函数使用async／await关键字。然而我们不能使用在回调式的异步函数上（如setTimeout）。

#### EventEmitter模块

EventEmitter以Node.js异步事件驱动为内核，实现促进Node.js中对象间的通信。Node.js许多内置模块都是继承自EventEmitter。

EventEmitter的代码很简单：事件的触发对象触发已经注册的监听其。因此，事件触发对象通常有两个特点：

- 触发已经注册的事件
- 注册事件或移除注册事件监听器

对象继承EventEmitter，就可以使用EventEmitter。

```
class MyEmitter extends EventEmitter {

}
```

通过实例化已经继承EventEmitter的类生成触发事件的对象。

```
const myEmitter = new MyEmitter();
```

在事件触发对象整个生命周期中，我们通过触发事件名，触发任何我们想要作用的事件监听器。

```
myEmitter.emit('something-happened');
```

触发事件监听器是新条件出现，这些新条件通常是事件触发对象内部状态改变的信号。

我们通过on函数注册事件监听器，每当对象触发事件监听器的事件名，这些监听事件将会执行。

#### 事件并不就是异步

*让我们看一个示例代码：*

```
const EventEmitter = require('events');

class WithLog extends EventEmitter {
  execute(taskFunc) {
    console.log('Before executing');
    this.emit('begin');
    taskFunc();
    this.emit('end');
    console.log('After executing');
  }
}

const withLog = new WithLog();

withLog.on('begin', () => console.log('About to execute'));
withLog.on('end', () => console.log('Done with execute'));

withLog.execute(() => console.log('*** Executing task ***'));
```

WithLog是事件触发对象，在对象内部定义一个execute函数。这个函数接收一个参数，这个任务函数是一个打印函数。在这个任务函数执行前后都触发事件。

为了看清楚事件执行的先后顺序，我们注册了相应名字的事件监听器，最后我们执行WithLog对象中execute函数。

*下面是函数执行的结果：*

```
Before executing
About to execute
*** Executing task ***
Done with execute
After executing
```

我想让大家注意到的是这里的输出内容都是同步的。在这段代码中没有任何异步的行为发生。

- 输出第一行是"Before executing"
- 以begin命名的事件输出"About execute"
- 通过参数传递的函数输出"***Executing task***"
- 以begin命名的事件输出"Done with execute"
- 最后输出"After executing"

就像古老的回调函数，不假设事件是同步还是异步执行。

我们可以假设下面的测试用例（在withLog对象中的execute函数是setImmediate）：

```
// ...

withLog.execute(() => {
  setImmediate(() => {
    console.log('*** Executing task ***')
  });
});
```

现在函数输出将会是下面：

```
Before executing
About to execute
Done with execute
After executing
*** Executing task ***
```

在异步回调函数后，执行"Done with execute"和"After executing"将不在正确。

我们需要回调函数（或promises对象）与事件驱动的对象相结合，实现在异步调用后执行事件触发。上面的例子就很好的说明这一点。

事件相对于传统回调函数还有另一个优势，程序可以通过定义不同的监听对象，实现多次触发相同的函数。如果通过回调函数实现相同的功能，则需要在函数中些许多逻辑。事件是实现在程序核心基础上，通过外部插件构建函数的好方法。你可以把事件想象成勾子点，通过勾子点状态的变化定制一些函数。

#### 异步的事件

我们将同步的示例转换为异步将会更有利于我们理解，

```
const fs = require('fs');
const EventEmitter = require('events');

class WithTime extends EventEmitter {
  execute(asyncFunc, ...args) {
    this.emit('begin');
    console.time('execute');
    asyncFunc(...args, (err, data) => {
      if (err) {
        return this.emit('error', err);
      }

      this.emit('data', data);
      console.timeEnd('execute');
      this.emit('end');
    });
  }
}

const withTime = new WithTime();

withTime.on('begin', () => console.log('About to execute'));
withTime.on('end', () => console.log('Done with execute'));

withTime.execute(fs.readFile, __filename);
```

WithTime类执行异步函数（asyncFunc），在异步函数中通过console.time和console.timeEnd输出时间，并在异步函数执行的前后触发相应的事件。如果异步函数抛出异常，将会触发error/data事件。

我们使用fs.readFile函数作为测试用例中的异步函数。通过事件监听，我们就可以代替回掉函数实现异步调用。

代码执行后，相应事件按顺序触发并且得到异步函数的执行时间。下面是我们得到的运行结果：

```
About to execute
execute: 4.507ms
Done with execute
```

**请注意**上面的例子是我们通过回调函数和事件相结合完成的。如果我们让回调函数支持promise对象，我们就可以通过async/await实现相同的功能。

```
class WithTime extends EventEmitter {
  async execute(asyncFunc, ...args) {
    this.emit('begin');
    try {
      console.time('execute');
      const data = await asyncFunc(...args);
      this.emit('data', data);
      console.timeEnd('execute');
      this.emit('end');
    } catch(err) {
      this.emit('error', err);
    }
  }
}
```

我不知道你们是怎么认为的，在我看来，这种实现方式要比基于回调函数的代码或是.then/.catch的代码要更加易读。
async/await的功能使我们尽可能接近JavaScript语言本身，我认为这是一个巨大的胜利。

#### 事件参数和错误

在上面的例子中，有两个事件触发时还传了额外的参数。

*异常事件触发异常对象*

```
this.emit('error', err);
 ```

*数据事件触发数据对象*

```
this.emit('data', data);
```

我们可以在触发事件函数中，事件名后添加尽可能多的参数，所有这些参数将会传递到事件监听器上。

例如下面的数据事件：我们注册的事件监听器，将会获取我们触发事件时传递进去的参数（事件名除外）。data对象就是异步函数asyncFunc返回的数据。

```
withTime.on('data', (data) => {
  // do something with data
});

```

在我们使用回掉函数实现异步调用的例子中，如果我们不去监听异常事件，程序将会退出。

为了说明这一点，在原示例的基础上，添加调用产生异常的函数。

```
class WithTime extends EventEmitter {
  execute(asyncFunc, ...args) {
    console.time('execute');
    asyncFunc(...args, (err, data) => {
      if (err) {
        return this.emit('error', err); // Not Handled
      }

      console.timeEnd('execute');
    });
  }
}

const withTime = new WithTime();

withTime.execute(fs.readFile, ''); // BAD CALL
withTime.execute(fs.readFile, __filename);
```
上面例子中，WithTime类第一次执行将会产生异常。程序将会崩溃并退出。

```
events.js:163
      throw er; // Unhandled 'error' event
      ^
Error: ENOENT: no such file or directory, open ''
```

WithTime类第二次执行将因为程序的崩溃，受到影响，从而不能被执行。

如果在代码中注册异常监听器，node程序的生命周期将会发生变化。如下例：

```
withTime.on('error', (err) => {
  // do something with err, for example log it somewhere
  console.log(err)
});
```

如果按照上面的方法，第一次执行execute函数产生的异常将会被捕获，node的生命周期也不会终止。这样就不会影响代码继续向下执行， 在程序控制端将会输出：

```
{ Error: ENOENT: no such file or directory, open '' errno: -2, code: 'ENOENT', syscall: 'open', path: '' }
execute: 4.276ms
```

值得注意，程序现在的行为，与以promise对象为基础的函数的行为不相同。仅仅输出一个警告，但是程序的正常运行并不会受到影响。

```
UnhandledPromiseRejectionWarning: Unhandled promise rejection (rejection id: 1): Error: ENOENT: no such file or directory, open ''
DeprecationWarning: Unhandled promise rejections are deprecated. In the future, promise rejections that are not handled will terminate the Node.js process with a non-zero exit code.
```

通过触发在全局定义的uncaughtException事件监听器，是另外一种捕获异常的方式。然后，使用全局事件监听器捕获异常是不明智的选择。

一般建议避免使用全局uncaughtException事件监听器，但是如果你必须要这么做（报告发生了异常或是清理缓存），这时必须要终止程序。像如下代码：

```
process.on('uncaughtException', (err) => {
  // something went unhandled.
  // Do any cleanup and exit anyway!

  console.error(err); // don't do just that.

  // FORCE exit the process too.
  process.exit(1);
});
```

但是，如果在同一时间发生多次触发异常事件。这就意味着uncaughtException事件监听器将会被多次触发，这对于清理缓存（清除内存），将会是灾难。一个例子是对数据库关闭的操作可能会被多次触发。

EventEmitter模块暴露一个once方法。即使多次通过once触发事件监听器，这个方法只会让事件监听器触发一次。因此，这是一个使用uncaughtException事件监听器很实用的方法，因为第一次触发出现异常时，我们就会做一些数据库或内存的清理，然后退出程序。

#### 监听事件的次序

如果对同一事件注册多个监听器，那么这些监听器的调用将会按顺序进行。

```
// प्रथम
withTime.on('data', (data) => {
  console.log(`Length: ${data.length}`);
});

// दूसरा
withTime.on('data', (data) => {
  console.log(`Characters: ${data.toString().length}`);
});

withTime.execute(fs.readFile, __filename);
```

上面代码的执行后，输出的结果是"Length"行在"Characters"行之前，因为我们定义的事件监听器将会按顺序执行。

如果你定义新的事件监听器，我们要想做这个事件监听器最先被触发，可以使用prependListener方法。

```
// प्रथम
withTime.on('data', (data) => {
  console.log(`Length: ${data.length}`);
});

// दूसरा
withTime.prependListener('data', (data) => {
  console.log(`Characters: ${data.toString().length}`);
});

withTime.execute(fs.readFile, __filename);
```

上面的代码将会使"Characters"的结果将会被最先输出。

最后，如果需要删除事件监听器，我们可以使用removeListener方法。

这就是我关于这个主题的所有阐述，非常感谢您的阅读，期待下次与你相遇。
