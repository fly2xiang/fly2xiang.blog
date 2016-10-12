title: "Node.js 常见面试题"
date: 2015-07-01 11:05:00
tags:
- Node.js
- 面试题
- 答案
categories: 
- NodeJS

---
## 前言

打开这篇文章时，你要么想要聘请Node.js开发者，要么想要应聘Node.js开发者。下面将给出你一些要问的问题和一些你应该知道的问题答案。

在你看问题之前，现要明确三点：

首先，这些问题都是皮毛，问一些像这样的问题并不能评判一个人，但是它可以使你了解一个人的在Node.js方面的经验。这种类型的问题并不能告诉你面试者的思维方式和工作习惯。

其次，显示生活中的问题更能够展示应聘者的知识——我们喜欢和员工进行结对编程。

第三，最重要的，我们都是人，让您的招聘过程越受欢迎越好。

## 有用的Node.js面试题

- 什么是 error-first callback ？
- 如何避免回调函数嵌套？
- Node程序如何监听80端口？
- 什么是事件循环（event loop）？
- 使用什么工具检查代码风格？
- 操作错误（operational errors）和程序错误（programmer errors）的区别是什么？
- 为什么 npm shrinkwarp 非常有用？
- 什么是stub？说出它的用途？
- 什么是测试金字塔？在做HTTP API的时候要怎么实现？
- 你最熟悉的Node框架是什么？为什么？

#### 现在我们来看看问题的答案？

### 什么是 error-first callback ？

error-first callback 用来传递错误和数据。第一个参数永远是一个错误对象（error-object），回调函数必须检查它。余下的参数用不过来传递数据。

```javascript
fs.readFile(filePath, function(err, data) {  
  if (err) {
    //处理出现错误的情况
  }
  //处理数据
});
```

_这个问题有什么帮助？_

这个问题可以考察面试者对于Node异步操作基本知识的见解。

### 如何避免回调函数嵌套？

模块化：将回调写成单独的函数

使用 _Promises_

使用 `yield` 和 _Generators_ 和/或 _Promises_

_这个问题有什么帮助？_

这个问题的答案可能有大的差异，这取决面试者是否又去了解Node最新的发展，包括ES6，ES7和最新的流程控制库。

### Node程序如何监听80端口？

脑筋急转弯！你不应该直接使用Node监听80端口（在*nix系统中），这样做需要root权限，对于运行程序来说这不是一个好主意。

不过，你可以使Node监听1024以上的端口，然后在Node前面部署nginx反向代理。

_这个问题有什么帮助？_

这个问题帮你考察面试者是否有Node应用经验。

### 什么是事件循环（event loop）？

至少从开发者的角度来看，Node.js 是单线程运行的。底层使用libuv使用多线程。

每一个I/O操作都需要一个回调，一旦操作完成会被事件循环执行。更详细的解释可以观看下面的视频：

<iframe width="560" height="315" src="https://www.youtube.com/embed/8aGhZQkoFbQ" frameborder="0" allowfullscreen></iframe>

_这个问题有什么帮助？_

这个问题可以看出面试者在Node上的底层知识的深度，如果他知道libuv的话。

- 使用什么工具检查代码风格？

你很多工具可以选择：

- JSLint by Douglas Crockford
- JSHint
- ESLint
- JSCS

开发团队项目时，强制指定代码风格和使用静态分析，捕捉常见的错误，这些工具都非常有用。

_这个问题有什么帮助？_

如果你们谈到了开发大型javascript应用程序，这些他应该知道。

### 操作错误（operational errors）和程序错误（programmer errors）的区别是什么？

操作错误不是bug，是系统的问题，例如超时或者硬件故障。

另一方面，程序错误（programmer errors）是实际的错误。

_这个问题有什么帮助？_

这个问题与Node相关，你可以得到关于面试者技术水平的有用信息。

### 

### 为什么 npm shrinkwarp 非常有用？

This command locks down the versions of a package's dependencies so that you can control exactly which versions of each dependency will be used when your package is installed. - npmjs.com

这个命令在部署Node.js应用时是非常有用的——它可以保证所部属的版本就是依赖的版本。

_这个问题有什么帮助？_

这个问题可以让你对面试者的npm cli和Node.js操作最佳实践知识有更深入的了解。

### 什么是stub？说出它的用途？

Stubs是模拟模块或组件行为的程序。
Stubs提供已知的答案来调用函数，另外你还可以断言哪个stubs被调用。

```javascript
var fs = require('fs');

var readFileStub = sinon.stub(fs, 'readFile', function (path, cb) {  
  return cb(null, 'filecontent');
});

expect(readFileStub).to.be.called;  
readFileStub.restore();  
```

_这个问题有什么帮助？_

这个问题考察面试者的测试知识，如果他不知道什么是Stubs，你可以问他是如何进行单元测试的。

### 什么是测试金字塔？在做HTTP API的时候要怎么实现？

测试金字塔意思是在写测试时应该编写的底层但愿测试要多于高级的端到端测试。

对于HTTP APIs，应该归结为：

- 对你的模型多很多单元测试
- 在你的模型与其他交互时更少的集成测试
- 更少的验收测试，在HTTP端

_这个问题有什么帮助？_

这个问题告诉你面试者在测试上有多少经验，特别是他能在每一个级别给出建议。

### 你最熟悉的Node框架是什么？为什么？

这个问题没有标准的答案，考察面试者对他所使用的框架的了解深度，包括使用的原因、利弊。



参考链接：

><http://blog.risingstack.com/node-js-interview-questions/>






