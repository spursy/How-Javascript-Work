>文章翻译自[How to Mock an Express session](https://medium.com/the-node-js-collection/how-to-mock-an-express-session-fe62baf5a611)

你是否正在使用Express框架？你是不是正在做服务端的集成测试（end-to-end）？这里我将讨论在项目中遇到的问题。这篇文章涵盖了我在使用Express框架做集成测试时所遇到的挑战，以及我是如何寻找技术解决方案的。

当你也遇到相同的问题时，希望这篇文章对你有所帮助。

### 背景
我做的项目是先要向amazon s3批量上传图片文件，然后对这些文件做一些裁剪、压缩和编辑的任务。

使用Express中间件做认证、授权以及设置上下文的工作都很顺利。我使用[cookie-session](https://www.npmjs.com/package/cookie-session)作为控制session的中间件，这就意味着需要[cookie-session](https://www.npmjs.com/package/cookie-session)负责相应session数据的加密、配置和解密。

我知道许多开发者使用[sinon for unit testing](http://sinonjs.org/)。如果你对它不熟悉，我先简短介绍下它：[sinon for unit testing](http://sinonjs.org/)是通过Javascript的测试框架（如[mocha](https://mochajs.org/)）来实现mock的方法。但是问题来了，如果要实现Express Session的mock测试？在[Express](https://expressjs.com/)中有没有实现mock session的解决方案？

![Test Expressing.jpeg](https://upload-images.jianshu.io/upload_images/704770-f7555daea88c41b4.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 挑战

下面我将讲述我所遇到的挑战。我有两个路由，一个是向s3上传图片的、另一个是对上传图片做一些额外处理（如压缩、编辑）。对于第一个路由我先设置请求的session，然后再跳转到第二个路由。

1）api/image/create

```
exports.create = function(req, res) {
  // upload image to s3
  var photos = PhotosTable.create(req.files.names); // add entries in database
  req.session.count = photos.length; // set count in req.session
  res.redirect(`api/image/submit`);
}
```
2) api/image/submit (need to add a test case for this one)
```
exports.submit = function(req, res) {
  if (req.session.count && req.session.count > 0) { // get count from req.session
    // perform further tasks
    res.sendStatus(200); 
  } else {
    res.sendStatus(400); 
  }
}
```

我想mock第二个路由（api/image/submit）。然而问题是：（pi/image/submit）这个路由需要先核查req.session.count，然后在继续后面的操作。如果count是空值，回应400。使用Mocha框架你只能mock第一个路由，不能实现下面的功能：第一个路由跳转到第二个路由，然后程序从第二个路由向第一个路由的返回值，然后验证第二个路由返回值的测试用例。

##### 这个问题看起来很简单，但是实现起来遇到很多挑战

最简单的解决方案是在测试用例中，通过req params直接请求第二个路由，然后检查params是否存在，如果存在就取消检查req.session.count。

然而，这种解决方案并不是最好的。因为
- 这不是一种普通的方式，因此这种方法不能被重用
- 在代码中添加额外的if/else来控制req.params，就会需要更多的代码来确保原来的逻辑不受影响

我们也可以放弃使用两个个路由，把所有的代码都写在一个路由中。但是我们真的要这么做吗？据我所知，**不应该为了测试用例更改代码（one should never change implementatin for the sake of test cases）**

这个问题看起来很普通，但是我在网上（至少是在StackOverFlow）上并没有找到解决方案。起初我猜测在一些模块中已经解决了这个问题，诸如mocha， 或是[cookie-seeion](https://www.npmjs.com/package/cookie-session)等经常在Express中使用的模块。

**然而，并没有。**

### 我最终是如何解决的
在程序中我是使用cookie-session来维护Express与session间的管理工作。cookie-session通过Keygrip和Crypto来设置和验证session的签名。

在设置session cookies时需要name和key：
name被用来作为cookie的名字。
key是作为Keygrip对cookie值进行签名的概要值。

我在测试用例请求第二个路由时，设置req.session.count的值。下面就是代码：
```
// name = "my-session" ->  base64 value of required object (cookie)
// name.sig = "my-session.sig" -> signed value of cookie using Keygrip

let cookie = Buffer.from(JSON.stringify({"count":2})).toString('base64'); // base64 converted value of cookie

let kg = Keygrip(['testKey']) // same key as I'm using in my app
let hash = kg.sign('my-session=' + cookie);

await request(app).get('api/image/submit').set("Accept", "text/html")
    .set('cookie', ['my-session=' + cookie + '; ' + 'my-session.sig=' + hash + ';'])
    .expect(200);
 ```
这仅仅是一条mock session的测试用例。在大多数情况下，测试用例需要定制所要测试的参数值，但这个代码可以设置任何你想测试的参数。

我已经创建一个Nodejs.js包并发不到npm中，通过这个包[mock-session](https://www.npmjs.com/package/mock-session),你可以很容易做mock session的测试。













