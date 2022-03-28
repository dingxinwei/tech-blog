# axios源码解析
从日常使用的角度分析源码</br>
## axios项目目录解析
```
├── /dist/                     # 项目输出目录
├── /lib/                      # 项目源码目录
│ ├── /cancel/                 # 定义取消功能
│ ├── /core/                   # 一些核心功能
│ │ ├── Axios.js               # axios的核心主类
│ │ ├── dispatchRequest.js     # 用来调用http请求适配器方法发送请求
│ │ ├── InterceptorManager.js  # 拦截器构造函数
│ │ └── settle.js              # 根据http响应状态，改变Promise的状态
│ ├── /helpers/                # 一些辅助方法
│ ├── /adapters/               # 定义请求的适配器 xhr、http
│ │ ├── http.js                # 实现http适配器
│ │ └── xhr.js                 # 实现xhr适配器
│ ├── axios.js                 # 对外暴露接口
│ ├── defaults.js              # 默认配置 
│ └── utils.js                 # 公用工具
├── package.json               # 项目信息
├── index.d.ts                 # 配置TypeScript的声明文件
└── index.js                   # 入口文件
```
##  使用axios发送一次请求的流程
1. 入口函数
2. Axios构造函数
3. 请求拦截器
4. dispatchRequest
5. 请求数据转换器
6. 请求适配器
7. 响应数据转换器
8. 响应拦截器
## axios发送请求的几种方式
```javascript
this.$axios({ url,method,data,headers})
this.$axios.get(url,{headers}})
this.$axios.post(url, data, {headers})
this.$axios.request({
  url,
  method,
  headers
})
```
为什么axios发送请求有这么多种使用方式，怎么实现的？</br>
## axios.js 入口函数
```javascript
function createInstance(defaultConfig) { // 入口函数
  var context = new Axios(defaultConfig); 
  var instance = bind(Axios.prototype.request, context); // bind返回的是一个函数在这个函数内会调用Axios原型上的request方法，context为调用request方法时绑定的this值
  
  utils.extend(instance, Axios.prototype, context); // 拷贝Axios原型对象上的属性，并给方法绑定this值

  utils.extend(instance, context); // 拷贝实例属性，如拦截器，默认配置

  instance.create = function create(instanceConfig) { // 对外提供创建新实例的方法，传入自定义配置
    return createInstance(mergeConfig(defaultConfig, instanceConfig));
  };

  return instance;
}
var axios = createInstance(defaults);
...
module.exports.default = axios;
```
## Axios.js 构造函数
```javascript
function Axios(instanceConfig) { // 构造函数
  this.defaults = instanceConfig;
  this.interceptors = {
    request: new InterceptorManager(),
    response: new InterceptorManager()
  };
}
Axios.prototype.request = function request(configOrUrl, config) { // 核心方法，axios发送请求都是执行的这个函数
  ...
};

utils.forEach(['delete', 'get', 'head', 'options'], function forEachMethodNoData(method) { //给Axios原型对象加入一些请求方法
  Axios.prototype[method] = function(url, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: (config || {}).data
    }));
  };
});
utils.forEach(['post', 'put', 'patch'], function forEachMethodWithData(method) { //给Axios原型对象加入一些请求方法
  Axios.prototype[method] = function(url, data, config) {
    return this.request(mergeConfig(config || {}, {
      method: method,
      url: url,
      data: data
    }));
  };
});

module.exports = Axios;
```
从这些源码中我们能看出来几种使用方式实际上都是在执行request函数</br>
## axios拦截器和数据转换器的使用
拦截器，数据转换器作为axios的一个特点，在日常开发中也是很常用，拦截器又分为请求拦截器和响应拦截器，在请求拦截器中可以设置token，响应拦截器中统一处理错误状态码，
数据转换器也分为请求数据转换器和响应数据转换器，请求数据转换器是在发起请求前转发请求数据，响应数据转换器是在获得响应后转换响应数据。
### 拦截器和拦截器使用方式
```javascript
axios.interceptors.request.use(config => {
 return config; 
}, error => { 
 return Promise.reject(error); 
})
axios.interceptors.response.use(response => {
 return response; 
}, error => {
  return Promise.reject(error); 
})
axios.defaults.transformRequest.push((data, headers) => {
  // ...处理data
  return data;
});
axios.defaults.transformResponse.push((data, headers) => {
  // ...处理data
  return data;
});
```
## request方法
```javascript
Axios.prototype.request = function request(configOrUrl, config) {
  
  ...
  
  while (requestInterceptorChain.length) { // 执行请求拦截器
    var onFulfilled = requestInterceptorChain.shift();
    var onRejected = requestInterceptorChain.shift();
    try {
      newConfig = onFulfilled(newConfig);
    } catch (error) {
      onRejected(error);
      break;
    }
  }

  try {
    promise = dispatchRequest(newConfig); // 发送请求
  } catch (error) {
    return Promise.reject(error);
  }

  while (responseInterceptorChain.length) { // 执行响应拦截器
    promise = promise.then(responseInterceptorChain.shift(), responseInterceptorChain.shift());
  }

  return promise;
};

```
处理请求的方法，发送请求前先转换数据，然后发送请求，得到响应后转换响应数据</br>
```javascript
function dispatchRequest(config) {
  ...
  // 转换请求数据
  config.data = transformData.call(
    config,
    config.data,
    config.headers,
    config.transformRequest
  );

 ...

  var adapter = config.adapter || defaults.adapter;// 如果没有提供适配器，则使用默认的适配器

  return adapter(config).then(function onAdapterResolution(response) {
    

    // 转换响应数据
    response.data = transformData.call(
      config,
      response.data,
      response.headers,
      config.transformResponse
    );

    return response;
  }, function onAdapterRejection(reason) {
   
      // 转换响应数据
      if (reason && reason.response) {
        reason.response.data = transformData.call(
          config,
          reason.response.data,
          reason.response.headers,
          config.transformResponse
        );
      }
    return Promise.reject(reason);
  });
};
```
## 根据当前环境进行适配
```javascript
function getDefaultAdapter() {
  var adapter;
  if (typeof XMLHttpRequest !== 'undefined') { // 浏览器环境
    
    adapter = require('../adapters/xhr');
  } else if (typeof process !== 'undefined' && Object.prototype.toString.call(process) === '[object process]') { // node环境
    
    adapter = require('../adapters/http');
  }
  return adapter;
}
```
## 为什么axios能在浏览器和node中使用
 axios对浏览器和node做了适配，在浏览器中使用XMLHttpRequest发送请求，在node使用http模块发送请求，xhr.js 里面适配了浏览器环境</br>
```javascript
module.exports = function xhrAdapter(config) {
  return new Promise(function dispatchXhrRequest(resolve, reject) {
   ...

    var request = new XMLHttpRequest();

    function onloadend() {
      if (!request) {
        return;
      }
      
      var responseHeaders = 'getAllResponseHeaders' in request ? parseHeaders(request.getAllResponseHeaders()) : null;
      var responseData = !responseType || responseType === 'text' ||  responseType === 'json' ?
        request.responseText : request.response;
      var response = {
        data: responseData,
        status: request.status,
        statusText: request.statusText,
        headers: responseHeaders,
        config: config,
        request: request
      };
      // settle函数主要校验状态码，校验通过调用_resolve，并把response当成参数传递过去，改变promise状态为fulfilled。不通过则调用_reject，改变promise状态为rejected
      settle(function _resolve(value) {
        resolve(value);
        done();
      }, function _reject(err) {
        reject(err);
        done();
      }, response);

      // Clean up request
      request = null;
    }

    if ('onloadend' in request) {
      // Use onloadend if available
      request.onloadend = onloadend;
    } 
    // buildURL会把params中的参数拼接在url后面
    request.open(config.method.toUpperCase(), buildURL(fullPath, config.params, config.paramsSerializer), true);

   ...   

    // 发送请求，requestData为请求体中携带的参数
    request.send(requestData);
  });
};

```
http.js 里面适配了node环境, node的适配比浏览器的复杂一些，其中涉及一些请求数据转换Buffer，设置http代理
```javascript
module.exports = function httpAdapter(config) {
  return new Promise(function dispatchHttpRequest(resolvePromise, rejectPromise) {
   
    var resolve = function resolve(value) {// 成功 返回响应数据
     
      resolvePromise(value);
    };
    
    var reject = function reject(value) { // 失败 返回错误信息
     
      rejected = true;
      rejectPromise(value);
    };
   
    ...  

    var options = {
      path: buildURL(parsed.path, config.params, config.paramsSerializer).replace(/^\?/, ''),
      method: config.method.toUpperCase(),
      headers: headers,
      agent: agent,
      agents: { http: config.httpAgent, https: config.httpsAgent },
      auth: auth
    };

    ... 
    var transport;
    
    if (config.transport) {
      transport = config.transport;
    } else if (config.maxRedirects === 0) {
      transport = isHttpsProxy ? https : http;
    } 

   
    // 发起请求并返回请求对象
    var req = transport.request(options, function handleResponse(res) {
      if (req.aborted) return;

      var stream = res;
      
      var lastRequest = res.req || req;
      
      var response = {
        status: res.statusCode,
        statusText: res.statusMessage,
        headers: res.headers,
        config: config,
        request: lastRequest
      }
      var responseBuffer = [];
      var totalResponseBytes = 0;
      stream.on('data', function handleStreamData(chunk) { // 获取响应数据
          responseBuffer.push(chunk);
          totalResponseBytes += chunk.length
        });
         stream.on('end', function handleStreamEnd() { // 响应数据获取结束执行
          try {
            var responseData = responseBuffer.length === 1 ? responseBuffer[0] : Buffer.concat(responseBuffer);
            if (config.responseType !== 'arraybuffer') {
              responseData = responseData.toString(config.responseEncoding);
              if (!config.responseEncoding || config.responseEncoding === 'utf8') {
                responseData = utils.stripBOM(responseData);
              }
            }
            response.data = responseData;
          } catch (err) {
            reject(AxiosError.from(err, null, config, response.request, response));
          }
          settle(resolve, reject, response); // 改变promise状态 并返回响应数据
        });
      }
    });

    
    if (utils.isStream(data)) {
      data.on('error', function handleStreamError(err) {
        reject(AxiosError.from(err, config, null, req));
      }).pipe(req);
    } else {
      req.end(data); // 把请求体数据写入请求并结束请求
    }
  });
};
```
## 其他的一些工具方法
utils.js 
```javascript
function forEach(obj, fn) { // 迭代器方法
  // Don't bother if no value provided
  if (obj === null || typeof obj === 'undefined') {
    return;
  }

  // Force an array if not already something iterable
  if (typeof obj !== 'object') {
    /*eslint no-param-reassign:0*/
    obj = [obj];
  }

  if (isArray(obj)) {
    // Iterate over array values
    for (var i = 0, l = obj.length; i < l; i++) {
      fn.call(null, obj[i], i, obj);
    }
  } else {
    // Iterate over object keys
    for (var key in obj) {
      if (Object.prototype.hasOwnProperty.call(obj, key)) {
        fn.call(null, obj[key], key, obj);
      }
    }
  }
}
```
helpers/bind.js
```javascript
module.exports = function bind(fn, thisArg) { // bind函数的实现
  return function wrap() {
    var args = new Array(arguments.length);
    for (var i = 0; i < args.length; i++) {
      args[i] = arguments[i];
    }
    return fn.apply(thisArg, args);
  };
};
```
## 总结
axios是一个比较好的开源库，其源码中使用一些设计模式，如工厂模式，迭代器模式，发布订阅模式（取消请求的功能实现中用到了），适配器模式，
还有类似异步中间件的处理请求响应的方式，就像koa的洋葱模型一样，一次请求会经过一系列的流程处理。axios拦截器和数据转换器的实现就是在这种处理请求的方式上建立的。
axios中的一些设计实现和思想值得学习和借鉴。