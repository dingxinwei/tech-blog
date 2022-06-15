# 发布一个npm包
如何发布一个npm包，在日常开发中我们有可能需要把自己开发的一些类库，cli工具或者组件库发布出去给其他人使用

## 注册一个npm账号
需要在npm上注册一个账号，可以在[官网](https://www.npmjs.com/)上进行注册，也可以通过在命令行中执行npm adduser来创建一个账号,在执行npm adduesr之前需要把npm的镜像源切回官方镜像源
```bash
npm config get registry
npm config set registry https://registry.npmjs.org/ 
```

## 登录账号
注册完账号后，在命令行输入npm loing，然后填写用户名，密码，邮箱，输入邮箱后会发送一个一次性的密码填写上去即可

## 发布
进入到要发布的项目的路径下，执行npm publish即可发布

## 踩坑
1. 如果在登录的时候没有切回官方镜像源会报错，
2. 如果发布的包名是@<scope>/xxx这种的，可能发布失败，这种格式的包名npm会认为是一个私有包，私有包发布是需要收费的，我们可以在发布的命令后面添加一个access参数即可
3. 包名重复也可能发布失败，这时候需要修改一下package.json里面的name字段
```bash
npm publish --access public

```