## Koa

### 概述

​	Koa是一个**web框架**， 其目的是为Web应用程序和API提供更小、更富表现力、更健壮的基石 。

特点：

​	中间件架构

​	增强错误处理

​	轻量无捆绑

​	提供优雅的方法用于编写服务器代码



## 基本结构

```
.
├── application.js
├── request.js
├── response.js
└── context.js
```

application.js：中间件的管理、`http.createServer`的回调处理，生成`Context`作为本次请求的参数，并调用中间件。

request.js ：为了简化一些 对`http.createServer`回调的参数`request` 的常用操作， 进行封装

response.js：为了简化一些 对`http.createServer`回调的参数`response`的常用操作， 进行封装

context.js：整合`request`与`response`的部分功能，并提供一些额外的功能



示例：

封装`http.createServer`回调的参数`request` 和`response`后get 或set Header 方式如下：

```js
// 获取Content-Type
context.request.get('Content-Type')

// 设置Content-Type
context.response.set({
  'Content-Type': 'application/json',
  'Content-Length': '18'
})
```

把request.get 和 response.set 代理到`context`上，get 或set Header 方式如下：

```js
context.get('Content-Type')

// 设置Content-Type
context.set({
  'Content-Type': 'application/json',
  'Content-Length': '18'
})
```

#### context

对`request`  和`response` 进行封装会导致get 或 set Header层级更深，需要先通过`context`找到`request`、`response`，然后才能进行操作。为了简化操作，把`request`和`response`的一些方法和属性代理到`context`上。

Delegate#method(name)

​	允许在host上访问给定的方法名。

Delegate#access(name)

​	 为委托对象上具有给定名称的属性创建"accessor"(即getter和setter)。

```js
delegate(proto, 'response')
  .method('attachment') 
  .method('redirect')
  .method('remove')
  .method('vary')
  .method('set')
  .method('append')
  .method('flushHeaders')
  .access('status')
  .access('message')
  .access('body')
  .access('length')
  .access('type')
  .access('lastModified')
  .access('etag')
  .getter('headerSent')
  .getter('writable');

/**
 * Request delegation.
 */

delegate(proto, 'request')
  .method('acceptsLanguages')
  .method('acceptsEncodings')
  .method('acceptsCharsets')
  .method('accepts')
  .method('get')
  .method('is')
  .access('querystring')
  .access('idempotent')
  .access('socket')
  .access('search')
  .access('method')
  .access('query')
  .access('path')
  .access('url')
  .access('accept')
  .getter('origin')
  .getter('href')
  .getter('subdomains')
  .getter('protocol')
  .getter('host')
  .getter('hostname')
  .getter('URL')
  .getter('header')
  .getter('headers')
  .getter('secure')
  .getter('stale')
  .getter('fresh')
  .getter('ips')
  .getter('ip');

```



### 工作原理

#### Hello World

让我们从构建一个返回'Hello World' 的http服务开始学习koa。

直接使用node的http模块，构建方式如下：

```js
const http = require('http')

//3. 监听到3000端口上有http请求，就返回'Hello World'
const helloWorldFunc = (request, response) => {
    response.end('Hello World') 
}

http.createServer(helloWorldFunc) //1. 创建http服务
    .listen(3000, _ => console.log('Server run as http://127.0.0.1:3000')) //2. 当监听到3000端口上有http请求，执行helloWorldFunc函数返回'Hello World'
```

构建流程如下：

1. 首先使用http模块**createServer**方法创建http 服务

2. 然后使用http模块的**listen**方法启动http 服务并在指定**端口**上进行监听

3. 当监听到指定的端口上有http请求，对请求进行处理和响应



在koa中构建相同http服务方式如下：

```js
const Koa = require('koa');
const app = module.exports = new Koa(); // 通过Koa构造函数创建Koa实例

function helloWorldFunc(ctx) {
    ctx.body = 'Hello World';
}
app.use(helloWorldFunc); // 使用Koa实例的use方法加载fn1

//1. 创建http服务；2. 启动http 服务并在3000端口上监听连接；3. 当监听到3000端口上有http请求，就调用通过koa 的use方法加载的helloWorldFunc函数 返回'Hello World'
app.listen(3000, _ => console.log('Server run as http://127.0.0.1:3000'));

```

构建流程如下：

1. 使用koa的**listen**方法创建http服务，启动http 服务并在指定端口上监听连接。
2. 当监听到指定端口上有http请求，就调用通过koa 的**use**方法加载的方法对请求进行处理和响应

很明显， koa的listen方法对http模块中**http.createServer** 和 **server.listen** 两个方法进行了封装。

#### listen()

```js
  /**
   * Shorthand for:
   *
   *    http.createServer(app.callback()).listen(...)
   *
   * @param {Mixed} ...
   * @return {Server}
   * @api public
   */

  listen(...args) {
    debug('listen');
    const server = http.createServer(this.callback()); // 创建http服务
    return server.listen(...args);// 启动http 服务并在指定的端口上监听连接；
  }
```



在listen方法中当监听到端口上有http请求时，会调用内部函数**callback()**对http请求进行处理并做出响应。

那么**callback()**是如何对http请求进行处理的？

#### callback()

```js
  /**
   * Return a request handler callback
   * for node's native http server.
   *
   * @return {Function}
   * @api public
   */

  callback() {
    const fn = compose(this.middleware); //组合通过use方法加载的中间件

    if (!this.listenerCount('error')) this.on('error', this.onerror);

    const handleRequest = (req, res) => {
	  // 把req, res上的很多方法代理到了ctx上，创建本次请求使用的上下文
      const ctx = this.createContext(req, res);
      return this.handleRequest(ctx, fn);//把处理后的中间件fn和ctx 作为参数传递给 handleRequest
    };

    return handleRequest;
  }

```

`this.middleware` 是一个数组，在调用Koa的**构造函数**时定义并初始化。上述的示例中，`this.middleware` 中保存着helloWorldFunc 方法。Koa通过**use**方法把helloWorldFunc 方法添加到`this.middleware` 中。

callback工作流程：

1. 使用**compose**处理保存在`this.middleware` 的中间件， 将`this.middleware` 中的中间件转换为我们想要的洋葱模型格式**fn**。
2. 通过执行**createContext** 把req, res上的很多方法代理到了ctx上，创建本次请求使用的上下文。
3. 把处理后的中间件fn和ctx 作为参数传递给 **handleRequest**， 在handleRequest处理http请求并做出响应。

这里涉及以下关键点：

​	构造函数，use 方法,  compose方法,  createContext 方法， handleRequest方法

#### Koa构造函数

```js
constructor(options) {
    super(); //访问和调用一个对象的父对象上的函数
    options = options || {};
    this.proxy = options.proxy || false;
    this.subdomainOffset = options.subdomainOffset || 2;
    this.env = options.env || process.env.NODE_ENV || 'development';
    if (options.keys) this.keys = options.keys;
    this.middleware = [];// 用来存储中间件
    this.context = Object.create(context); // 把 context 放入实例
    this.request = Object.create(request);// 把 request 放入实例
    this.response = Object.create(response);// 把 response 放入实例
    if (util.inspect.custom) {
        this[util.inspect.custom] = this.inspect;
    }
}
```

构造函数创建并初始化一些属性。将引入的`context`、`request`以及`response`通过`Object.create`拷贝的方式放到实例中。

#### use(）

用来加载中间件

```js
  /**
   * Use the given middleware `fn`.
   *
   * Old-style middleware will be converted.
   *
   * @param {Function} fn
   * @return {Application} self
   * @api public
   */

  use(fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!');
    if (isGeneratorFunction(fn)) {
      deprecate('Support for generators will be removed in v3. ' +
        'See the documentation for examples of how to convert old middleware ' +
        'https://github.com/koajs/koa/blob/master/docs/migration.md');
      fn = convert(fn);
    }
    debug('use %s', fn._name || fn.name || '-');
    this.middleware.push(fn);
    return this;
  }
```



通过push方法把中间件函数添加到`this.middleware`中



#### koa-compose

```js
const compose = require('koa-compose');
```

##### 作用

​	合并中间件

##### 使用

​	compose([a, b, c, ...])

​	返回值：一个符合洋葱圈模型格式中间件

##### 中间件

​	Koa中间件机制就是将一组需要顺序执行的函数复合为一个函数，外层函数的参数实际是内层函数的返回值。**洋葱圈模型**可以形象表示这种机制。



##### 洋葱圈模型

![洋葱圈](D:\web-learn\node\Documents\node-blog\洋葱圈.png)





##### 源码

```js
 function compose (middleware) {
  console.log("compose middleware=", middleware);
  if (!Array.isArray(middleware)) throw new TypeError('Middleware stack must be an array!')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new TypeError('Middleware must be composed of functions!')
  }

  /**
   * @param {Object} context
   * @return {Promise}
   * @api public
   */

  return function (context, next) {
    // last called middleware #
    let index = -1
    return dispatch(0)
    function dispatch (i) {
      //判断中间件的下标，防止中间件多次执行next
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i] //取出中间件
      if (i === middleware.length) fn = next // middleware中的中间件函数全部执行完毕
      if (!fn) return Promise.resolve() //middleware中最后一个中间件函数调用next()
      try {
        //将自身绑定的
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}

```



在compose方法中 从middleware数组中按顺序取出中间件并执行。

示例：

```js
function func1(ctx, next) {
    console.log("func1 start");
    next();
    console.log("func1 end");
}
function func2(ctx, next) {
    console.log("func2 start");
    next()
    console.log("func2 end");
}

app.use(func1);
app.use(func2);
```



输出结果：

```
func1 start
func2 start
func2 end
func1 end
```



##### 工作流程

![洋葱圈工作流程](D:\web-learn\node\Documents\node-blog\洋葱圈工作流程.jpg)

总结：

通过示例可以看到中间件以"先进后出"（first-in-last-out）的顺序执行。

当前中间件通过`next` 进入下一个中间件

`next`在当前中间件执行完毕后进行`resolve`，返回到上一层中间件



#### createContext()

创建本次请求使用的上下文

```js
createContext(req, res) {
    const context = Object.create(this.context);
    const request = context.request = Object.create(this.request);
    const response = context.response = Object.create(this.response);
    context.app = request.app = response.app = this;
    context.req = request.req = response.req = req;
    context.res = request.res = response.res = res;
    request.ctx = response.ctx = context;
    request.response = response;
    response.request = request;
    context.originalUrl = request.originalUrl = req.url;
    context.state = {};
    return context;
}
```



#### handleRequest()

处理http请求并做出响应。

```js
  /**
   * Handle request in callback.
   *
   * @api private
   */

  handleRequest(ctx, fnMiddleware) {
    const res = ctx.res;
    res.statusCode = 404;
    const onerror = err => ctx.onerror(err);
    const handleResponse = () => respond(ctx);
    onFinished(res, onerror); // 监听响应res是否完成，如果响应完成时出错，调用onerror
    return fnMiddleware(ctx).then(handleResponse).catch(onerror);
  }
```



使用onFinished监听响应res是否完成，如果响应完成时出错，调用onerror

执行中间件函数fnMiddleware.

​	如果成功则调用respond处理响应

​	如果失败则执行onerror抛出错误



#### 总结

使用koa构建http服务的步骤如下：

1. 通首先通过**Koa构造函数**创建Koa实例
2. 通过Koa实例的**use**方法加载中间件
3. 通过Koa实例的**listen** 方法监听端口并启动服务
4. 当监听到指定端口上有http请求，调用内部函数**callback()**对http请求进行处理并做出响应。

