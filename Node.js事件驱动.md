![event-drive.jpeg](https://upload-images.jianshu.io/upload_images/704770-ebf164104a4eb4df.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

大多数Node.js模块（诸如Http requests、responses、streams等）内置了EventEmitter模块，因此这些模块可以通过emit和listen实现事件的触发和监听。

![event-emmiter.png](https://upload-images.jianshu.io/upload_images/704770-d4592ff3ce92ca87.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

事件驱动的最简单实现形式是：一些流行的Node.js方法中（诸如 fs.readFile）的回调函数。按照这种说法，当Node.js的callback函数准备就绪，事件一旦被处罚，回调函数将作为事件的处理程序。

*让我们一起探索最基本的实现形式。*

#### Node当你准备好了，调用我

Node处理异步最原始的方法机会是回调函数，那是在很久以前，那时Node还没有内置promises和async／await的特性。

回调函数基本就是你传递给其它函数的参数。这对Javascript是可行的，因为函数是第一类对象。

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

这段代码是典型的Nodejs回调函数。在回调函数中遵循错误优先的原则，错误信息的参数可以为空，回调函数的结果作为回调函数第二个参数。我们都应该遵循这条原则，因为开发者都是假定其他的代码都是按照这种原则。

#### 现代Javascript对回调函数的改进

在现代的Javascript中，我们有了promise对象。Promises可以看作是对回调函数作异步调用的优化的接口。不再通过将回调函数的结果和回调函数可能出现的错误作为回调函数的参数。Promises对象允许代码分开处理函数成功返回结果和函数出现的错误，通过链条的方式处理多个异步回调函数，避免出现回调嵌套的地狱。

如果readFileAsArray支持promises，我们就可以写出下面的代码：
```
readFileAsArray('./numbers.txt')
  .then(lines => {
    const numbers = lines.map(Number);
    const oddNumbers = numbers.filter(n => n%2 === 1);
    console.log('Odd numbers count:', oddNumbers.length);
  })
  .catch(console.error);
 ```
通过调用.then函数获取异步函数的结果，而不是将结果放进回调函数中。 .catch函数可以获取回调函数的错误信息。

由于新Promise对象的出现，让现代的Javascript代码很方便的支持promise接口。

下面的代码就是对已经通过回调函数进行异步调用的另一种封装，通过promise对象实现readFileAsArray函数的一部调用：
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

当我们想引用异步函数中的错误信息，可以调用reject函数。当我们想调用异步函数返回的数据，可以调用resolve函数。

#### 通过async／await 调用promises对象

通过异步回调的接口可以使代码在需要异步回调时变的非常简单，但是随着回调的增多，代码也会显得很凌乱。

Promise对象对异步回调优化了一点，Generator函数在Promise对象的基础上又又优化了一点。对异步函数调用最友好的方式还要数async，通过async函数我们可以以同步编程的方式，实现异步调用。这种编程方式的出现是的异步代码在可读性方面有了质的提升。

下面的代码就是如何使用async／await调用readFileAsArray：

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