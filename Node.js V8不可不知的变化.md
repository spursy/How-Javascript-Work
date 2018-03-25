
自从Node.js以V8作为Javascript的执行引擎推出以来，深受大家熟知和喜爱。V8引擎是Googe的Chrome浏览器的lJavascript虚拟。设计之初，就是要让Javascript执行的更快，至少要比现有的Javascript引擎拥有更高的效率。但是由于Javascript动态类型、弱类型、基于原型语言特性，为高效执行Javascript代码带来很多困难。本片文件就是围绕V8引擎性能优化的进化展开。

V8引擎可以高效的执行Javascript主要是因为使用即使编译器（JIT），它可以在运行时优化Javascript的性能。V8引擎启动时的速度就是FullCodegen（V8引擎的第一代内核）的两倍。后来V8引擎团队实现了Crankshaft，它可以实现很多FullCodegen不能实现的性能优化。

作为一位自90年代起就是Javascript的观察者和使用者，无论使用什么执行引擎来优化Javascript代码都是反直觉的，因为Java script代码的执行速度明显变慢的原因通常难以理解。

近些年我与[Matteo Collina](https://twitter.com/matteocollina)一直都在从事如何才能写出高效的Node.js代码探索。这就意味着我们要知道代码在V8引擎中执行的速度的快慢。

自V8引擎团队推出新一代JIT编译内核Turbofan后，我们关于Javascript性能的假设也将被彻底颠覆。

通过对V8的不同版本做了一系列的观察和各种性能测试结果的比较，我们发现那些一开始被认为是V8的性能杀手的编程方式对Turbofan不在适合。

诚然在优化V8执行引擎之前，我们应首先关注它的API设计风格、内部实现算法和数据结构。这些微小的测试标准是V8执行引擎升级后执行Javascript的性能标准。我们可以根据这些性能比较改变我们的编程风格从而优化Javascript代码的执行性能。

我们将测试V8执行引擎在以下版本（5.1、5.8、5.9、6.0、6.1）的性能测试。

不同V8版本对应的内核如下：Node 6使用的是V8的5.1版本（Crankshaft即时编译器）、Node 8.0和8.2使用的是V8的5.8版本（Crankshaft和Turbofan的混合编译器）、Node 8.3或是8.4可能对着这V8的5.9或是6.0版。在我写这篇文章时V8的最新版本是6.1，在仓库[https://github.com/nodejs/node-v8](https://github.com/nodejs/node-v8)中正在集成与Node环境的测试，这样就是说V8的6.1版就会集成在未来某个版本的Node.js中。

来吧，让我们一起看看这些性能测试，这也意味着这些性能优化将会出现未来的某个版本的Node.js中。注意：这些测试结果都是通过[benchmark.js](https://www.npmjs.com/package/benchmark)测试的，绘制的值也是美妙的执行数，因此在表格中数值越高代表性能越好。

### TRY/CATCH的之殇
对try/catch代码块的优化在社区中被广泛称赞。

我们将针对以下四个场景比较try/catch在不同V8版本和使用场景下的性能。
- 在方法函数中使用try/catch (sum with try/catch)
- 在方法函数中不实用try/catch (sum without try/catch)
- 使用try/catch包装函数 (sum wrapped)
- 方法函数内外都不使用try/catch (sum function)

Code ： [https://github.com/davidmarkclements/v8-perf/blob/master/bench/try-catch.js](https://github.com/davidmarkclements/v8-perf/blob/master/bench/try-catch.js)
![try:catch.png](https://upload-images.jianshu.io/upload_images/704770-5984a2df94ba538c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过性能比较可以看到在Node 6（V8 5.1）中使用无论函数内外使用try/catch都会带来性能的瓶颈，但是在Node 8.0-8.2（V8 5.8）中这种影响就相对变小。

同时也要注意到Node 6（V8 5.1）和Node 8.0-8.2（V8 5.8）中，在方法函数内使用try/catch要比在方法函数外使用try/catch， 带来更差的性能表现。

然而在Node 8.3版本后，在方法函数内外使用try/catch带来的性能瓶颈几乎不存在。

尽管如此，不要高兴的太早。我在和Matteo在准备一些性能研讨材料时，Turbofan在一些具体的场景中出现优化/反优化无限循环的怪圈（这将是破坏V8性能的杀手）。

### 从object中删除properties

多年来，delete的问题已经严重限制了想提高Javascript执行性能的所有开发者。

关于delete的问题大致可以归结为Javascript的动态性或是原型链的语言特性，使的在实现属性（类型）查找时变得很困难。

V8引擎技术为了快速生成对象属性，在生成对象时采用在C++层面上对象"形状"的性质。"形状"本质上就是属性的键或值（包括原型链上的键或值），它们被成为隐藏的类。如果不确定对象的"类型"，V8通过Hash表的方式查询对象的属性。Hash表这种查询方法比较慢，这被认为是对象运行时待优化的一个语言问题。过去当我们从对象的属性序列上删除属性前，会做一次Hash表的查询。这也就是避免使用delete，而是通过设置属性为undefined的原因。即使使用后一种方式，当我们在检查对象的属性是否存在时，也会有和前一种方式同样的性能问题。然后后一种方式在预序列化时性能很好，因为JSON.stringify并不把undefined放在输出的结果中（undefined并不是一种合法的JSON值）。

现在，让我们对比下新版Turbofan对这个性能问题所做的优化吧。

在接下来的性能比较中，我们将关注一下三种情况：
- 在设置对象的属性为undefined后，序列化对象（setting to undefined）
- 在通过delete删除对象的属性后，序列化对象  (delete)
- 同通过delete删除对象最近添加的属性后，序列化对象  (delete last property)

**Code:** [https://github.com/davidmarkclements/v8-perf/blob/master/bench/property-removal.js](https://github.com/davidmarkclements/v8-perf/blob/master/bench/property-removal.js)

![property.png](https://upload-images.jianshu.io/upload_images/704770-8cfc3867c43c6c35.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在V8 6.0和6.1 版本里， 通过Turbonfan优化后deleting最近添加的属性会比较高效，它的性能表现要比直接设置对象的属性为undefined要好。这是一个好消息，预示着V8团队将会持续优化delete的性能。然而如果对象的属性不是最近添加，直接删除对象的属性将会导致性能的下降。即便如此我依旧推荐在代码中通过delete删除对象的属性。

### ARGUMENTS的数组化

与箭头函数不同，普通函数存在隐式参数组（arguments），这个数组并不是真正的数组而是参数对象。

为了通过数组的方法使用隐式参数组（arguments），我们会把隐式参数组对象的属性拷贝到数组中。在过去Javascript的设计哲学是较少代码意味着优越的性能。尽管这条规则为浏览器减少了代码荷载，但是同样的代码在服务端就会带来性能的瓶颈。于是如何精炼而高效的将arguments对象转换为数组变得很流行。

Array.prototype.slice.call(arguments)：把Arguments参数对象传递给Array的slice函数，Arguments参数对象作为slice函数的this上下文，从而将参数对象转换为数组。

然而当把函数的隐式参数对象通过函数上下文暴露给外部函数使用（通过Array.slice.call(arguments)将隐式参数对象转换为数组后作为另一个函数的参数），这种方式就会导致代码性能的下降。

接下来将围绕V8四版本中，直接将隐式参数对象转换为数组和将隐式参数对象拷贝到数组的两个主题展开测试对比。

下面是测试用例：
- 直接将隐式参数对象转换为数组 （leaky arguments）
- 通过Array.prototype.slice将隐式参数对象转换为数组 （Array prototype arguments）
- 通过for循环隐式参数对象的属性并把它们赋值给数组 （for-loop copy arguments）
- 通过使用EcmaScript 2015中将对象转换为数组的新特性 （spread operator）

**Code:** [https://github.com/davidmarkclements/v8-perf/blob/master/bench/arguments.js](https://github.com/davidmarkclements/v8-perf/blob/master/bench/arguments.js)

![argument1.png](https://upload-images.jianshu.io/upload_images/704770-7d2f76283e2a020f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

让我们通过线性图对比不同代码的执行性能：

![argument2.png](https://upload-images.jianshu.io/upload_images/704770-daddcc6f06d71b98.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 柯里化和BINDING

柯里化指的是可以在嵌套闭包范围内扑捉函数的状态。

例如：
```
function add (a, b) {
  return a + b
}
const add10 = function (n) {
  return add(10, n)
}
console.log(add10(20))
```

add10函数10作为add函数的一个参数。


通过EcmaScript中的bind函数可以更简洁的实现柯里化的功能：
```
function add (a, b) {
  return a + b
}
const add10 = add.bind(null, 10)
console.log(add10(20))
```
然而我们一贯不支持使用bind函数，因为它比闭包在性能上明显要慢。

下面我们在V8不同的版本上测试bind和闭包的性能差异，下面是四个测试用例：
- 在一个函数中通过第一个参数柯里化调用另一个函数 （curry）
- 在箭头函数中通过第一个参数柯里化调用另一个函数 （fat array curry）
- 通过bind将第一个参数传递给函数，实现函数调用  （bind）
- 直接调用函数

**Code:** [https://github.com/davidmarkclements/v8-perf/blob/master/bench/currying.js](https://github.com/davidmarkclements/v8-perf/blob/master/bench/currying.js)

![currying.png](https://upload-images.jianshu.io/upload_images/704770-619f22a51e233d48.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这个测试结果的线性图非常清晰的表明这些方法在V8的最近几个版本的性能表现。箭头函数在实现柯里化传参时的性能要比普通函数优越（至少通过线性表可以明显看出性能差异）。在V8 5.1（Node 6）和5.8（Node 8.0-8.2）中通过bind实现柯里化的性能相对比较差，明显可以看出箭头函数是一个不错的选择。然而V8的5.9版（Node.js 8.3)bind的速度达到历史最优，即便与未来V8 6.1版的性能差异也几乎可以忽略不计。

对比实现的柯里化的不同方式，箭头函数是最优的选择方案。尽管剪头函数实现柯里化的性能比普通函数实现相同功能的性能更优，预测在未来V8版本中箭头函数的性能将会与普通函数的性能相差无几，这样箭头函数的性能几乎没有优势。  预警：为了获取更全面的测试数据，我们应该对更多维的数据做关于不同方式实现柯里化的性能测试。

### 字节码的计算

函数的大小（包括函数签名、空白行以及注释）是否会影响它与V8引擎的内联？是的，为函数添加注释可能会带来大约百分之十的性能下降。这个issue有没有在Turbofan编译器中得到改观，让我们一起看看吧。

我们将通过下面四个测试用例展开性能间的测试对比：

-  在函数中调用另一个函数（sum small function）
- 在一个函数中添加大量的注释 （long all together）
- 在函数中调用另一个函数，被调用函数中添加大量注释 （sum long function）

**Code:** [https://github.com/davidmarkclements/v8-perf/blob/master/bench/function-size.js](https://github.com/davidmarkclements/v8-perf/blob/master/bench/function-size.js)

![chracter_count.png](https://upload-images.jianshu.io/upload_images/704770-6b2a63a88dd28adf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在V8 5.1（Node 6）中，sum small function与long all together的性能是一样的。这向我们展示V8引擎是如何解析函数调用的？当在函数中调用一个另一个函数时，V8就像是把被调用的函数写在调用函数中。即便我们在函数中为被调用函数写了大量的注释，这些注释好像已经内嵌在函数中，因此sum smal function与long all together的函数性能是一样的。  然而在V8 5.1（Node 6）中，在函数中调用一个有大量注释的函数时，性能就会有明显的下降。

在V8 5.8（Node 8.0-8.2中），除了在函数中调用另一个函数的性能产生微小的下降，其它的性能表现几乎与V8 5.1一致。这可能是由于函数执行时V8引擎要在Crankshaft和Turbofan两个编译器间切换，造成的性能的损耗（例如当调用函数在Crankshaft中，被调用函数在Turbofan中，这样在编译这段代码时就会在两个编译器间切换）。

在V8 5.9及其以上版本（Node 8.3+），无论是在函数中或是在被调用函数中添加大量注释、空格，代码的性能表现几乎是一样的。这是由于在自V8 5.9后，引擎的编译器只有Turbofan。Turbofan计算所要执行的函数的大小时，使用的是AST（抽象语法树）。抽象语法树通过函数时实际指令的数目计算函数的大小，而不像Crankshaft编译器是通过函数中字节（包括注释，函数签名，空白行等）的数目计算函数的大小。

值得注意的是，采用采用新计算函数的大小的算法后，关于这三测试用例的性能下降了。

另外，我们仍然需要保证函数是精练的，以及在函数中避免使用大量的注释或空格（行）。如果你想要更高的执行性能，最好还是避免使用调用函数，选择把被调用函数写在函数体中。当然我们的优化也要与以下规则保持平衡，当函数实际执行的代码的数量达到一定量后，执行性能也会有瓶颈。因此把被调用函数的代码拷贝到调用函数中，也可能会造成性能下降。总结下，手动内联代码只是一把隐形的枪，在实际编程中最好使用编译器内联代码。

### 32位与64位的数值类型

众所周知Javascript只有一种数值类型:number。

然而，V8引擎是由C++实现的，所以必须对数字值的Javascript的底层上进行是32字节还是64字节的选择。

当引擎断定一个JS数值类型的量不带小数时，就会断定改值是32字节的数值类型，如果验证不是，则更改为64位字节的数值类型。这看起来是一种很不错的机制，因为在很多测试用例下，数值的大小都是在-2147483648到2147483647之间。如果Javascript的数值类型超过这个区间，JIT编译器就会设置改数值的底层是浮点数值类型。这种动态设置数值底层类型的机制可能会影响编译器的性能。

接下来的性能分析将会通过下面三个测试用例：

- 函数内的数值计算在32位字节的范围内 （sum small）
- 函数内的数值计算开始在32位字节内与后来在64位字节内 （from samll to big）
- 函数内的数值计算在64位的浮点类型范围内  （all big）

**Code:** [https://github.com/davidmarkclements/v8-perf/blob/master/bench/numbers.js](https://github.com/davidmarkclements/v8-perf/blob/master/bench/numbers.js)

![number.png](https://upload-images.jianshu.io/upload_images/704770-b4de8e96b47cd251.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过图表我们可以看出无论是在Node 6（V8 5.1）、Node 8(V8 5.8) 还是未来更高的版本中，在处理64位的数值时编译器的性能只有处理32位字节的1/2或是2/3。因此，如果你将要使用非常长的数值类型，先把它们转换为字符串类型。

我们也可以明显的看出在Node 6（V8 5.1）和Node 8.1（v8 5.8）版本中，执行32位的数值类型的性能比较高，但是在Node 8.3 +（V8 5.9 +）中性能就明显下降。然而在Node 8.3+（V8 5.9+）中处理64位的浮点数的性能显著增强。这种变化好像是由于新的编译器在处理32位数值自身速度的下降导致的，而不是由于在测试函数中调用其它函数或是由于测试代码中的for循环导致的。

### 遍历对象
我们有很多种方法实现便利对象的属性值，并对属性值做一些处理。让我看看在V8引擎的不同版本中哪种方式的性能是最优的。

下面将是我们用作测试性能的测试用例：
- 使用for-in循环对象的属性，然后通过hasOwnProperty判断属性是否存在对象中，最后获取该属性在对象中对应的值 （for in）
- 使用数组的reduce函数遍历键值（Object.keys）数组，通过遍历函数获取对象的属性值 （Object.keys functional）
- 使用数组的reduce函数遍历键值（Object.keys）数组，通过遍历函数（箭头函数）获取对象的属性值 （Object.keys functional with array）
-  使用for循环对象的键值数组（Object.keys），在循环中通过键获取对应的对象属性值 （Object.keys with for loop）

我们也可以对V8 5.8， 5.9，6.0和6.1做一些额外的性能测试：

- 使用数组的reduce函数遍历对象属性值（Object.values）数组（Object.values functional）
- 使用数组的reduce函数遍历对象属性值（Object.values）数组，通过遍历函数（箭头函数）获取对象的属性值 （Object.values functional with array）
- 使用for-in循环对象的属性值数组 （Object.values with for loop）

*我们不能对V8 5.1 （Node 6）做关于Object.values的性能测试，因为这个版本的V8还不支持这个EcmaScript2017的新特性。*

![iterator.png](https://upload-images.jianshu.io/upload_images/704770-27e97546903d7a90.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Node （V8 5.1）和Node 8.0-8.2 （V8 5.8）中，通过for-in遍历数组的键值数组然后获取对象的属性值是最快的。它几乎达到4千万每分钟的处理速度，这样的效率几乎是第二名的8百万每分钟处理速度的5倍。

在 V8 6.0（Node 8.3）中， 相比较上一代V8引擎for-in的处理速度几乎下降到四分之一，但仍然是所有遍历数组对象的方法中最快的一个。

在V8 6.1中， Objects.keys处理速度向前迈进，一度超过for - in成为性能最高的方法。

Turbofan背后的驱动原理优化直观的编程行为，也就是说，针对开发者最符合人体工程力学的用例进行优化。

使用对象的属性数组直接获取对象的属性要比使用对象的键值数组，然后通过键值获取属性值的方法要慢。最重要的，通过循环仍然要比函数式迭代的速度要快。因此当我们在实际开发中需要遍历属性值时，我们要多做一些权衡工作。

此外，对于那些因为其性能优势而使用for-in的人来说，当我们失去大量速度而没有其他替代方法时，这将是一个痛苦的时刻。




























