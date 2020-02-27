
# 一、Mongodb简介及说明
## 1.1简介 
MongoDB是一个基于分布式文件存储的开源系统，将数据存储为类似JSON的文档，在爬虫方面作为数据存储服务应用广泛，笔者在这里就MongoDB应用写一份简明文档，以供读者快速搭建使用MongoDB
## 1.2文档说明
### 1.2.1占位符
假设MongoDB在/home/mongodb目录下，为了书写和表示方便我们用{MongodbFileName}表示MongoDB服务所在路径，故大括号均表示占位符，阅读者切记替换成自己相应的文件或者文件目录
### 1.2.2代码块
假设代码是Linux命令，则会在前面加上相应的注释，例如：
```
//Linux
mkdir mongodb
```
假设是js代码，则：
```
//node.js
console.log("从单机到副本集到分片到机房着火");
```
### 1.2.3注释
除1.2.2外，其余均为注释，例如：
```
//node.js
//print log
console.log("从单机到副本集到分片到机房着火");
```
# 二、部署使用单例MongoDB
## 2.1下载tar包并安装
### 2.1.1下载
前往[MongoDB官网](https://www.mongodb.com/)下载免费版tar包，笔者此处以MongoDB V3.4为例
### 2.1.2解压安装
```
//Linux
tar zxf {MongoDB File Name}
//create file 
mkdir mongodb
cd mongodb
//data
mkdir data
//log
mkdir log
```
找到或者创建mongodb.conf文件（文件配置见2.3章节），在bin下启动MongoDB服务
```
./mongod --config {mongodb}/mongodb.conf
```
正常情况下，服务启动后会出现如下提示：
```
//Linux
forked process:{port}
child process started successfully,parent exiting
```
如果出现错误，请检查配置文件和查看log目录下的日志
## 2.3配置文件简介和使用
### 2.3.1配置文件Demo
```
//mongodb.conf
datapath = {MongoDB DataPath}
logpath = {MongoDB Log File}
port = {port}
journal = true
directoryperdb = true
```
需要注意datapath是具体到目录，logpath具体到文件，例如：
```
datapath = /home/mongodb/data
logpath = /home/mongodb/log/logs
```
### 2.3.2使用
> `mongodb://{IP}:{Port}`

例如：
`mongodb://127.0.0.1:27017`
# 三、部署使用MongoDB副本集
## 3.1副本集基础概念
### 3.1.1副本集概念
将数据冗余备份，在多个服务器上存储数据副本，提高了数据的可用性，保证数据的安全性
### 3.1.2节点介绍
副本集包括主节点、从节点、选举节点，所有节点保持奇数以保证部分节点挂掉时，能选出备用节点
## 3.2配置与使用
### 3.2.1mongodb.conf配置
```
//mongodb.conf
datapath = {MongoDB DataPath}
logpath = {MongoDB Log File}
port = {port}
journal = true
directoryperdb = true
replSet = {Name}
```
在单例的基础上，添加replSet配置，为副本集设定名称，假设在服务器A、B、C使用同样的配置，启动服务（启动方式和单例类似）
### 3.2.2副本集
使用mongodb客户端
```
//linux
cd {mongodb service}/bin
// mongodb client
./mongo
```
配置副本集
```
//mongodb
//初始化
rs.initiate({
            "_id" : "{Name}",
            "members" : [
                    {
                            "_id" : 0,
                            "host" : "{Host A}:{port}",
                             priority:1
                    },
                    {
                            "_id" : 1,
                            "host" : "{Host B}:{port}",
                             priority:2
                    },
                    {
                            "_id" : 2,
                            "host" : "{Host C}:{port}",
                             arbiterOnly:true
                    }
            ]
})
//添加新的从节点
rs.add({Host IP}:{port})
//添加仲裁节点
rs.addArb({Host IP}:{port})
//查看副本集情况
re.status()
//查看副本集配置
re.conf()
```
此处配置priority为优先级，arbiterOnly为选举节点
### 3.2.3使用
> `mongodb://{host}:{port},{host2}:{port}/?connect=replicaSet;replicaSet={Name};connectTimeoutMS=300000`

例如：
`mongodb://localhost:27017,localhost:27888/?connect=replicaSet;replicaSet=Test;connectTimeoutMS=300000`
# 四、MongoDB权限验证
## 4.1创建用户
一般情况下，mongodb都部署到授信网络环境下，如果服务处于外网，可以开启安全认证
```
//mongodb client
//切换到admin集合
use admin;
//create User
db.createUser({user:"TestUser",pwd:"123456",roles:[{role:"readWriteAnyDatabase",db:"admin"},{role:"dbAdminAnyDatabase",db:"admin"}]});
//query User list
db.getUsers();
```
## 4.2修改MongoDb配置和使用安全连接
加上auth配置
```
//mongodb.conf
auth = true
```
重启服务
## 4.3使用
> `mongodb://{username}:{password}@{host}:{port}`

例如：
`mongodb://TestUser:123456@127.0.0.1:27017`

此处的role请参考[官方文档](https://docs.mongodb.com/)
# 五、机房着火参考
> 火警电话：119

在做好副本集的情况下，如果单一机房发生着火，请报火警，然后从未着火的服务器机房查看MongoDB运行情况
```
//mongodb client
//查看副本集情况
re.status()
```
# 六、预告
MongoDb分片及应用