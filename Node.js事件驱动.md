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