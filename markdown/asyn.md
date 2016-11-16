# JavaScript异步编程

JavaScript（引擎）是单线程（我们暂且叫它**主线程**）。本人非常认同下面的解释：

> **JavaScript引擎是单线程运行的,浏览器无论在什么时候都只且只有一个线程在运行JavaScript程序。**

JavaScript是单线程的，所有任务需要排队，前一个任务结束，才会执行后一个任务。如果前一个任务耗时长，后一个任务就不得不一直等着，就会造成阻塞问题。

但是实际上还存在其他的**浏览器线程**。例如：处理AJAX请求的线程、处理DOM事件的线程、定时器线程等等，不妨叫它们**工作线程**。

JavaScript（引擎）是单线程的，但并不代表异步操作就是在JavaScript引擎单线程中执行的。请记住，所有的异步操作都是通过浏览器新开一个线程的，如AJAX线程。不过这些线程是浏览器自身已经内置的，我们是不能通过JavaScript自定义新线程的。

那么，为什么JavaScript不能有多个线程呢？多线程不就可以解决这些问题了吗？[阮一峰](http://www.ruanyifeng.com/)是这样解释的：

> 作为浏览器脚本语言，JavaScript的主要用途是与用户互动，以及操作DOM。这决定了它只能是单线程，否则会带来很复杂的同步问题。所以，为了避免复杂性，从一诞生，JavaScript就是单线程，这已经成了这门语言的核心特征，将来也不会改变。

或许你还有疑问既然JavaScript是单线程的，怎么还可以实现类似多线程的异步操作，这个后续会详说。

## 基础理论

### 异步与同步

同步和异步关注的是**消息通信机制** ，`同步通信`可以理解为调用函数得到返回结果（当然可以返回undefined），我们就传达了消息（执行完成等消息）。`异步通信`我们就需要利用浏览器提供的`Event Loop线程`（JavaScript引擎线程与其他进程（主要是各种I/O操作）的通信）。我们自己是无法利用js创建新的异步消息通信方式的。JavaScript就是基于`事件驱动`的（就是基于浏览器的Event Loop线程）。

#### 同步

所谓同步，就是要上一个任务完成后，下一个任务才能开始。

举个人工查询话费的例子：

```js
//任务一
function firstTask() {
  //打电话问话费剩余多少
  //客服说稍等一下，帮您查一下
  //然后您等了10秒钟
  //客服说还剩2毛
  return 2;
}
//任务二
secondTask(){
  //吃饭
}

var surplus = firstTask();//第一个任务花了10秒多
secondTask()//只能等客服查话费给答复后，才能吃饭
```

#### 异步

所谓异步，就是多任务可以同时开始，完成后**主动**汇报结果。

举个人机查询话费的例子：

```js
//任务一
function firstTask(callback) {
  //打电话问话费剩余多少
  //机器客服说，相关查询会通过信息发送到您的手机
  check(callback);//异步查询话费
  //然后您挂机，去吃饭
}
//任务二
secondTask(c)
  //吃饭
  //吃饭中途收到话费信息，知道了剩余多少话费，继续吃饭
}

//直接返回
firstTask(function(num,err){
  if(!err) {
    //异步话费反馈
  }else {
    //异步反馈，查询失败
  }
});
secondTask(surplus);//不用等待，直接吃饭，途中收到话费反馈
console.log("吃完了")//也可能吃完后才收到话费反馈
```

### 浏览器线程与JavaScript线程

浏览器内核实现（不同浏览器实现方式大同小异）允许多个线程异步执行，这些线程在内核制控下相互配合以保持同步。浏览器内核的实现至少有四个常驻线程：**javascript引擎线程**、**界面渲染线程**、**浏览器事件触发线程**、**Event Loop线程**。除些以外，也有一些执行完就终止的线程：如Http请求线程等，这些异步线程都会产生不同的异步事件。

- javascript引擎线程

  就是我们平常说的JavaScript是单线程，指的就是这个线程。执行代码，执行浏览器派给他的任务（通过用户操作）。后续我们称这个为**主线程**，很多浏览器线程都是围绕这个线程运行的。

- 界面渲染线程

  该线程负责渲染浏览器界面HTML标签，当界面需要重绘(Repaint)或由于某种操作引发回流(Reflow)时，该线程就会执行。
  **渲染线程与JavaScript引擎线程是互斥的！**因为JavaScript脚本是可操纵DOM元素，在修改这些元素属性同时渲染界面，那么渲染线程前后获得的元素数据就可能不一致了。在JavaScript引擎运行脚本期间，浏览器渲染线程都是处于挂起状态的。所以,在脚本中执行对界面进行更新操作,如添加结点，删除结点或改变结点的外观等更新并不会立即体现出来,这些操作将保存在一个队列中，待JavaScript引擎空闲时才有机会渲染出来。

- 事件触发线程

  JavaScript脚本的执行（javascript引擎线程）不影响html元素事件的触发（但是读取返回结果，就需要等待JavaScript脚本这行完成）。

- Event Loop线程（事件循环线程）

  可以理解为消息线程，这个线程负责与其他进程（主要是各种I/O操作）的通信。异步就是靠他，才有用武之地。

还有就是，JavaScript被设计为`单线程`，使用JavaScript编程是是不可以**开辟新的线程**。但JavaScript是可以利用浏览器的其他线程的，如处理AJAX请求的线程、处理DOM事件的线程、定时器线程等等。

## 异步详解

### JavaScript 引擎执行期间（runtime）的一些概念

[![Stack、Heap、Queue](https://camo.githubusercontent.com/fcc76ec13a1606994c1c867ae2aaf25837ae98fc/68747470733a2f2f646576656c6f7065722e6d6f7a696c6c612e6f72672f66696c65732f343631372f64656661756c742e737667)](https://camo.githubusercontent.com/fcc76ec13a1606994c1c867ae2aaf25837ae98fc/68747470733a2f2f646576656c6f7065722e6d6f7a696c6c612e6f72672f66696c65732f343631372f64656661756c742e737667)

#### Stack（栈）

可以理解为任务**执行栈**，从上到下执行。当栈为空后（**同步任务**执行完成后 ），会读取**异步任务列队**。

#### Heap（堆）

一个用来表示内存中一大片非结构化区域的名字，对象都被分配在这。异步跟这个没有直接关系，在本文可以不理会。

#### Queue（队列）

可以简单理解为**异步任务队列**（可以简单理解为异步回调函数），只要异步任务有了运行结果，就在**异步任务队列**之后放置一个事件。需要等待同步任务（Stack）执行完毕后，主线程才会读取异步任务，进入**执行栈**。

### 异步机制

有童鞋不理解为什么JavaScript是单线程的却能执行AJAX、事件响应等类似多线程一样的操作呢？

因为JavaScript有一种异步机制（当然要浏览器支持）。

首先引用一句话：

> **浏览器中的JavaScript引擎是基于事件驱动的。**

这里的事件指的是`Event Loop`（事件循环）,Event Loop可以简单理解为JavaScript的`异步机制`，Event Loop是一种并发的模型。（所谓“并发”是指两个或两个以上的事件在同一时间间隔中发生，交替时间很短，用户察觉不到，形成了“宏观”意义上的并发操作。）

> 用户的各种I/O操作、触发事件等完成后都会通过Event Loop线程在**异步任务队列**之后放置一个事件， 主线程（javascript引擎线程）空闲时会从异步任务队列中读取事件，这个过程是不断循环的。这整个过程称为Event Loop（事件循环）。

Event Loop的一个特性是永不阻塞，理论上可以添加无数个事件到异**步任务列队**而不阻塞。但是如果有一个异步任务完成后，主线程读取并执行的时间过长会造成阻呢？

主线程执行异步任务结果造成的阻塞不属于Event Loop阻塞。简单点说，**主线程阻塞不是Event Loop阻塞！**。

### 异步操作类型

- 事件

- 定时器

  setTimeout、setInterval、setImmediate

- 异步HTTP请求

  Ajax、fetch请求等。

- I/O类操作（Node.js）

  文件读取、写入等等。

### 异步操作编写方式

JS 中最基础的异步调用方式是 callback，它将回调函数` callback` 传给异步 API，由浏览器或 Node 在异步完成后，通知 JS 引擎调用 `callback`。对于简单的异步操作，用` callback `实现，是够用的。但随着负责交互页面和 Node 出现，callback 方案的弊端开始浮现出来。` Promise `规范孕育而生，并被纳入 ES6 的规范中。为了让异步编写像同步一样，es6出了`Generator`规范，后来 ES7 又在` Generator `的基础上将`async` 函数（其实就是`Generator`的语法糖）纳入标准。

#### 异步之callback

就是像平常我们事件绑定的回调函数。

```js
var callback = function(e){
  alert(e)
}
document.addEventListener('click',callback);
```

#### 异步之Promise

简单的异步操作使用callback是没问题的，但是当callback嵌套太多，就形成了callback hell（回调地狱）。如下：

```js
function callback(){
  function callback2(){
    function callback3(){
      function callback4(){
        function callback5(){
          ....
        }
      }
    }
  }
}
```

上面代码看起来特别不好，于是就有了Promise。

Promises/A规范是Kris Zyp于2009年提出来的，它通过规范API接口来简化异步编程，使我们的异步逻辑代码更易理解。它由社区最早提出和实现，ES6将其写进了语言标准，统一了用法，原生提供了`Promise`对象。

详细请看阮一峰的[Promise对象](http://es6.ruanyifeng.com/#docs/promise)，那里说得很清楚。

#### 异步之Generator(es6)

Generator函数是ES6提供的一种异步编程解决方案，语法行为与传统函数完全不同，是一种全新的编写方式，目前浏览器还未兼容。

详细请看阮一峰的[Generator 函数](http://es6.ruanyifeng.com/#docs/generator)。

#### 异步之Async函数(es7)

ES7提供了`async`函数，使得异步操作变得更加方便。`async`函数是什么？一句话，`async`函数就是Generator函数的语法糖，不过内置执行器，不需要使用第三方执行器（或者自己写的执行器）。`async`函数更像同步操作。

## 参考文章

- [JavaScript 运行机制详解：再谈Event Loop](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)
- [JavaScript线程机制](http://blog.inching.org/2014/02/16/javascript-thread/)
- [关于JavaScript单线程的一些事](https://github.com/JChehe/blog/blob/master/posts/%E5%85%B3%E4%BA%8EJavaScript%E5%8D%95%E7%BA%BF%E7%A8%8B%E7%9A%84%E4%B8%80%E4%BA%9B%E4%BA%8B.md)
- [Promise对象](http://es6.ruanyifeng.com/#docs/promise)