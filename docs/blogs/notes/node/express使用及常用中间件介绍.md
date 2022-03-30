# express使用及常用中间件介绍
express是一个基于node.js的轻量级web开发框架，内部实现比较简单，主要是用来开发web或者移动端应用。类似的框架有koa(express原班人马开发)，egg(阿里开源项目，基于koa)等
## 使用场景
### 1.node全栈开发
构建前后端分离应用：vue + express + mysql 前后端不分离：ejs(其他模板引擎) + express + mysql
### 2.中间层开发
中间层开发就是使用express在前端和后端中间加了一层，前端发请求不是直接发往后端，而是先发到中间层，然后中间层再向后端发送请求。请求响应也是后端响应到中间层，中间层在响应到前端。</br>
这样做的好处主要有：</br>
1. 解决跨域问题，在中间层使用CORS协议来解决跨域问题，中间层到后端是不存在跨域问题，所以不需要在后端做跨域处理。
2. 统一在中间层对请求做一些操作，如权限的验证。类似于axios的拦截器，只不过中间层是对请求做定制化处理，而axios的拦截器是对请求做共通化处理。
```javascript
//中间层
router.get('/users', allowd(['admin','superadmin']), async (req, res) => {
  const users = await axios.get('http://localhost:8080');
  res.json({users});
})
//axios
//添加请求拦截器
axios.interceptors.request.use(config=>{
	config.baseURL="https://193.121.12.121:8999/";
  if (token) {  // 判断是否存在token，如果存在的话，则每个http header都加上token
    config.headers['Authorization'] = token;
  } 
	return config;
},(err)=>{
  console.log(error)
  // 发生错误做的一些事情
  return Promise.reject(error);
})
// 添加响应拦截器
axios.interceptors.response.use(res => {
  // 对响应数据做点什么
  const code = res.data.code || 200 //未设置状态默认为200
  //根据错误状态设置错误信息
  if (res.data.code === 400) {
     Message({
      message: '登录失效，请重新登录',
      type: 'error',
      duration: 5 * 1000
    })
  } else if (res.data.code === 401) {
    Message({
      message: '未授权的操作',
      type: 'error',
      duration: 5 * 1000
    })
  } 
},(err)=>{
  console.log('err' + err)
  let { message } = err;
  if (message === "Network Error") {
    message = "后端接口连接异常";
  }
  else if (message.includes("timeout")) {
    message = "系统接口请求超时";
  }
});
```
3. SSR(服务端渲染)，在中间层进行首屏渲染，返回客户端一个有数据的html结构。利于SEO(搜索引擎优化),使网页可以被搜索引擎爬取，提高网站排名。</br>
![avatar](https://upload-images.jianshu.io/upload_images/6522842-06b3ecf2586648fa.png?imageMogr2/auto-orient/strip|imageView2/2/w/1196/format/webp)
![avatar](https://upload-images.jianshu.io/upload_images/6522842-ee4752e9500e9976.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)
![avatar](https://upload-images.jianshu.io/upload_images/6522842-b86735d102cef2c4.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)
中间层可以独立出来，独立部署，也可以放到前端项目中，和前端项目一起部署。

## express的优缺点
### 优点
生态比较完善，各种中间件基本上都有
### 缺点 
处理错误不够友好，中间件内部报错无法在外部捕获，只能通过层层外抛交给全局处理错误的中间件处理
没有对异步进行处理，执行中间件是通过递归然后回调，不够优雅。
## express的中间件
epxress的核心就是中间件，一个express应用就是在调用各种不同的中间件。
中间件的功能
1. 执行任何代码。
2. 修改请求和响应对象。
3. 终结请求-响应 循环。
4. 调用堆栈中的下一个中间件。
### 应用级别的中间件
```javascript
app.use(function(req, res, next) {
  res.setHeader('Access-Control-Allow-Origin', 'http://localhost:8080');
  if (process.env.RESOLVE_CORS) {
    res.setHeader('Access-Control-Allow-Origin', 'http://localhost:8080');
    res.setHeader('Access-Control-Allow-Credentials', true);
    res.setHeader(
      'Access-Control-Allow-Methods',
      'GET, POST, OPTIONS, PUT, DELETE'
    );
    res.setHeader('Access-Control-Allow-Headers', [
      'access-control-allow-origin',
      'x-requested-with'
    ]);
  }
  next();
});
```
### 路由中间件
#### express.Router() *
router是express的路由中间件，express.Router()执行后返回一个对象，该对象是express的路由中间件。可以使用express.Router()进行路由拆分，把对同一资源的请求放在一个模块中，最后在app.js中注册中间件。
```javascript
//app.js
const express = require('express');
const app = express();
const user = require('./user');
app.use('/user',user);
//user.js
const express = require('express');
const router = express.Router();
router.get('/login',function(req,res){
  console.log('login');
})
module.exports = router;
```
### 错误处理中间件
```javascript
app.use(function (err, req, res, next) {
  // set locals, only providing error in development
  res.locals.message = err.message;
  res.locals.error = req.app.get('env') === 'development' ? err : {};

  res.status(err.status || 500).send(res.locals.message);
});
```
### 内置中间件
#### express.static() *
express.static是express的一个中间件，用来提供静态文件服务的。
```javascript
app.use('/static', express.static(__dirname +'public'))
```
#### express.text()
把请求参数解析为字符串
#### express.urlencoded()
解析x-www-form-urlencoded格式的请求数据
```javascript
// express部分源码
/**
 * Expose middleware
 */
exports.json = bodyParser.json
exports.query = require('./middleware/query');
exports.raw = bodyParser.raw
exports.static = require('serve-static');
exports.text = bodyParser.text
exports.urlencoded = bodyParser.urlencoded
```
#### express.json()
expres内置的一个中间件，解析编码为application/json的请求参数，基于 body-parser中间件，v4.16及以上版本支持
#### express.raw()
express用来把请求参数解析到缓冲区的一个中间件，v4.17及以上版本支持
### 第三方中间件
#### express-session
session功能
```javascript
const session = require('express-session'),
app.use(session({
  secret :  'secret', // 对session id 相关的cookie 进行签名
  resave : true,
  saveUninitialized: false, // 是否保存未初始化的会话
  cookie : {
      maxAge : 1000 * 60 * 3, // 设置 session 的有效时间，单位毫秒
  },
}))
app.use(session(initSessionSettings));

router.get('/getLang', [
  function(req, res) {
    req.session.lang = req.query.lang;
    req.session.save(() => { //当session中的数据发生变化时，在响应结束时自动调用该回调函数
      res.redirect(req.headers.referer);//重定向到之前的页面
    });
  }
]);
```
#### cookie-parser
解析cookie
```javascript
const cookieParser = require('cookie-parser');
app.use(cookieParser());
```
#### multer
处理文件上传
```javascript
const multer = require('multer');
const storage = multer.diskStorage({
  destination: function (req, file, cb) {
    cb(null, './public/upload')
  },
  filename: function (req, file, cb) {
    const uniqueSuffix = Date.now() + file.originalname.slice(file.originalname.lastIndexOf('.'))
    cb(null, file.fieldname + uniqueSuffix)
  }
  const upload = multer({ storage })
  // user为上传时file的属性名 upload.single处理上传单个文件，下载文件时需要引入静态文件服务中间件
  app.post('/upload', upload.single('user'), function(req, res){
  res.json({
    url: `http://localhost:3000/static/${req.file.filename}`
  })
})
```
#### cors
处理跨域
```javascript
const cors = require('cors');
app.use(
  cors({
    origin: true,
    credentials: true
  })
)
```
#### body-parser
解析请求参数
```javascript
const bodyParser = require('body-parser');
app.use(bodyParser.json());
app.use(
  bodyParser.urlencoded({
    extended: true
  })
);
```
## express的API
### express() * (标*号的代表是比较常用的api)
创建一个express应用
```javascript
const express = require('express');
const app = express();

app.get('/', function(req, res){
  res.send('hello world');
});

app.listen(3000);

//express源码
exports = module.exports = createApplication;
/**
 * Create an express application.
 *
 * @return {Function}
 * @api public
 */

function createApplication() {//一个工厂函数
  const app = function(req, res, next) {//所有请求都会来这报到
    app.handle(req, res, next);//处理请求对应的回调函数
  };

  mixin(app, EventEmitter.prototype, false);//混入了node http模块的一些属性
  mixin(app, proto, false);//混入了application的一些属性

  // expose the prototype that will get set on requests
  app.request = Object.create(req, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  // expose the prototype that will get set on responses
  app.response = Object.create(res, {
    app: { configurable: true, enumerable: true, writable: true, value: app }
  })

  app.init();//做了一些初始化操作，如设置jsonp回调函数的key,返回子域名的默认偏移量
  return app;
}
```

### Application
#### app.set(name,value) *

```javascript
app.set('view engine', 'ejs');
```
#### app.get(name) *
```javascript
app.get('view engine'); // => ejs
```
#### app.enable(name)
将设置项name的值设置为true
```javascript
app.enable('trust proxy');
app.get('trust proxy');
// => true
```
#### app.disable(name)
将设置项name的值设置为false
```javascript
app.disable('trust proxy');
app.get('trust proxy');
// => false
```
#### app.enabled(name)
检查设置项 name 是否已启用
```javascript
app.enabled('trust proxy');// => false
app.enable('trust proxy');
app.enabled('trust proxy');// => true
```
#### app.disabled(name)
检查设置项 name 是否已禁用
```javascript
app.disabled('trust proxy'); // => false
app.disable('trust proxy');
app.disabled('trust proxy'); // => true
```
#### app.use([path], function) *
设置中间件(实际上就是一个函数，没有什么特殊的...)function，path为可选参数，默认为/，当不指定path，中间件会在所有请求前使用，当指定path时，中间件只在特定路由下使用，path不支持正则。
```javascript
app.use(function(req,res,next){
  console.log('%s %s',req,method, req.url);
  next();//如果不调用next函数，后面的操作不会执行
})
app.use('/static', express.static(__dirname +'public')) // http://localhost:3000/static/test.png 会到public目录下寻找对应的资源 __driname 当前模块所属目录的绝对路径，__filename 当前模块的绝对路径
app.use('/user',handlerUser); // /user, /user/getname --> 都匹配
```
#### settings
env 运行时环境，默认为 process.env.NODE_ENV 或者 "development"
trust proxy 激活反向代理，默认未激活状态
jsonp callback name 修改默认?callback=的jsonp回调的名字
json replacer JSON replacer 替换时的回调, 默认为null
json spaces JSON 响应的空格数量，开发环境下是2 , 生产环境是0
case sensitive routing 路由的大小写敏感, 默认是关闭状态， "/Foo" 和"/foo" 是一样的
strict routing 路由的严格格式, 默认情况下 "/foo" 和 "/foo/" 是被同样对待的
view cache 模板缓存，在生产环境中是默认开启的
view engine 模板引擎
views 模板的目录, 默认是"process.cwd() + ./views"
#### app.engine(ext,callback) *
注册模板引擎
```javascript
app.engine('html', require('ejs').renderFile);
app.set('view engine', 'html');//自定义一个html模板引擎，实际上还是使用ejs模板引擎来处理的 视图文件的后缀名就可以写成xxx.html了
```
#### app.param(name, callback)*
给路由中的参数添加回调触发器
```javascript
//http://localhost:3000/user/123
app.param('user_id',function(req,res,next,user_id){//router.param的回调 比router.get的回调先执行
  console.log(user_id));// --> 123
})
app.get('/user/:user_id',function(req,res){
  console.log(req.params.user_id);// --> 123
})
```
#### app.all(path,[callback...],callback) *
设置路由，处理请求和响应，path用来指定路径，并支持正则
```javascript
app.all('/user', handlerUser)//严格匹配/user, /user/getname --> 不匹配
app.all('/user/*', handlerUser)// /user, /user/getname --> 都匹配
```
#### app.all 和 app.use区别
1.作用不同，app.all是设置路由，用来处理请求和响应，app.use是设置中间件，一般是不处理请求和响应（其实也可以处理）</br>
2.匹配路径不同（前提:app.all不使用正则），app.all的路径是严格匹配的，app.use是查看路由是否以指定的路径开头的</br>
2.path参数支持正则不同，app.all path参数支持正则，app.use path参数不支持正则
#### app.locals
app.locals是一个应用程序级的变量，在js文件设置app.locals的值，然后在视图文件中使用即可
```javascript 
//js文件
app.locals.username = 'test';

//视图文件
<%= locals.username %>
```
#### app.render(view, [options]， callback)
app.render是用来渲染视图的，view是视图名，options是传给视图的数据，callback可以处理渲染后的模板字符串
#### app.listen() *
在给定的主机和端口地址上监听
```javascript
app.listen(3000)
//或者这样写
const http = require('http');
const https = require('https');
http.createServer(app).listen(80);
https.createServer(app).listen(443);

//express源码
app.listen = function listen() {
  const server = http.createServer(this);
  return server.listen.apply(server, arguments);
};
```
#### app.METHOD() *
处理http请求的方法，有app.get()、app.delete()、app.post()、app.patch、app.patch()等
### request
#### req.params *
获取请求路径上的参数，注：req.parms只能获取请求url上的参数，url ? 号后面的参数无法使用req.parms获取。
```javascript
router.get('/user/:username',function(req, rs){
  console.log(req.params.username); // GET /user/dxw --> dxw
})
```
#### req.query *
获取url ?号（get方式）后面的参数
```javascript
router.get('/user',function(req, res){
  console.log(req.query.username); // GET /user?username=dxw --> dxw
})
```
#### req.body *
获取请求体中的数据，需要使用bodyParser()中间件才能使用req.body来获取请求中的数据
```javascript
const bodyParser = require('body-parser');
const express = require('express');
const app = express();
const router = express.Router();
app.use(bodyParser.json()); // 处理编码格式为application/json
app.use(bodyParser.urlencoded({ extended: false }));//处理编码格式为application/x-www-form-urlencoded extended为true使用qs模块解析参数，qs模板可以解析嵌套数据，extended为false时使用querystring模板解析参数，querystring模板只能解析单层数据
router.post('/login', function(req,res){
  console.log('%s %s',req.body.username,req.body.password);//POST /login username=dxw password=123 --> dxw 123
})
```
#### req.param(name)
获取name参数的值，实际上也是查找 req.params、req.query、req.body对象</br>
查找顺序</br>
1. req.params
2. req.body
3. req.query</br>
```javascript
//epxress req.param()源码
req.param = function param(name, defaultValue) {
  const params = this.params || {};
  const body = this.body || {};
  const query = this.query || {};

  const args = arguments.length === 1
    ? 'name'
    : 'name, default';
  deprecate('req.param(' + args + '): Use req.params, req.body, or req.query instead');

  if (null != params[name] && params.hasOwnProperty(name)) return params[name];
  if (null != body[name]) return body[name];
  if (null != query[name]) return query[name];

  return defaultValue;
};
```
   
注：req.param在express 4.x中已弃用。
#### req.route
获取请求路由信息。
```javascript
app.get('/user/:id?', function userIdHandler (req, res) {
  console.log(req.route)
  res.send('GET')
})
// req.route对象结构
{ path: '/user/:id?',
  stack:
   [ { handle: [Function: userIdHandler],
       name: 'userIdHandler',
       params: undefined,
       path: undefined,
       keys: [],
       regexp: /^\/?$/i,
       method: 'get' } ],
  methods: { get: true } }
```
#### req.get(field) *
获取请求头中的值
```javascript
console.log(req.get('Content-Type')); // --> "text/plain"
```
#### req.accepts(types)
根据http请求的accept 字段，判断types是否可以接受，匹配成功返回types，不成功返回false，实际上就是字符串的匹配操作
```javascript
// Accept: text/html
req.accepts('html') // html
```
#### req.is(type)
根据http请求的Content-Type字段，判断type是否和Content-Type字段值匹配，匹配成功则返回type，不成功返回false
#### req.ip *
获取ip
```javascript
console.log(req.ip); // --> 127.0.0.1
```
#### req.path *
获取请求路径
```javascript
console.log(req.path); // http:localhost:8080/user/login --> /user/login
```
#### req.hostname *
获取ip
```javascript
console.log(req.hostname); // --> localhost
```
#### req.xhr
判断请求是否为由XMLHttpRequest发的请求，如果是返回true，否则返回false
#### req.protocol 
获取请求协议
```javascript
console.log(req.protocol); // --> http
```
#### req.subdomains
返回一个子域名数组，例如"tobi.ferrets.example.com" 返回 ['ferrets', 'tobi']
```javascript
//express req.subdomains 源码
defineGetter(req, 'subdomains', function subdomains() {
  const hostname = this.hostname;

  if (!hostname) return [];

  const offset = this.app.get('subdomain offset');//默认为2，可以进行设置
  const subdomains = !isIP(hostname)
    ? hostname.split('.').reverse()
    : [hostname];

  return subdomains.slice(offset);
});

```
#### req.originalUrl *
获取请求路径
```javascript
//localhost:3000/user/login
console.log(req.originalUrl) // --> /user/login
app.use('/admin', function (req, res, next) { // GET 'http://www.example.com/admin/new'
  console.dir(req.originalUrl) // '/admin/new'
  console.dir(req.baseUrl) // '/admin'
  console.dir(req.path) // '/new' req.path在中间件无法获取挂载点也就是use的第一个参数。
  next()
})
```
#### req.acceptsCharset(charset)
返回接受的字符集，如果传入的charset在Accept-Charset请求头字段里面没有则返回false
#### req.acceptsLanguage(lang)
返回接受的语言，如果传入的lang在Accept-Language请求头字段里面没有则返回false
#### req.acceptsEncodings(encoding)
返回接受的编码格式，如果传入的encoding在Accept-Encoding请求头字段里面没有则返回false
### response
#### res.status(code) *
设置http请求状态
```javascript
res.status(200);
res.status(200).send('操作成功');
res.status(404).sendfile('404.png');
```
#### res.set(field,value) *
设置响应头中信息
```javascript
res.set('Content-Type','application/json');
```
#### res.get(field) *
获取http响应头中的信息
```javascript
res.get('Content-Type'); // --> "text/plain"
```
#### res.cookie(name, value, [options])
设置cookie
```javascript
res.cookie('rememberme', '1', { maxAge: 900000, httpOnly: true })
res.cookie('rememberme', '1', { expires: new Date(Date.now() + 900000), httpOnly: true }
```
#### res.clearCookie(name, [options])
清除cookie，传人cookie的name
#### res.redirect([status], url) *
res.redirect是用来对请求进行重定向
```javascript
res.redirect('/login');
res.redirect(301,'/login');
```
#### res.location(path) *
设置http响应的location字段
```javascript
res.location('/login');
```
#### res.send([body]) *
res.send是对请求进行响应的，可以发送字符串，对象，数组，Buffer对象
```javascript
res.send('hello');
res.send('</b>hello</b>');
res.send({name:'test'});
res.send([1,2,3]);
res.send(Buffer.from('hello'));
```
#### res.json([body]) *
res.json会使用JSON.stringify()把响应数据转化为json格式的字符串。
```javascript
res.json({name:'test'});
res.status(500).json({error: '服务器内部错误'});
```
#### res.jsonp([body]) *
res.jsonp使用res.json返回数据，并支持跨域访问
```javascript
// http://localhost:3000
app.get('/jsonp', function(req, res){
  console.log('test jsonp');
  res.jsonp({msg: 'hello'});
})

// http://localhost:8080
$.getJSON("http://localhost:3000/jsonp?callback=?", function(data) {//jquery 方式
        console.log(data.msg); 
  });
function handleData(data){
  console.log(data.msg);
}
<script src="http://localhost:3000/jsonp?callback=handleData"></script>//原生方式
```
#### res.type(type)
设置响应头中的Content-Type字段。
```javascript
res.type('.html')
// => 'text/html'
res.type('html')
// => 'text/html'
res.type('json')
// => 'application/json'
res.type('application/json')
// => 'application/json'
res.type('png')
// => 'image/png'
```
#### res.format(object)
根据请求头中的Accept字段进行匹配，如果匹配成功则执行对应的回调
```javascript
res.format({
  'text/plain': function () {
    res.send('hey1')
  },

  'text/html': function () {
    res.send('<p>hey</p>')
  },

  'application/json': function () {
    res.send({ message: 'hey' })
  },

  default: function () {
    // log the request and respond with 406
    res.status(406).send('Not Acceptable')
  }
})
```
#### res.attachment([filename]) *
设置响应头的Content-Disposition字段,如果提供filename会设置Content-Disposition字段，并基于res.type()来设置Content-Type字段
```javascript
res.attachment()
// Content-Disposition: attachment

res.attachment('path/to/logo.png')
// Content-Disposition: attachment; filename="logo.png"
// Content-Type: image/png
```
#### res.sendfile(path, [options], [fn]]) *
res.sendfile用来传输文件，path是文件的路径名，options是一些设置，可以设置文件的根路径，设置http headers等，fn是一个回调，发送文件成功或失败时回调。
```javascript
app.get('/getfile1',function(req,res){
  res.sendfile('./test.png',function(err){
    if (err) {
      console.log(err);
    } else {
      console.log('发送成功');
    }
  })
})
app.get('/getfile2',function(req,res){
  res.sendfile(
    req.query.filename,
    {root: path.join(__dirname, 'public')},
    function(err){
      if (err) {
        console.log(err);
      } else {
        console.log('发送成功');
      }
  })
})
```
#### res.download(path, [filename], [fn]) *
res.download是用来下载文件的，传输文件还是使用res.sendfile进行传输，res.download和res.sendfile的区别是：res.download在浏览器中会下载文件，res.sendfile会把发送过来的文件在浏览器中预览打开
```javascript
res.download('./test.png',function(err){
  if (err) {
    console.log(err);
  } else {
    console.log('发送成功');
   }
});
```
#### res.links(links)
设置http响应头的link字段,链接其他页面
```javascript
res.links({
  next: 'http://api.example.com/users?page=2',
  last: 'http://api.example.com/users?page=5'
})
Link: <http://api.example.com/users?page=2>; rel="next",
      <http://api.example.com/users?page=5>; rel="last"
```

#### res.locals *
res.locals也是用来存储一些数据，然后在页面上使用，res.locals和app.locals的区别是：app.locals存储的数据在整个应用程序中都可以使用，res.locals存储的数据只能在一次请求中使用。
```javascript
//user.js
app.get('/user',function(req,res){
  res.locals.msg='hello';
  res.render('index');
})
//index.ejs
<b><%= locals.msg %></b> // --> hello
//login.ejs
<b><%= locals.msg %></b> // --> 页面上什么都没有
```
res.locals和app.locals在前后端分离项目中应该不会用到，在前后端不分离项目中用的应该很多。
#### res.render(view, [locals], callback) *
渲染视图的方法，view是视图名称，locals是可选参数，是传入页面的数据，和res.locals差不多，callback中可以对渲染后的模板字符串进行操作。
```javascript
//user.js
app.get('/user',function(req,res){
  res.render('index', {msg: 'hello'}, function(err,html){
    console.log(html); // --> <b>hello</b>
  });
})
//index.ejs
<b><%= locals.msg %></b> // --> hello
```
### Router
使用express.Router()来创建路由
#### router.all(path, callback)
类似app.all()，可以用来处理请求，path参数支持正则
#### router.METHOD() *
router.METHOD也是处理请求和响应的，和app.METHOD()差不多，如果要对路由进行拆分一般使用的就是router.METHOD()来处理请求和响应，如果不对路由进行拆分使用app.METHOD()就可以了
#### router.param(name, callback) *
给路由中的参数添加回调触发器，类似上面的app.param()
```javascript
//http://localhost:3000/user/123
router.param('user_id',function(req,res,next,user_id){//router.param的回调 比router.get的回调先执行
  console.log(user_id));// --> 123
})
router.get('/user/:user_id',function(req,res){
  console.log(req.params.user_id);// --> 123
})
```
#### router.route(path) *
返回一个单个路由的实例
```javascript
//一个rest api的例子
router.route('/users/:user_id')
  .all(function (req, res, next) {//一个特定路由的中间件可以这么认为（用来对所有请求做共通处理）
    next()
  })
  .post(function (req, res, next) {//增 C
    //operation
  })
  .delete(function (req, res, next) {//删 D
    //operation
  )}
  .put(function (req, res, next) {//改　U
    //operation
  })
  .get(function (req, res, next) {//查 R
    //operation
  })
```
#### router.use(path, callback)
类似app.use()，用来注册中间件。