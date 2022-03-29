# express原理解析
## express的本质
express通过工厂函数（createApplication）生成了一个函数，生成的函数作为http.createServer的回调函数，当请求来的时候会调用这个回调函数对请求进行处理。
```javascript
原生创建http server 
  consthttp = require('http');
  http.createServer(req, res => {
    //对请求进行处理

  }).listen(3000);
```
```javascript
  使用express
 const express = require('express');
 const http = require('http');
 const app = express();
 app.get('/', (req,res) => {
    //处理请求
    
 })
 http.createServer(app).listen(3000);
```
## 中间件原理
中间件其实就是一个函数，当请求来了express根据url来匹配中间件，如果匹配成功则调用对应的中间件。express使用了异步串行来执行中间件，执行顺序和在代码中的定义的顺序相同，异步串行的实现是把所有的中间件按照定义时的顺序加入到一个数组中去，执行的时候从数组中取出并执行，执行的时候如果调用了next函数则取出下一个中间件进行执行，没有调用next则结束中间件的调用。
## 源码目录
```javascript
middleware  中间件目录
init.js 扩展req,res，初始化locals
query.js 解析url中的请求参数，封装到req.query中
router 
index.js 该文件有一个stack存储了所有的中间件（中间件的核心实现）
layer.js 封装中间件，把中间件存到layer对象中
route.js 生成一个独立的路由
application.js  扩展http.createServer回调函数的功能
express.js 项目入口，该文件中有一个工厂函数会生成http.createServer的回调函数
request.js 扩展req的功能
response.js 扩展res的功能
utils.js  常用的工具函数
view.js 存储视图信息
```
## 一次请求执行的过程
求到来时，调用回调函数app，app调用application.js中的handle函数，application.js中的handle函数调用router/index.js中handle函数，
router/index.js中的hanle函数调用layer.js中的handle_request函数，layer.js中的handle_request函数会执行匹配成功的中间件</br>
express.js  传给http.createServer的回调函数</br>
![avatar](../public/img/app-func.png)</br>
application.js</br>
![avatar](../public/img/hanle-func.png)</br>
router/index.js router.stack存放了所有的中间件和路由，遍历stack，取出存放的路由和中间件和当前的请求路径进行匹配，匹配成功则执行，不成功则和下一个进行匹配</br>
![avatar](../public/img/proto-handle-func.png)</br>
router/layer.js   一个layer对象对应一个中间件，在layer.handle_request中执行中间件</br>
![avatar](../public/img/hanle_request.png)</br>
## epxress创建实例的入口
```javascript
function createApplication() {//工厂函数
  var app = function(req, res, next) {//该函数是最终返回给http.createServer的回调函数
    app.handle(req, res, next);
  };


  mixin(app, EventEmitter.prototype, false);//混入EventEmitter.prototype的属性
  mixin(app, proto, false);//混入application.js中的一些函数


  // expose the prototype that will get set on requests
  app.request = Object.create(req, {//基于request.js创建
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })


  // expose the prototype that will get set on responses
  app.response = Object.create(res, {//基于reponse.js创建
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })


  app.init();//做了一些初始化的操作
  return app;
}
```
## 注册中间件及中间件执行
```javascript
app.use = function use(fn) {//注册中间件的函数
  var offset = 0;
  var path = '/';//默认路径


  // default path to '/'
  // disambiguate app.use([fn])
  if (typeof fn !== 'function') {//如果fn不是函数
    var arg = fn;


    while (Array.isArray(arg) && arg.length !== 0) {//取出第一个参数作为路径
      arg = arg[0];
    }


    // first arg is the path
    if (typeof arg !== 'function') {
      offset = 1;
      path = fn;
    }
  }


  var fns = flatten(slice.call(arguments, offset));//取出所有的回调函数


  if (fns.length === 0) {
    throw new TypeError('app.use() requires a middleware function')
  }


  // setup router
  this.lazyrouter();
  var router = this._router;


  fns.forEach(function (fn) {
    // non-express app
    if (!fn || !fn.handle || !fn.set) {
      return router.use(path, fn);//注册中间件
    }


    debug('.use app under %s', path);
    fn.mountpath = path;
    fn.parent = this;


    // restore .app property on req and res
    router.use(path, function mounted_app(req, res, next) {
      var orig = req.app;
      fn.handle(req, res, function (err) {
        setPrototypeOf(req, orig.request)
        setPrototypeOf(res, orig.response)
        next(err);
      });
    });


    // mounted an app
    fn.emit('mount', this);
  }, this);


  return this;
};

app.handle = function handle(req, res, callback) {//该方法在http.createServer中的回调函数中调用
  var router = this._router;//获取router


  // final handler
  var done = callback || finalhandler(req, res, {//如果回调不存在则获取最终的处理
    env: this.get('env'),
    onerror: logerror.bind(this)
  });


  // no routes
  if (!router) {
    debug('no routes defined on app');
    done();//没有router则调用最终的处理
    return;
  }


  router.handle(req, res, done);//调用router的handle函数进行处理
};
```
