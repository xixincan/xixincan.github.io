---
title: Windows系统中配置MongoDB服务
date: 2020-05-20 15:32:56
cover: https://i.loli.net/2020/05/20/OtsCTa2hQ5cpowX.jpg
tags: 
- Windows
- NoSQL
- MongoDB
categories: MongoDB
keywords: 
- MongoDB
---
## MongoDB配置为Windows服务

### 第一种方式 

以管理员身份打开命令行，cd 到安装目录的 bin 文件夹下，执行以下命令：
```
mongod –dbpath E:\MongoDB\data\db –logpath E:\MongoDB\log\mongo.log –logappend –serviceName MongoDB –install 
```
其中,数据库路径为E:\MongoDB\data\db  
日志路径为E:\MongoDB\log\mongo.log  
服务名为MongoDB; （-auth表示需要登录验证）   
&emsp;&emsp;成功的话 cmd 会有提示已安装服务成功。另外可以在任务管理器的服务列表中查看。 运行 cmd 直接执行：
```bash
net start MongoDB
``` 
提示服务启动成功。 
```bash
net stop MongoDB 
```
用来关闭服务。


### 第二种方式
 
 创建文件conf/mongo.cfg
 ```uastcontextlanguage
 systemLog:
     destination: file
     path: D:\MongoDB\log\mongo.log
 storage:
     dbPath: D:\MongoDB\data\db
```
 &emsp;&emsp;将mongoDB所在的bin目录添加到环境变量PATH中
 &emsp;&emsp;以管理员身份打开命令行,  执行
 ```bash
 mongod.exe --config "D:\MongoDB\conf\mongo.cfg" --install
```
 &emsp;&emsp;要使用备用 dbpath，可以在配置文件（例如：C:\mongodb\mongod.cfg）或命令行中通过 --dbpath 选项指定。  
 &emsp;&emsp;如果需要，您可以安装 mongod.exe 或 mongos.exe 的多个实例的服务。只需要通过使用 --serviceName 和 --serviceDisplayName 指定不同的实例名。只有当存在足够的系统资源和系统的设计需要这么做。   
 启动MongoDB服务
 ```bash
net start MongoDB
```
 关闭MongoDB服务
 ```bash
net stop MongoDB
```
 移除 MongoDB 服务
 ```bash
mongod.exe --remove
```
 &emsp;&emsp;**命令行下运行 MongoDB 服务器 和 配置 MongoDB 服务 任选一个方式启动就可以。**