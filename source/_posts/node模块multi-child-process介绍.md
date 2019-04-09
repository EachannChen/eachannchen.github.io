---
title: node模块multi-child-process介绍
date: 2019-04-08 15:27:12
tags:
     - node
     - npm
     - multi-child-process
---

这一篇文章主要是介绍下我最近做的一个npm包。附[github](https://github.com/EachannChen/multi-child-process)链接。
### 背景介绍
如果需要想了解如何使用包，可以直接跳到[**使用介绍** ](#usage-pane)。

先介绍下这个npm包是在怎样的背景下产生的。前段时间需要部署一个客户端，接受服务端的信号，然后批量注册人脸样本。而这个需要先下载每个人的人脸图片，然后调用第三方API对该人进行个人信息（包括人脸）的CRUD。800个人，需要1多小时。觉得效率太低了，萌生做个优化的想法。
<!-- more -->
#### 模块的构思

以下是我当时做这个模块的思路：

1.首先要抽离业务场景，充分利用cpu，避免因为node单线程造成的局限性。

2.如何友好地对外暴露接口，让使用者更关注业务逻辑，并且兼容node的异步调用逻辑。

3.如何维护进程池，并把进程池的技术逻辑和使用者的业务代码做一个分离，保证在业务代码处理完后，进程池里的进程可以复用。

#### 解决方案
针对以上提的3点，我后来的解决方案是这样的：

1.既然单进程有局限，那就充分利用cpu，启用多进程。这个npm包的进程池的默认进程数是当前服务器的cup个数，原理是使用child_process的fork()，衍生出新的 Node.js 进程。通过两者之间建立的 IPC 通信通道，父进程发送需要运行的Module的function名字，function所在模块的绝对路径，以及function所需的参数。子进程运行完毕后，通过通信通道返回运行结果给父进程。父进程通过回调函数输出error和result。

2.友好的暴露对外接口，就是需要封装子进程的fork，以及返回子进程运行结果。这里抓住node异步回调的特点。让使用者只需把需要运行的funciton通过module.exports暴露出来，就像日常对外暴露的router处理函数或者工具函数一样，传入函数名，绝对路径和所需的参数即可。

3.进程的创建和销毁本身就是一个系统资源的损耗。所以，如果可以维护一个进程池，不断的接受父进程的通信，处理一些费时的任务，这对快速处理任务是极好的。对那些进程间通信较少，对于阻塞性操作，并且相对独立的运行任务，进程池是个相对不错的方案。ChildProcess 类的实例都是 EventEmitter。所以，为每个 ChildProcess 类的实例注册了两类监听器，一类是通过.on()注册对'exit'，'error'，'messgae'的ChildProcess的进程池维护操作。另一类是通过.once()注册对'exit'，'error'，'messgae'的业务function的操作。这样就完成了对进程池的业务部分和技术逻辑，做了一个分离，保证了进程池里的进程在业务代码处理完后可以复用。

啰嗦了这么多，下面用过代码实例介绍如何使用该npm包。


### <a name="usage-pane"></a>使用介绍
#### 安装
可以通过 npm 命令安装.

``` bash
$ npm install multi-child-process
```
#### 测试
``` bash
$ npm test
```
#### 使用
首先require模块

```js
var pool = require('multi-child-process');
```
在使用childprocess之前，需要先初始化进程池：

```js
var procPool = new pool.initPool(3);//default: the number of cpus
```
调用`pool.initPool()`，默认产生的子进程数是服务器的cpu个数，你可以根据自己需要传入一个正整数。但是上限是10。如果你确实需要更大的进程数，你可以修改源码。


接下里就可以初始化子进程了，以下参数必须传递：

```js
pool.initChildProc(workPath, jobname, jobArgObject, cb);
```
- `workPath`(string) : 模块的所在文件的绝对路径，比如 `__dirname+'/logic.js'`
- `jobname`(string)  : 在`workPath`路径下的模块exports的函数名字，比如 'module.exports.jobname=jobname'
- `jobArgObject`(object): 对应函数所需的参数, 比如 {key1:value1,key2:value2}
- `cb`(function)  : 子进程运行完后的回调函数

```js
//./main.js
pool.initChildProc(__dirname+'/logic.js', jobname, jobArgObject, function(err,ret){
//err: error of child process or job
//ret: result of job function callback
});

//./logic.js
function jobname(jobArgObject,callback){
//callback(error,ret)
}
module.exports.jobname=jobname;
```
以下是对外API

如果进程池已经完成初始化，函数 `pool.isInited()` 会返回去true。
```js
pool.isInited();
```

如果所有的子进程都没在运行任务，则事件 '`isAllAvail`'会被触发。
```js
procPool.on('isAllAvail',cb);
```

如果所有的子进程都没在运行任务, 函数 `pool.isAllAvail()` 会返回true。
```js
pool.isAllAvail();
```

返回在运行任务的子进程数目。
```js
pool.actiProcNum();
```

返回子进程池的大小。
```js
pool.totalProcNum();
```

如果在完成所有任务后，确定进程池不再需要的话，可以调用 `pool.closePool(cb)` 函数。但是建议在监听到事件'`isAllAvail`'被触发后，再调用`pool.closePool(cb)`。参数`cb` 是关闭进程池后的回调函数。
```js
pool.closePool(cb);
```




### 注意点
对exports的function有一点特殊要求，要求function本身是能独立运行完的node服务，这点和以往的router处理函数或者工具函数稍微有点不同。这也是因为fork()本身会衍生一个新的 Node.js 进程。

### 效果对比
对原来的业务场景，我做了使用该模块前后的耗时对比。
场景：操作66张人脸图片的批量注册。

使用前：总共耗时6分钟20秒，平均每人费时5.75秒。
使用4个进程（默认值）：总共耗时4分钟28秒，平均每人费时4.06秒。
使用8个进程：总共耗时4分钟24秒，平均每人费时4.0秒。

如果大家在使用的过程中碰到问题，或者需要改进的，欢迎提[issue](https://github.com/EachannChen/multi-child-process/issues)