# Node.js的异步编程
最近参加了公司组织的Node.js培训，一周的培训过后，回想Node.js与其他编程语言最大的不(nan)同(dian)，或者说最大的优势就是天然支持异步并发机制，这也是Node.js一直被吹捧的优势。而且Node.js不需要编程人员去关心当前任务交由哪个线程处理，无需指定线程池，考虑线程安全等其他编程语言需要注意的问题，因为Node.js是单线程模式(Single Threaded)，所执行代码默认都在单一线程内执行，而并发则交由异步和回调函数后台(Event Loop)处理，编程人员只需专注于代码逻辑上的开发。

这种机制下，当使用Node.js处理请求时，如果有读取文件或者访问数据库等操作，当前线程不必等待整个操作处理完成，而可以将这些开销较大的操作交由后台处理，当前线程可以继续执行之后的任务，处理后续请求。
## 异步编程示例
举一个简单的异步编程的例子。
```js
console.log("node.js");

setTimeout(function () {
	console.log("hello");
}, 1000);

console.log("world");
```
有过Javascript编程经验的朋友对`setTimeout`一定不陌生。`setTimeout(callback, milliseconds)`是Javascript中，内置的计时函数，用法是，在`N`毫秒之后，执行某个回调`callback`函数。以上代码运行结果如下：
```
node.js
world
hello
[Finished in 1.1s]
```
虽然`console.log("world");`写在`setTimeout`之后，但这并不意味着需要等待`setTimeout`执行完成，`console.log("hello");`处于`callback`函数中，回调函数的意思是，待`setTimeout`延时1000ms结束后，再回到当前线程执行。所以输出结果中，`hello`是最后输出的一行。
## 同步与异步的区别
在继续解释异步机制之前，有必要对比一下异步编程和通常我们理解的同步编程的区别。在Node.js官方文档中，将同步和异步对应为`Blocking`和`Non-Blocking`模式，`Blocking`的意思是当有类似I/O读写文件这种非Javascript操作时，等待操作执行完毕再执行后续任务，而`Non-Blocking`则不等待，先执行后续代码，设定一个callback handler来处理这些操作返回的结果。

以读文件并输出文件内容为例，`Blocking`模式的代码如下：
```js
const fs = require('fs');
const data = fs.readFileSync('/file.md'); // blocks here until file is read
console.log(data);
moreWork(); // will run after console.log
```
以上代码顺序执行，先读取文件内容，输出，再执行接下来的`moreWork()`。同样的逻辑，`Non-Blocking`模式代码如下：
```js
const fs = require('fs');
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
  console.log(data);
});
moreWork(); // will run before console.log
```
在异步模式下，无需等待文件读取完成，即可执行后续代码。这种`Non-Blocking`模式也是增加Node.js服务吞吐量的关键。
## Event Loop
更进一步，Node.js是如何在“单线程”模式下支持高并发的呢？秘密就在于Event Loop模块，这是由C++编写的`libuv`多平台异步编程库支持的，负责监控异步事件，调度线程池，返回异步任务执行结果到主线程。
![Event Loop](https://miro.medium.com/max/2000/1*evOcy9n3vslkDt0Mj8mBYw.jpeg)

我们可以根据上图理解Event Loop的工作原理：
- 对于Node.js的application，处理请求，或者执行代码时，可以通过异步函数接口，例如上面提到的`setTimeout`和`fs.readFile`将需要异步执行的事件，添加到Event Queue，事件队列中。
- Node.js的服务启动时，会有一个单独的线程启动Event Loop模块。
- Event Loop会一直监听事件队列中的任务，按照FIFO的顺序，依次出列，交由内部的C++线程池处理。
- 处理完成之后，Event Loop会收到对应任务的响应，再返回给Node.js对应的线程中，执行回调callback函数。

Event Loop就像一个事件分发器，对于提交到事件队列的请求，依次分发到线程池中，交由后台线程处理，并负责将处理结果返回给对应的callback做后续处理。和通常的Web应用中的Router模块或者URL dispatcher，由异曲同工之妙。
## 异常处理
在异步编程中，对于异常的处理和同步相比也有所不同，还是以上述读取文件为例。

对于`Blocking`模式异常处理，类似Java，可以这样处理：
```js
const fs = require('fs');
try{
  const data = fs.readFileSync('/file.md'); // blocks here until file is read
  console.log(data);
}catch(err){
  throw new Error(err.message);
}
moreWork(); // will run after console.log
```
如果异步模式也使用`try catch`这种常规操作，代码如下：
```js
const fs = require('fs');
try{
  fs.readFile('/file.md', (err, data) => {
    console.log(data);
  });
}catch(err){
  throw new Error(err.message);
}
moreWork(); // will run before console.log
```
显而易见，根据异步执行顺序，主线程并不会等到文件读取完毕就会执行`try catch`，事实上是捕捉不到任何异常的。那么callback执行时，是怎么检查当前操作有没有异常呢？细心的你应该发现，`callback`函数的一个参数就将`err`返回了，通常我们在`callback`中检查`err`是否为空，就能捕捉异常了。
```js
fs.readFile('/file.md', (err, data) => {
  if (err) throw err;
  console.log(data);
});
```

## Promise

看到这里，也许朋友们会有个疑问，如果有N个异步操作，彼此有依赖，前一个操作必须成功，才能执行后面的操作，也就是要按顺序执行，代码应该怎么写呢？假设我们有A，B，C三个异步操作需要执行，A成功才能执行B，同样B成功才能执行C。代码大致如下：
```js
function main(callback) {
    // Do something.
    asyncA(function (err, data) {
        if (err) {
            callback(err);
        } else {
            // Do something
            asyncB(function (err, data) {
                if (err) {
                    callback(err);
                } else {
                    // Do something
                    asyncC(function (err, data) {
                        if (err) {
                            callback(err);
                        } else {
                            // Do something
                            callback(null);
                        }
                    });
                }
            });
        }
    });
}

main(function (err) {
    if (err) {
        // Deal with exception.
    }
});
```
光只有3个操作，代码的可读性就已经很差了，还要在每个`callback`中处理异常。这时，Node.js的一个缺点就体现出来了，逻辑复杂的操作，会影响代码可读性，维护代码的成本也会响应增加。为了解决这个问题，Javascritp从ES6开始支持`Promise`接口。

`Promise`是一个异步函数的接口，接受两个函数`resolve`和`reject`来构造一个这样的接口。`resolve`是当异步任务执行成功时，会执行`resolve`并返回异步执行结果，对应的`reject`表示执行失败时，如何处理该异常。构造一个`Promise`的方法如下：
```js
const myFirstPromise = new Promise((resolve, reject) => {
  // do something asynchronous which eventually calls either:
  //
  //   resolve(someValue); // fulfilled
  // or
  //   reject("failure reason"); // rejected
});
```
有了`Promise`我们可以将以上执行A，B，C三个异步函数改写为：
```js
function asyncA () {
  // Do something
  return new Promise((resolve, reject) => {
    // resolve(data)
  });
}
function asyncB () {
  // Do something
  return new Promise((resolve, reject) => {
    // resolve(data)
  });
}
function asyncC () {
  // Do something
  return new Promise((resolve, reject) => {
    // resolve(null)
  });
}

function main(callback) {
  // Do something.
  return new Promise((resolve, reject) => {
    // resolve(data);
  });
}

main.then(asyncA)
    .then(asyncB)
    .then(asyncC)
    .catch((error) => throw new Error(err.message));
```
我们重新构造了`asyncA`, `asyncB`, `asyncC`函数，让他们的回调被`Promise`封装，无需每个回调中都需要单独处理异常。当`main`函数执行完毕时，我们只需要使用简单的`then`就可以将`main`函数执行结果传递给`aysncA`，`aysncA`执行完毕再传递给`asyncB`以此类推。`then`是`Promise`提供的方法，可以链式的将上一个`Promise`对象的`resolve`和`reject`传递给下一个`Promise`对象。这样整个代码的可读性得到了大大的改善。

更巧妙的是`Promise`的异常处理，无需在链式调用的每个异步回调函数中做判断，使用`Promise`的`catch`接口，类似于Java的`try catch`一样，一旦发生异常，可以在一个地方进行判断处理。`Promise`允许在链式调用中定义多个`catch`，异常发生后，会往后寻找，离当前执行函数最近的`catch`来处理。

## 总结
本文介绍了Node.js异步编程的基本方法，可以看到的是Node.js的天然异步机制使得程序的执行效率和线程资源的使用率大大得到提高。如果使用Node.js作为Web应用的开发，优点显而易见，前后端都可使用Javascript作为编程语言，而且无论是Node.js还是前端的React，Vue都有庞大的生态圈，有丰富的第三方库可供使用，节省开发成本。但Node.js的缺点在异步机制中也得到了体现，异步编程思维更为复杂，对逻辑能力有更高的要求，同时代码可读性也有一定的牺牲。另外，由于Event Loop主要是针对Web应用请求设计的，当需要处理响应时间较长的请求时，例如需要消耗大量CPU资源进行计算时，Event Loop的稳定性被证明并不好。另外，对于数据库操作的支持，与其他语言的MVC框架相比，也没有很友好的中间件支持。

## 引用
1. https://dev.to/siwalikm/async-programming-basics-every-js-developer-should-know-in-2018-a9c
2. https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
3. https://nodejs.dev/introduction-to-nodejs
4. http://nqdeng.github.io/7-days-nodejs/#7.7
5. https://itnext.io/multi-threading-and-multi-process-in-node-js-ffa5bb5cde98
6. https://www.youtube.com/watch?v=zphcsoSJMvM
