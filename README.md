# Chronos
Chronos是古希腊神话中的第二代众神之王，亦为时间之神。所以我把Chronos作为我的个人博客系统的名字。

# 简介
Chronos基于开源博客框架[Hexo](https://hexo.io/zh-cn/)搭建，博客文章使用Markdown编辑发布，Hexo这里就不再介绍，简单介绍一下本项目的准备工作。
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
