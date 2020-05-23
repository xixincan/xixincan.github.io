---
title: 解决国内Homebrew安装太慢问题
date: 2020-05-23 16:54:52
tags: 
- Homebrew
- MacOS
categories: MacOS
keywords:
- Homebrew
- MacOS
cover: https://i.loli.net/2020/05/23/F1TcbNwvr73HMA8.jpg
---
# 介绍
Homebrew是一款Mac OS平台下的软件包管理工具，拥有安装、卸载、更新、查看、搜索等很多实用的功能。简单的一条指令，就可以实现包管理，而不用你关心各种依赖和文件路径的情况，十分方便快捷。

本文主要解决问题：Homebrew常规安装太慢；以及通过brew install安装软件太慢，还有时不时的自动updating巨耗时的问题。

首先安利官网：https://brew.sh/index_zh-cn

官网安装命令：
```shell
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"
```

官网卸载命令：
```shell
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"
```
# 问题
国内官网安装基本很慢，速度不忍直视，5KB/s......；这速度怎么对得起科学上网？

一开始大概是这个样子：
```shell
==> This script will install:
 /usr/local/bin/brew
 /usr/local/share/doc/homebrew
 /usr/local/share/man/man1/brew.1
 /usr/local/share/zsh/site-functions/_brew
 /usr/local/etc/bash_completion.d/brew
 /usr/local/Homebrew
 Press RETURN to continue or any other key to abort
==> Downloading and installing Homebrew...
```

大部分情况是安装了一会就报错，然后是这个样子：
```shell
curl: (7) Failed to connect to raw.githubusercontent.com port 443: Connection refused
```
抓狂😫……

# 解决
那我们就不能愉快地安装HomeBrew了吗？(试了一下，卸载倒是挺快的……WTF？？？)

简单五步，搞定HomeBrew安装。

## 第一步：创建HomeBrew文件夹

首先确保/usr/local/Homebrew文件夹不存在，存在的话删除，然后执行：
```shell
sudo mkdir /usr/local/Homebrew
```

## 第二步：git克隆
```shell
sudo git clone https://mirrors.ustc.edu.cn/brew.git /usr/local/Homebrew
```
 或者 
```shell
sudo git clone https://mirrors.aliyun.com/homebrew/brew.git /usr/local/Homebrew
```
 或者 
```shell
sudo git clone https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git /usr/local/Homebrew
```
回车后，会提示Receiving objects: xx% 等待下载完成。
```shell
Cloning into '/usr/local/Homebrew'... 
remote: Counting objects: 132526, done. 
remote: Total 132526 (delta 0), reused 0 (delta 0) 
Receiving objects: 100% (132526/132526), 32.16 MiB | 1.09 MiB/s, done. 
Resolving deltas: 100% (97548/97548), done.
```
## 第三步：创建一个快捷方式到/usr/local/bin目录
```shell
sudo ln -s /usr/local/Homebrew/bin/brew /usr/local/bin/brew
```
如果提示File exists表示/usr/local/bin文件夹里面已经有brew，删除后再运行第三步。

## 第四步：创建core文件夹 并 再次git克隆
```shell
sudo mkdir -p /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core
```
以及
```shell
sudo git clone https://mirrors.ustc.edu.cn/homebrew-core.git /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core
```
 或者 
```shell
sudo git clone https://mirrors.aliyun.com/homebrew/homebrew-core.git /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core
```
 或者 
```shell
sudo git clone https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git /usr/local/Homebrew/Library/Taps/homebrew/homebrew-core
```
完成后有如下信息输出：
```shell
Cloning into '/usr/local/Homebrew/Library/Taps/homebrew/homebrew-core'... 
remote: Counting objects: 688626, done. 
remote: Total 688626 (delta 0), reused 0 (delta 0) 
Receiving objects: 100% (688626/688626), 223.64 MiB | 6.83 MiB/s, done. 
Resolving deltas: 100% (455339/455339), done.
```

## 第五步：获取权限 并 运行更新
```shell
sudo chown -R $(whoami) /usr/local/Homebrew
```
以及
```shell
brew update
```
稍等一会儿～大功告成！

## 附注
最后设置：设置环境变量，再运行下面两句后，重启终端：（命令中的链接地址可以替换为第二步或者第四步中对应的链接地址）
```shell
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.zshrc 

echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.bash_profile
```
brew有一个自检程序，如果有问题自检试试：
```shell
brew doctor
```
查看全部安装路径
```shell
brew list
```
查看指定软件安装路径
```shell
brew list 软件名
```
另外如果已经用官网的命令成功安装好了Homebrew的童鞋（好吧果然你们有耐心。。。），可以通过替换镜像源来解决安装软件慢以及更新慢的问题：

**当然如果是通过本文介绍的安装方法是不用替换的**

替换brew.git:（命令中的链接地址可以替换为第二步中对应的链接地址）
```shell
cd "$(brew --repo)"

git remote set-url origin https://mirrors.ustc.edu.cn/brew.git
```
替换homebrew-core.git:（命令中的链接地址可以替换为第四步中对应的链接地址）
```shell
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"

git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-core.git
```
如果想要重置回默认的源：

重置brew.git:
```shell
cd "$(brew --repo)"

git remote set-url origin https://github.com/Homebrew/brew.git
```
重置homebrew-core.git:
```shell
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"

git remote set-url origin https://github.com/Homebrew/homebrew-core.git
```

