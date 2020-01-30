---
layout: post
title: koa解读
subtitle: koa解读
date: 2019-10-05
author: NYC
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
  - Git/webpack
---



对象可以定义自己的 [util.inspect.custom](depth, opts)（或已废弃的 inspect(depth, opts)） 函数，util.inspect() 会调用并使用查看对象时的结果.

delegate模块    获取值等操作，第一个对象是被执行的对象
ex：

```
示例：  把reponse对象的特定属性赋值到proto对象上去。

delegate(proto, 'response')
  .method('attachment')
  .method('redirect')
  .method('remove')
  .method('vary')
 

```

## koa的1版本兼容

```
  use(fn) {   
	//  1.x版本和2.x版本的差别。convert
    if (typeof fn !== "function")
      throw new TypeError("middleware must be a function!");
    if (isGeneratorFunction(fn)) {
      deprecate(
        "Support for generators will be removed in v3. " +
          "See the documentation for examples of how to convert old middleware " +
          "https://github.com/koajs/koa/blob/master/docs/migration.md"
      );
      fn = convert(fn);
    }
    debug("use %s", fn._name || fn.name || "-");
    this.middleware.push(fn);
    return this;
  }
  
  
```

##application.js

```
四个作用：
// 通过http启动web服务，创建koa实例。
// 处理合并中间件为洋葱模型。
// 创建和封装高内聚的context。
// 实现异步函数的统一错误处理机制。

   //监测服务端主动推送
    if ("HEAD" == ctx.method) {
    if (!res.headersSent && isJSON(body)) {
      ctx.length = Buffer.byteLength(JSON.stringify(body));
    }
    return res.end();
  }
   // 对流/buffer/string分别处理
  if (Buffer.isBuffer(body)) return res.end(body);
  if ("string" == typeof body) return res.end(body);
  if (body instanceof Stream) return body.pipe(res);
```


## request.js
```
  get host() {
    // 获取当前主机（hostname:port）。当 app.proxy 是 true 时支持 X-Forwarded-Host，否则使用 Host。和ip定位相关
    const proxy = this.app.proxy;
    let host = proxy && this.get('X-Forwarded-Host');
    if (!host) {
      if (this.req.httpVersionMajor >= 2) host = this.get(':authority');
      // http2.0压缩请求头部
      if (!host) host = this.get('Host');
    }
    if (!host) return '';
    return host.split(/\s*,\s*/, 1)[0];
  },
```


## 常见中间件

#### 区分
```
router/mount： 生成中间价被use， 生成一堆能够被app.use的中间件
view/bodypaser： 挂载到context上.还有ctx.body
send/static:   直接调用app.use()
```

#### bodypaser
```
const readStream = require('./lib/read_stream');
let strictJSONReg = /^[\x20\x09\x0a\x0d]*(\[|\{)/;

let jsonTypes = [
  'application/json'
];

let formTypes = [
  'application/x-www-form-urlencoded'
];

let textTypes = [
  'text/plain'
];

function parseQueryStr(queryStr) {
  let queryData = {};
  let queryStrList = queryStr.split('&');
  for (let [ index, queryStr ] of queryStrList.entries()) {
    let itemList = queryStr.split('=');
    queryData[ itemList[0] ] = decodeURIComponent(itemList[1]);
  }
  return queryData;
}

function bodyParser(opts = {}) {
  return async function(ctx, next) {
    // 拦截post请求
    if (!ctx.request.body && ctx.method === 'POST') {
      // 解析请求体中的表单信息
      let body = await readStream(ctx.request.req);
      let result = body;
      if (ctx.request.is(formTypes)) {
        result = parseQueryStr(body);
      } else if (ctx.request.is(jsonTypes)) {
        if (strictJSONReg.test(body)) {
          try {
            result = JSON.parse(body);
          } catch (err) {
            ctx.throw(500, err);
          }
        }
      } else if (ctx.request.is(textTypes)) {
        result = body;
      }

      // 将请求体中的信息挂载到山下文的request 属性中
      ctx.request.body = result;
    }
    await next();
  };
}

module.exports = bodyParser;

```


#### co

```
function co(gen) {
  var ctx = this;
  var args = slice.call(arguments, 1);
  return new Promise(function(resolve, reject) {
    if (typeof gen === "function") gen = gen.apply(ctx, args);
    if (!gen || typeof gen.next !== "function") return resolve(gen);

    onFulfilled();
    function onFulfilled(res) {
      var ret;
      try {
        ret = gen.next(res);
      } catch (e) {
        return reject(e);
      }
      next(ret);
      return null;
    }
    function onRejected(err) {
      var ret;
      try {
        ret = gen.throw(err);
      } catch (e) {
        return reject(e);
      }
      next(ret);
    }
    function next(ret) {
      if (ret.done) return resolve(ret.value);
      var value = toPromise.call(ctx, ret.value);
      if (value && isPromise(value)) return value.then(onFulfilled, onRejected);
      return onRejected(
        new TypeError(
          "You may only yield a function, promise, generator, array, or object, " +
            'but the following object was passed: "' +
            String(ret.value) +
            '"'
        )
      );
    }
  });
}

// 整个函数返回一个promise对象，这与async/await一致。
// 在返回的promise中，首先判断是否为可执行的生成器函数，然后调用函数，获取到遍历对象。
// 然后第一次手动执行onFulfilled函数，这个函数就是来调用next()方法的。
// 声明onRejected函数，主要用来调用生成器的throw()方法来结束执行和报错的。
// 在3、4步骤中，只要调用了g.next()方法，最终都会调用co自己声明的next函数，这个函数的主要工作就是将给promise的value转成promise，然后再在promise的下一个异步去调用onFulfilled和onRejected函数，以此往复。


```

#### delegate

```
function Delegator(proto, target) {
    if (!(this instanceof Delegator)) return new Delegator(proto, target);
    this.proto = proto;
    this.target = target;
    this.methods = [];
    this.getters = [];
    this.setters = [];
    this.fluents = [];
  }
  
// const x = Delegator(proto, 'sbjj')
// const x = new Delegator(proto, 'sbjj')


Delegator.prototype.getter = function(name){
    var proto = this.proto;
    var target = this.target;
    this.getters.push(name);
  
    proto.__defineGetter__(name, function(){  // 兼容性
      return this[target][name];
    });
  
    return this;
};
Delegator.prototype.setter = function(name){
    var proto = this.proto;
    var target = this.target;
    this.setters.push(name);
  
    proto.__defineSetter__(name, function(val){
      return this[target][name] = val;
    });
  
    return this;
  };
  
// 给外部对象调用内部函数的功能
  Delegator.prototype.method = function(name){
    var proto = this.proto;
    var target = this.target;
    this.methods.push(name);
  
    proto[name] = function(){
      return this[target][name].apply(this[target], arguments);
    };
  
    return this;
};
  

```


#### compose 
```
'use strict'

/**
 * Expose compositor.
 */

module.exports = compose

/**
 * Compose `middleware` returning
 * a fully valid middleware comprised
 * of all those which are passed.
 *
 * @param {Array} middleware
 * @return {Function}
 * @api public
 */

function compose (middleware) {
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
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i]
      if (i === middleware.length) fn = next
      if (!fn) return Promise.resolve()
      try {
        return Promise.resolve(fn(context, dispatch.bind(null, i + 1)));
      } catch (err) {
        return Promise.reject(err)
      }
    }
  }
}


```

#### koa-router

```
const methods = [
    'GET',
    'PUT',
    'PATCH',
    'POST',
    'DELETE'
  ];
  
  class Layer {
    constructor(path, methods, middleware, opts) {
      this.path = path;
      this.methods = methods;
      this.middleware = middleware;
      this.opts = opts;
    }
  }
  
  class Router {
    constructor(opts = {}) {
      this.stack = [];
    }
  
    register(path, methods, middleware, opts) {
      let route = new Layer(path, methods, middleware, opts);
      this.stack.push(route);
      return this;
    }
  
    routes() {
  
      let stock = this.stack;
      return async function(ctx, next) {
        let currentPath = ctx.path;
        let route;
  
        for (let i = 0; i < stock.length; i++) {
          let item = stock[i];
          if (currentPath === item.path && item.methods.indexOf(ctx.method) >= 0) {
            route = item.middleware;
            break;
          }
        }
  
        if (typeof route === 'function') {
          route(ctx, next);
          return;
        }
  
        await next();
      };
    }
  }
  
  methods.forEach(method => {
    Router.prototype[method.toLowerCase()] = Router.prototype[method] = function(path, middleware) {
      this.register(path, [method], middleware);
    };
  });
  
  module.exports = Router;

```


#### koa-send

```
const fs = require('fs');
const path = require('path');
const {
  basename,
  extname
} = path;

const defaultOpts = {
  root: '',
  maxage: 0,
  immutable: false,
  extensions: false,
  hidden: false,
  brotli: false,
  gzip: false,
  setHeaders: () => {}
};

async function send(ctx, urlPath, opts = defaultOpts) {
  const { root, hidden, immutable, maxage, brotli, gzip, setHeaders } = opts;
  let filePath = urlPath;

  // step 01: normalize path
  // 配置静态资源绝对目录地址
  try {
    filePath = decodeURIComponent(filePath);
    // check legal path
    if (/[\.]{2,}/ig.test(filePath)) {
      ctx.throw(403, 'Forbidden');
    }
  } catch (err) {
    ctx.throw(400, 'failed to decode');
  }

  filePath = path.join(root, urlPath);
  const fileBasename = basename(filePath);

  // step 02: check hidden file support
  // 判断是否支持隐藏文件
  if (hidden !== true && fileBasename.startsWith('.')) {
    ctx.throw(404, '404 Not Found');
    return;
  }

  // step 03: stat
  // 获取文件或者目录信息
  let stats; 
  try { 
    stats = fs.statSync(filePath);
    if (stats.isDirectory()) {
      ctx.throw(404, '404 Not Found');
    }
  } catch (err) {
    const notfound = ['ENOENT', 'ENAMETOOLONG', 'ENOTDIR']
    if (notfound.includes(err.code)) {
      ctx.throw(404, '404 Not Found');
      return;
    }
    err.status = 500
    throw err
  }

  let encodingExt = '';
  // step 04 check zip
  // 判断是否需要压缩
  if (ctx.acceptsEncodings('br', 'identity') === 'br' && brotli && (fs.existsSync(filePath + '.br'))) {
    filePath = filePath + '.br';
    ctx.set('Content-Encoding', 'br');
    ctx.res.removeHeader('Content-Length');
    encodingExt = '.br';
  } else if (ctx.acceptsEncodings('gzip', 'identity') === 'gzip' && gzip && (fs.existsSync(filePath + '.gz'))) {
    filePath = filePath + '.gz';
    ctx.set('Content-Encoding', 'gzip');
    ctx.res.removeHeader('Content-Length');
    encodingExt = '.gz';
  }

  // step 05 setHeaders
  // 设置HTTP头信息
  if (typeof setHeaders === 'function') {
    setHeaders(ctx.res, filePath, stats);
  }

  ctx.set('Content-Length', stats.size);
  if (!ctx.response.get('Last-Modified')) {
    ctx.set('Last-Modified', stats.mtime.toUTCString());
  }
  if (!ctx.response.get('Cache-Control')) {
    const directives = ['max-age=' + (maxage / 1000 | 0)];
    if (immutable) {
      directives.push('immutable');
    }
    ctx.set('Cache-Control', directives.join(','));
  }

  const ctxType = encodingExt !== '' ? extname(basename(filePath, encodingExt)) : extname(filePath);
  ctx.type = ctxType;

  // step 06 stream
  // 静态文件读取
  ctx.body = fs.createReadStream(filePath);
}

module.exports = send;
```

#### REadSTRAM
```
module.exports = readStream;

function readStream(req) {
  return new Promise((resolve, reject) => {
    try {
      streamEventListen(req, (data, err) => {
        if (data && !isError(err)) {
          resolve(data);
        } else {
          reject(err);
        }
      });
    } catch (err) {
      reject(err);
    }
  });
}

function isError(err) {
  return Object.prototype.toString.call(err).toLowerCase() === '[object error]';
}

function streamEventListen(req, callback) {
  let stream = req.req || req;
  let chunk = [];
  let complete = false;

  // attach listeners
  stream.on('aborted', onAborted);
  stream.on('close', cleanup);
  stream.on('data', onData);
  stream.on('end', onEnd);
  stream.on('error', onEnd);

  function onAborted() {
    if (complete) {
      return;
    }
    callback(null, new Error('request body parse aborted'));
  }

  function cleanup() {
    stream.removeListener('aborted', onAborted);
    stream.removeListener('data', onData);
    stream.removeListener('end', onEnd);
    stream.removeListener('error', onEnd);
    stream.removeListener('close', cleanup);
  }

  function onData(data) {
    if (complete) {
      return;
    }
    if (data) {
      chunk.push(data.toString());
    }
  }

  function onEnd(err) {
    if (complete) {
      return;
    }

    if (isError(err)) {
      callback(null, err);
      return;
    }

    complete = true;
    let result = chunk.join('');
    chunk = [];
    callback(result, null);
  }
}
```