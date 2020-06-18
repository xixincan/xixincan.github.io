# Chronos
Chronos是古希腊神话中的第二代众神之王，亦为时间之神。所以我把Chronos作为我的个人博客系统的名字。

# 简介
Chronos基于开源博客框架[Hexo](https://hexo.io/zh-cn/)搭建，博客文章使用Markdown编辑发布，并引入Travis CI进行持续集成服务，免去了每次更新博客后手动发布与部署的操作。Hexo这里就不再介绍，简单介绍一下本项目的准备工作。
下载本项目到本地后，需要安装Hexo运行环境。

# 安装Hexo
安装Hexo之前需要提前准备一些环境，当然如过你的电脑已经有了相关环境可以直接跳过准备阶段，直接进行Hexo安装。

## 依赖环境
- Git
- Node.js

## 安装Hexo
使用 npm 安装 Hexo。
```
 npm install -g hexo-cli
```
## 进阶安装和使用
对于熟悉 npm 的进阶用户，可以仅局部安装 hexo 包。
```
 npm install hexo
```
安装以后，可以使用以下两种方式执行 Hexo：
```
 npx hexo <command>
```
将 Hexo 所在的目录下的 node_modules 添加到环境变量之中即可直接使用 hexo <command>：
```
echo 'PATH="$PATH:./node_modules/.bin"' >> ~/.profile
```
## 附：Hexo命令
清除
```
hexo clean
```
新建文章
```
hexo n 文章title
```
生成静态文件
```
hexo g
```
本地部署运行(localhost:4000访问)
```
hexo s
```
发布
```
hexo d
```

## 使用Travis CI
Travis CI 只支持 Github，不支持其他代码托管服务。首先，访问官方网站 [travis-ci.org](https://travis-ci.org/)，点击右上角的个人头像，使用 Github 账户登入 Travis CI。

Travis 会列出 Github 上面你的所有仓库，以及你所属于的组织。此时，选择你需要 Travis 帮你构建的仓库，打开仓库旁边的开关。一旦激活了一个仓库，Travis 会监听这个仓库的所有变化。

在项目的根目录下面添加一个.travis.yml文件指定 Travis 的行为。该文件必须保存在 Github 仓库里面，一旦代码仓库有新的 Commit，Travis 就会去找这个文件，执行里面的命令。

示例：
```yaml
language: node_js
node_js: stable

# S: Build Lifecycle
install:
  - npm install


  #before_script:
  # - npm install -g gulp

script:
  - hexo g

after_script:
  - cd ./public
  - git init
  - git config user.name "xxc"
  - git config user.email "xxc@163.com"
  - git add .
  - git commit -m "Auto publish via Travis CI"
  - git push --force --quiet "https://${branchname}@${GH_REF}" master:master
# E: Build LifeCycle

branches:
  only:
    - branchname
env:
  global:
    - GH_REF: github.com/xxc/xxc.github.io.git

```
