## javascript异步

2018年5月23日



### 1 使用异步的场合

js是单线程的语言→在客户端使用时可能造成卡顿等结果→解决方法：异步。

常用的场合：

- 网络请求，如`ajax` `http.get`

- IO 操作，如`readFile` `readdir`

- 定时函数，如`setTimeout` `setInterval`

  ​

### 2 实现异步的核心原理

**实现异步的最核心原理，就是将callback作为参数传递给异步执行函数，当有结果返回之后再触发 callback执行**。

（1）所有同步任务都在主线程上执行，形成一个执行栈（execution context stack）。

（2）主线程之外，还存在一个"任务队列"（task queue）。只要异步任务有了运行结果，就在"任务队列"之中放置一个事件。

（3）一旦"执行栈"中的所有同步任务执行完毕，系统就会读取"任务队列"，看看里面有哪些事件。那些对应的异步任务，于是结束等待状态，进入执行栈，开始执行。

（4）主线程不断重复上面的第三步。

1个例子：

```
setTimeout(console.log, 0, 'a')
console.log('b')
console.log('c')
```



### 3 常见的异步操作

1. 回调callback

   1个ajax的例子。

   ```
   var ajax = $.ajax({
       url: 'data.json',
       success: function () {
           console.log('success')
       },
       error: function () {
           console.log('error')
       }
   })

   console.log(ajax) // 返回一个 XHR 对象
   ```

   好处是写起来比较无脑，坏处是读起来反人类。并且高度耦合。

2. 事件监听

   ```
     f1.on('done', f2);
     function f1(){
     setTimeout(function () {　　　　　　
      	...
      　　　　　f1.trigger('done');
      　　　　}, 1000);
      　　}
   ```

   可以绑定多个事件，每个事件可以指定多个回调函数。

   ​

3. 发布/订阅

   下面的例子来自Ben Alman的[Tiny Pub/Sub](https://gist.github.com/661855)，这是jQuery的一个插件。

   ```
   jQuery.subscribe("done", f2);

   ...

   　　function f1(){

   　　　　setTimeout(function () {

   　　　　　　// f1的任务代码

   　　　　　　jQuery.publish("done");

   　　　　}, 1000);

   　　}

   ...

   　　jQuery.unsubscribe("done", f2);//取消订阅
   ```

   其实还是比较类似！

   ​

4. promise

   battle赢家，在es6中被引入。

   一般的实现：

   ```js
   // 创建一个 Promise 实例（异步接口和 Promise 签订协议）
   var promise = new Promise(function (resolve,reject) {
     ajax('url',resolve,reject);
   });

   // 调用实例的 then catch 方法 （成功回调、异常回调与 Promise 签订协议）
   promise.then(function(value) {
     // success
   }).catch(function (error) {
     // error
   })
   ```

   jquery的场合

   ```
   function f1(){

   　　　　var dfd = $.Deferred();

   　　　　setTimeout(function () {

   　　　　　　// f1的任务代码

   　　　　　　dfd.resolve();

   　　　　}, 500);

   　　　　return dfd.promise;
   	//每一个异步任务返回一个Promise对象
   　　}
   　　
   f1().then(f2).fail(f3);
   ```

   axios也是基于promise的。

   ​

5. generator

   generator（生成器）是ES6标准引入的新的数据类型。一个generator看上去像一个函数，但可以返回多次。

   Generator 函数可以暂停执行和恢复执行.

   语法:  * yield

   ```
   function* gen(x){
     var y = yield x + 2;
     return y;
   }
   var g = gen(1);
   g.next() // { value: 3, done: false }
   g.next() // { value: undefined, done: true }
   ```

   co函数库- Generator 函数的自动执行。

   ```
   var fs = require('fs');

   var readFile = function (fileName){
     return new Promise(function (resolve, reject){
       fs.readFile(fileName, function(error, data){
         if (error) reject(error);
         resolve(data);
       });
     });
   };

   var gen = function* (){
     var f1 = yield readFile('/etc/fstab');
     var f2 = yield readFile('/etc/shells');
     console.log(f1.toString());
     console.log(f2.toString());
   };

   function run(gen){
     var g = gen();
     function next(data){
       var result = g.next(data);
       if (result.done) return result.value;
       result.value.then(function(data){
         next(data);
       });
     }
     next();
   }
   run(gen);

   //就是说上面的run直接给你封装好了
   var co = require('co');
   co(gen);

   //也支持then
   co(gen).then(function (){
     console.log('Generator 函数执行完成');
   })
   ```

   ​

   ​

6. async

   ```
   var fs = require('fs');

   var readFile = function (fileName){
     return new Promise(function (resolve, reject){
       fs.readFile(fileName, function(error, data){
         if (error) reject(error);
         resolve(data);
       });
     });
   };

   var gen = function* (){
     var f1 = yield readFile('/etc/fstab');
     var f2 = yield readFile('/etc/shells');
     console.log(f1.toString());
     console.log(f2.toString());
   };

   //async
   var asyncReadFile = async function (){
     var f1 = await readFile('/etc/fstab');
     var f2 = await readFile('/etc/shells');
     console.log(f1.toString());
     console.log(f2.toString());
   };
   ```

   ```
   function timeout(ms) {
     return new Promise((resolve) => {
       setTimeout(resolve, ms);
     });
   }

   async function asyncPrint(value, ms) {
     await timeout(ms);
     console.log(value)
     return 11;
   }

    asyncPrint('hello world', 50);
   ```

   比较generator:

   1. 内置执行器

   2. 更好的语义

   3. 更广的适用性

      co 函数库约定，yield 命令后面只能是 Thunk 函数或 Promise 对象

      ↓↓↓

      async 函数的 await 命令后面，可以跟 Promise 对象和原始类型的值

      ​

   * await 命令后面的 Promise 对象，运行结果可能是 rejected，所以最好把 await 命令放在 try...catch 代码块中。

   ```
   async function myFunction() {
     try {
       await somethingThatReturnsAPromise();
     } catch (err) {
       console.log(err);
     }
   }
   ```

   ​



### 参考

https://div.io/topic/1802

https://www.cnblogs.com/wangfupeng1988/p/6513070.html

http://fex.baidu.com/blog/2015/07/we-have-a-problem-with-promises/?qq-pf-to=pcqq.c2c 关于promise