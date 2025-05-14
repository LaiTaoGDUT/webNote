#### koa洋葱模型详解

##### 洋葱模型是什么

刚刚我们说到，koa的精髓就是它的中间件机制，而它使用的原理是洋葱模型，而洋葱模型是由多个同心层构成，它们相互连接，朝向代表核心的领域。并且基于控制反转。它有着以下原则：

**依赖性**

洋葱圈模型的每一个圆圈代表不同的责任层。一般来说，我们潜入得越深，就越接近于业务核心。通常外圈代表机制，内圈代表核心业务逻辑。属于内层的方法、变量和源代码可以依赖于外层，反过来也一样。

**数据封装**

每个责任层需要封装或隐藏内部的实现细节，并提供便于其他层消费的信息。其目的是最小化层与层之间的耦合，最大化跨层垂直切面内的耦合。

**关注点分离**

应用被分为若干层，每一层都有自己独立的职责，并解决不同的关注点。

**低耦合**

低耦合性，可以使一个层与另一个层交互，而不需要关注另一个层的内部。


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8e53ad0ae15b4ec990f909087631fe12~tplv-k3u1fbpfcp-watermark.image?)


我们可以看出洋葱模型是**职责链模式**的一种应用。

![在这里插入图片描述](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a58983f7e33044fdaf3c76e180bff8b7~tplv-k3u1fbpfcp-zoom-1.image)

职责链会将特定行为转换为被称作处理者的独立对象，每个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，每个处理者对象都可以对请求进行处理，如果不能处理，也可以直接传给下一个处理者对象，直到有对象处理为止。

-   降低处理者对象之间的耦合度。该模式使得一个对象无须知道到底是哪一个对象处理其请求以及链的结构，发送者和接收者也无须拥有对方的明确信息。
-   增强了系统的可扩展性。可以根据需要增加新的请求处理类，满足开闭原则。
-   可插拔，增强了给对象指派职责的灵活性。当工作流程发生变化，可以动态地改变链内的成员或者调动它们的次序，也可动态地新增或者删除责任。
-   责任链简化了对象之间的连接。每个对象只需保持一个指向其后继者的引用，不需保持其他所有处理者的引用，这避免了使用众多的 if 或者 if···else 语句。
-   责任分担。每个类只需要处理自己该处理的工作，不该处理的传递给下一个对象完成，明确各类的责任范围，符合类的单一职责原则。




koa2中的洋葱模型，就是指每一个 Koa 中间件都是一个责任层，它即可以掌管请求进入，也可以掌管响应返回。外层的中间件可以影响内层的请求阶段，内层的中间件可以影响外层的响应阶段。

koa中可以对每一个请求使用多个中间件，每个中间件会依次处理，并可以通过`next`方法暂停当前执行，待后续中间件执行完毕后继续往下执行。通过这种执行流程，开发者可以非常方便的开发一些中间件，中间件的功能可以非常自由，并且非常容易的整合到实际业务流程中。

基本上，Koa 所有的功能都是通过中间件实现的，关于koa的中间件洋葱执行模型，koa1中使用的是generator 执行的方式，koa2中则使用了async/await，其实它们是差不多的。下面主要讲koa2的中间件的使用和实现。



##### koa2洋葱模型中间件的使用

koa中间件的使用非常简单，往app.use方法中传入一个函数，那么这个函数就成了这个请求中会用到的一个中间件，每个中间件默认接受两个参数，第一个参数是`Context`对象，第二个参数是`next`函数。按照调用app.use的顺序，所有中间件排成了一个栈，以先进后出的方式执行，中间件只要调用`next`函数，就可以把执行权转交给下一个中间件，中间件执行完毕后，就把执行权重新交还给上一个中间件。

1.  最外层的中间件首先执行。
1.  调用`next`函数，把执行权交给下一个中间件。
1.  ...
1.  最内层的中间件最后执行。
1.  执行结束后，把执行权交回上一层的中间件。
1.  ...
1.  最外层的中间件收回执行权之后，执行`next`函数后面的代码。

```
let Koa = require('koa');

let app = new Koa();

app.use((ctx, next)=>{
    console.log(1)
    next();
    console.log(1)
});

app.use((ctx, next) => {
    console.log(2)
    next();
})

app.use((ctx, next) => {
    console.log(3)
  next();
})

app.listen(3000, () => {
    console.log('listenning on 3000');
});

// log
// 1 2 3 1
```

上面所示的例子，所有中间件都是同步的，不包含异步操作。如果中间件内包含了异步操作，那么中间件和它的上游中间件就必须写成 async函数。

```
let Koa = require('koa');

 let app = new Koa();
 
 app.use(async (ctx, next)=>{
     console.log(1)
     await next();
     console.log(1)
 });
 
 app.use(async (ctx, next) => {
    await new Promise(resolve => {
        resolve();
   })
    console.log(2);
    next();
 })
 
 app.use((ctx, next) => {
     console.log(3);
 })
 
 app.listen(3000, () => {
     console.log('listenning on 3000');
 });

// log ???
```




```
let Koa = require('koa');

let app = new Koa();
 
 app.use((ctx, next)=>{
     console.log(1)
     next();
     console.log(1)
 });
 
 app.use(async (ctx, next) => {
    await new Promise(resolve => {
        resolve();
   })
    console.log(2);
    next();
 })
 
 app.use((ctx, next) => {
     console.log(3);
 })
 
 app.listen(3000, () => {
     console.log('listenning on 3000');
 });

// log ???
```




中间件的使用方法三言两语就说完了，从上面的这些例子我们就可以看出一些洋葱圈模型实现的一些端倪，比如说每个中间件传入的`next`函数，它内部会去执行下一个中间件函数，并且`next`函数需要返回是一个`Promise`。




##### koa2洋葱模型中间件的实现

中间件的实现有两个最重要的点

1.  next的实现
1.  中间件的管理

如果要去实现一个洋葱模型的中间件，可以先看next的实现，假设现在我们有三个async函数

```
async function m1(next) {
    console.log('m1');
    await next();
}

async function m2(next) {
    console.log('m2');
    await next();
}

async function m3() {
    console.log('m3');
}
```

我们希望能实现让三个函数能按先进后出的方式执行。先考虑在m2的执行过程中，如果遇到`await next()`，就得去执行m3函数，然后让m2等待，那么显然我们就需要构造一个next函数，作用是调用m3，然后把next作为参数传给m2

```
const next1 = async function () {
    await m3();
}

m2(next1);

// 输出：m2,m3
```

再进一步，考虑从m1开始执行，那么m1的next参数需要的是一个执行m2的函数，并且给m2传入的next参数需要是一个执行m3的函数

```
let next1 = async function () {
    await m3();
}

let next2 = async function () {
    await m2(next1);
}

m1(next2);

// 输出：m1,m2,m3
```

通过手动实现每个中间件的next，我们让三个中间件以洋葱模型的规则去执行了，将这个思路拓展到n个中间件，我们会考虑到实现一个批量操作函数，调用一次批量操作函数，就去倒序遍历中间件列表，批量创建所有中间件的next函数，然后将next函数作为参数依次传给上一个中间件，构造好它们的联系，最后执行。那么我们只需要在请求打过来的时候调用一下，所有的中间件就可以按我们编排好的顺序来执行了。koa2对于中间件就是这么管理的，把所有中间件合成一个函数。

按照我们上面的思路，最容易想到的就是创建一个工厂方法`createNext`，它为每个中间件生成调用它的next函数，然后遍历中间件数组，倒序生成执行链。

```
// Application.js
class Application {
  constructor() {
    this.middlewares = [];
    this.context = context;
 }
  ...
  use(middleware) {
    this.middlewares.push(middleware);
 }

  compose(middlewares) {
    return function (ctx, lastNext) {

      function createNext(middleware, oldNext) {
        return async () => {
          await middleware(ctx, oldNext);
       }
     }

      let len = middlewares.length;
      let next = lastNext;
      if (!next) next = () => {
        return Promise.resolve();
     };
      for (let i = len - 1; i >= 0; i--) {
        let currentMiddleware = middlewares[i];
        next = createNext(currentMiddleware, next);
     }

      await next();
   }
 }

 callback() {
   const fn = this.compose(this.middlewares);
   await fn(ctx);
 }

  listen(...args) {
    let server = http.createServer(this.callback);
    server.listen(...args);
 }
 ...
}
  // app.use(middleware1);
  // app.use(middleware2);
  // app.listen(3000);
```




从上面的实现中我们看到，可以先倒序生成了所有中间件的next函数并把它们的联系构造好了之后，再顺序执行，反过来，我们也可以按顺序的方式，边执行边生成next函数，也就是koa2中的实现方式，我们实现一个dispatch函数，它接收中间件数组下标作为参数，将下标 i 传给dispatch函数，dispatch函数需要执行下标 i 指向的中间件，并把dispatch(i + 1)作为next参数传给当前执行的中间件，然后我们在请求打过来的时候执行dispatch(0)即可。

```
```

```
// Application.js
class Application {
  constructor() {
    this.middlewares = [];
    this.context = context;
  }
  ...
  use(middleware) {
    this.middlewares.push(middleware);
  }

  compose(middlewares) {
    return function (ctx, next) {
      // last called middleware #
      let index = -1
      function dispatch (i) {
        if (i <= index) return Promise.reject(new Error('next() called multiple times'))
        index = i
        let fn = middlewares[i]
        if (i === middlewares.length) fn = next
        if (!fn) return Promise.resolve()
        try {
          return Promise.resolve(fn(ctx, dispatch.bind(null, i + 1)));
        } catch (err) {
          return Promise.reject(err)
        }
      }
      return dispatch(0)
    }
  }

	callback() {
   const fn = this.compose(this.middlewares);
   await fn(ctx);
  }

  listen(...args) {
    let server = http.createServer(this.callback);
    server.listen(...args);
  }
	...
}
  // app.use(middleware1);
  // app.use(middleware2);
  // app.listen(3000);
```




还可以有其他实现的思路吗？

##### 洋葱模型的精妙之处

可以说，koa能这么流行的原因，很大程度归功于洋葱模型的中间件流程设计，为什么这么说呢，我们拿express来比较一下，express也有中间件，但是它是基于回调实现的线形模型，中间件是顺序执行的，不利于组合，不利于互操，并且实现也没有koa来得简单。举一个简单的例子感受一下，假如我们想实现一个记录请求响应时间的中间件：




使用express需要这么写：

```
const express = require('express')
const app = express()
app.use((req, res, next) => {
  req.requestTime = Date.now()
  next()
});
app.get('/', function (req, res) {
  var responseText = 'Hello World!';
  req.setHeader('x-response-time', Date.now() - req.requestTime);
  res.send(responseText);
})
app.listen(3000);
```




而使用koa2需要这么写：

```
const Koa = require('../lib/application')
const app = new Koa()
app.use(async (ctx, next) => {
  const start = Date.now()
  await next()
  const ms = Date.now() - start
  ctx.set('X-Response-Time', `${ms}ms`)
})
app.use(async (ctx, next) => {
  ctx.body = 'Hello World!'
  next()
})
app.listen(3000, () => {
  console.log('listenning on 3000')
})
```

我们可以看到，express实现就对业务代码有一定程度的侵扰，很容易会造成不同中间件间的耦合，毫无疑问 Koa 的中间件更加先进，写法也更加优雅。而Express 的线形机制不容易实现拦截处理逻辑：比如异常处理和统计响应时间，这在 Koa 里，一般只需要一个中间件就能全部搞定。