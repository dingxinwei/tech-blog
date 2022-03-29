# express获取请求参数方式
## req.params
   获取url路径上的参数，使用方式/user/123456 --> /user/:user --> req.params.user
## req.body
获取post请求方式的请求数据，需要借助body-parser模块</br>
app.use(bodyparser.urlencoded({ extended: false }));
处理content-type为application/x-www-form-urlencoded的请求数据</br>
app.use(bodyparser.json());处理content-type为application/json的请求数据
### 两种请求数据编码格式：
#### application/json
例如'{name:test,age:18}' 请求数据以json字符串的形式存在
#### application/x-www-form-urlencoded 
例如 name=test&age=18 请求数据以键值对的形式存在
## req.param(键名) 
express4.11以上版本已废弃
## req.query
获取url上的请求参数例如：/user?username=test&password=123
使用方式：req.query.username