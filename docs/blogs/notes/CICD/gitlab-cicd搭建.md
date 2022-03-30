# gitlab cicd
## CICD介绍
CICD指的持续集成（Continuous Integration）和持续交付（Continuous Delivery），简写为CICD，持续集成（CI）指的是开发人员向特性分支提交代码，执行构建和单元测试，通过测试后合并到主分支的过程，主要强调的分支代码的提交，构建及单元测试。持续交付（CD）指的在持续集成的基础上，将代码部署到测试服务器上，通过QA测试后再把代码部署到生产服务器的过程，主要强调的是代码的部署。
## gitlab cicd介绍
在gitlab上使用cicd需要注册一个runner，这个runner一般是部署在一台单独的机器上用来执行自动化脚本，gitlab提供了一些免费的runner，不过使用这些runner需要绑定信用卡。我们也可以自己搭建runner
## runner安装
Ubuntu 
```bash
sudo apt-get install gitlab-runner
```
centos8
```bash
dnf install gitlab-runner
```
centos7
```bash
yum install gitlab-runner
```
## runner注册
启动runner
```bash
gitlab-runner start
```
执行 $REGISTRATION_TOKEN是在gitlab仓库settings runner Specific runners下面获取
```
sudo gitlab-runner register --url https://gitlab.com/ --registration-token $REGISTRATION_TOKEN
```
执行上面的注册runner命令后会出现一些QA式的问答，填写完毕即可
## runner配置
在项目根目录添加一个.gitlab.yml文件, yml文件的书写格式请自行百度</br>
配置文件内容
```
stages: // cicd的阶段，每个阶段会执行对应的job
 - build
 - deploy
 - notify

build-job: // job
 stage: build
 tags: // 与runner注册时对应
  - push
 script: // job里需要执行的脚本
  - echo `pwd`
  - npm install --registry=https://registry.npm.taobao.org 
  - npm run build
 cache: // 缓存文件
  paths:
   - node_modules/
   - dist/
 artifacts:
   paths:
    - dist/
 only: // 限制哪些文件可以触发cicd
   - main
deploy-job:
 stage: deploy 
 tags:
  - deploay
 before_script: // 预先执行的一些脚本
   - mkdir -p ~/.ssh
   - chmod 700 ~/.ssh
   - eval $(ssh-agent -s)
   - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa
   - chmod 600 ~/.ssh/id_rsa
   - ssh-add ~/.ssh/id_rsa
 script:
  - pwd
  - ls -l
  - mv dist vue-ci 
  - scp  -r vue-ci root@ip:/root/dist
 only:
   - main
notify-error-job: 部署阶段的job
 stage: notify
 tags:
  - bke
 script:
  - curl -H 'Content-Type:application/json' -X POST -d "{\"emailSubject\":\"oa系统后端部署通知\",\"emailContent\":\"<div>项目部署结果：<span style='color:red'>失败</span></div>项目名称：$CI_PROJECT_NAME\n</br>提交人：$CI_COMMIT_AUTHOR\"}" http://ip/email/note
 only:
  - main
 when: on_failure // 当前面一个job失败时执行这个job
notify-success-job:
 stage: notify
 tags:
  - bke
 script:
  - curl -H 'Content-Type:application/json' -X POST -d "{\"emailSubject\":\"oa系统后端部署通知\",\"emailContent\":\"<div>项目部署结果：<span style='color:green'>成功</span></div>项目名称：$CI_PROJECT_NAME\n</br>提交人：$CI_COMMIT_AUTHOR\"}" http://ip/email
 only:
  - main
 when: on_success  // 当前面一个job成功时执行这个job
```
这边主要是分了三个阶段（stage），一个构建打包，一个部署，一个通知是否部署成功。每一个阶段有一些job，在job里配置了一些脚本（script），每个阶段可以有多个job，gitlab-runner的最小执行单位是按照job来执行，按照书写顺序执行。</br>
通知阶段的一些任务用到了gitlab的一些变量，如CI_PROJECT_NAME CI_COMMIT_AUTHOR，具体请<a href="https://docs.gitlab.com/ee/ci/variables/predefined_variables.html">参考</a>
## 踩坑记录
runner服务器和代码部署的服务器是两台不同的服务器，由于我选择的脚本执行方式是ssh方式，需要在runner服务器上配置ssh免密登录，配置流程是把runner服务器上的ssh公钥复制一份放到代码部署服务器的ssh目录下的authorized_keys文件中