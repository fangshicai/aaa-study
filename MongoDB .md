// todo 待做



// 1.根据服务器内存和cpu的上限来设置队列

// 2.数据存储服务器异步处理，处理完成后rpc调用处理服务器接口修改队列状态。

// 3.设置mongoDB的配置，比如连接数





// 1.接收数据处理服务器关机，如何接受并发送完请求在关机

// 2.负载均衡如何引流到正常的服务器。





# ubuntu安装



## 1. 确保权限

/var/lib/mongodb

```shell
sudo mkdir -p /var/lib/mongodb
sudo chown -R mongodb:mongodb /var/lib/mongodb
sudo chmod 700 /var/lib/mongodb
sudo mkdir -p /var/log/mongodb
sudo chown -R mongodb:mongodb /var/log/mongodb
sudo chmod 755 /var/log/mongodb

```



## 2. 官网按照

按照官方网站，社区版MongoDB安装

https://www.mongodb.com/zh-cn/docs/manual/tutorial/install-mongodb-on-ubuntu/

默认端口：27017 



启动

```shell
sudo mongod --config /etc/mongod.conf
```



查看状态

```shell
sudo systemctl status mongod
```



停止

```shell
sudo systemctl stop mongod
```



## 3. 连接工具

mongosh 连接工具安装

https://www.mongodb.com/zh-cn/docs/mongodb-shell/install/



mongosh使用

https://www.mongodb.com/zh-cn/docs/mongodb-shell/connect/#std-label-mdb-shell-connect



MongoDB for VS Code 使用

1. 安装VS Code

2. VS Code安装插件以及使用

3. 输入用户名和密码

   ![image-20250815110021885](.\images\Snipaste_2025-08-15_11-04-19.png)





# 连接

## 1.允许远程访问

**默认只能本机使用**

```shell
vim /etc/mongod.conf
```

修改为

```shell
net:
  port: 27017
  bindIp: 0.0.0.0
```

重启

```shell
sudo mongod --config /etc/mongod.conf
```



开放阿里云安全组，开放服务器防火墙。

连接命令，可以用`mongosh、MongoDB Compass Download (GUI)、vsCode（插件MongoDB for VS Code）`工具进行连接

```shell
mongosh.exe mongodb:ip:27017
```

**修改配置文件后，其他主机都可以访问mangoDB数据库，且不需要认证（不安全），有风险。**



## 2.用户名密码访问

见安全性网站：https://www.mongodb.com/zh-cn/docs/manual/security/

> Salted 质询响应身份验证机制 (SCRAM) 是 MongoDB 的默认身份验证机制。
>
> 当用户对自己[进行身份验证](https://www.mongodb.com/zh-cn/docs/manual/tutorial/authenticate-a-user/#std-label-authentication-auth-as-user)时，MongoDB 会使用 SCRAM 针对用户的 [`name`](https://www.mongodb.com/zh-cn/docs/manual/reference/system-users-collection/#mongodb-data-admin.system.users.user)、[`password`](https://www.mongodb.com/zh-cn/docs/manual/reference/system-users-collection/#mongodb-data-admin.system.users.credentials) 和 [`authentication database`](https://www.mongodb.com/zh-cn/docs/manual/reference/system-users-collection/#mongodb-data-admin.system.users.db) 来验证所提供的用户凭证。



### 数据库被攻击

所有数据库莫名其妙没了，查看日志：

```shell
tail -n 50 /var/log/mongodb/mongod.log
```

输出

```shell
{"t":{"$date":"2025-08-27T11:30:34.351+08:00"},"s":"I",  "c":"COMMAND",  "id":20337,   "ctx":"conn258","msg":"dropDatabase - starting","attr":{"db":"READ__ME_TO_RECOVER_YOUR_DATA"}}
{"t":{"$date":"2025-08-27T11:30:34.351+08:00"},"s":"I",  "c":"COMMAND",  "id":20338,   "ctx":"conn258","msg":"dropDatabase - dropping collection","attr":{"db":"READ__ME_TO_RECOVER_YOUR_DATA","namespace":"READ__ME_TO_RECOVER_YOUR_DATA.README"}}
```

**因为不用账号和密码就可以连接，被攻击了....**



**READ__ME_TO_RECOVER_YOUR_DATA**

- 这是典型的 MongoDB 勒索病毒/挖矿木马攻击。常见行为：
  攻击者扫描公网暴露的 MongoDB（未开启认证、未做防火墙限制）。
- 删除原有数据库（这里 syd 也被 drop 了）。
- 新建一个 READ__ME_TO_RECOVER_YOUR_DATA 数据库/集合，提示你要“联系恢复数据”。
- 有时还会顺便删掉 local 数据库里的日志，降低你恢复的机会。



### 认证登录（SCRAM 官方默认）

[使用 SCRAM 对自管理部署上的客户端进行身份验证 - 数据库手册 - MongoDB Docs](https://www.mongodb.com/zh-cn/docs/manual/tutorial/configure-scram-client-authentication/#std-label-scram-client-authentication)



1.   创建用户admin

   在服务器上使用`mongosh`登录，默认是不需要密码的。
   
   ```shell
   mongosh 
   ```
   
   使用admin数据库，作为认证用的库
   
   ```shell
   use admin
   ```
   
   创建admin用户，**root为最高权限（权限很大）**
   
   ```shell
   db.createUser(
     {
       user: "admin",
       pwd: "syd123456", // or cleartext password
       roles: [
         { role: "root", db: "admin" }
       ]
     }
   )
   ```
   
   

2.   修改配置文件

```shell
vim /etc/mongod.conf

security:
  authorization: enabled
```

重启 MongoDB

```shell
sudo mongod --config /etc/mongod.conf
```

**注意：重启后只能用账号名密码访问，不然即使登录进去了也操作不了**



3. 只创建指定数据库权限的用户syd

使用admin登录

```shell
mongosh -u admin -p --authenticationDatabase admin
```

创建syd用户

```shell

use admin

db.createUser({
  user: "syd",
  pwd: "syd123456",
  roles: [ { role: "readWrite", db: "syd_app" } ]
})


```

退出后执行

```
sudo mongod --config /etc/mongod.conf
```



4. 连接

```shell
mongosh -u admin -p --authenticationDatabase admin
# 查看所有用户
use admin
db.getUsers()
```



## 3.生产环境

建议把`/etc/mongod.conf`修改为只允许本机连接，`security`注释掉。

```
# 只允许本机连接
net:
  port: 27017
  bindIp: 127.0.0.1
#security:
#  authorization: enabled  # 启用认证

```



# mongoDB数据库操作



这里使用**vsCode（插件MongoDB for VS Code）**进行操作

**注意**：使用`mongosh `工具进行操作时，`use("syd_app")`需要修改为`syd_app`，具体看官方数据库手册文档吧。



概念

- 数据库：如果数据库不存在，MongoDB 会在您首次向该数据库存储数据时创建该数据库。因此，可以切换到一个不存在的数据库。

- 集合：MongoDB 将文档存储在集合中。集合类似于关系数据库中的表。如果集合不存在，MongoDB 会在您首次存储该集合的数据时创建该集合。

- 文档：MongoDB 中的记录是一个文档，它是由字段和值对组成的数据结构。MongoDB 文档类似于 JSON 对象。字段值可以包含其他文档、数组和文档数组。

  



创建集合

> 需求：50 万台设备，1 天新增 1.44 亿数据，必须对表建立索引, 对 device_id 和create_timestamp 建立索引，提高查询性能.

- 需要创建索引，目前创建复合索引。

```java
// MongoDB Playground
// Use Ctrl+Space inside a snippet or a string literal to trigger completions.

// The current database to use.
use("syd_app");
db.dropDatabase();
const database = "syd_app";
//  Create a new database.
use(database);

// 创建集合（表），并插入一条数据，
// 如果文档未指定 _id 字段，MongoDB 会将具有 ObjectId 值的 _id 字段添加到新文档中。
db.device_data.insertOne({
  device_id: '34CDB06B20DE',
  payload:'1204000000640000000500000000000000000000000000000000000001F400000000000000000000000000000000000000000000000000000000000000000000000000002F800000000000000000040331A200000000000000000000000000000000000300000000000000000000000000000000000000000000000000000467000000FFFFFF00000000051C00000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000000000000000000000000019080C0F2C2C0E6A',
  create_timestamp: Long(1) //使用Long() 构造函数可用于显式指定 64 位整数。
});

// 创建复合索引 
// 根据前缀索引规则 
// device_id可用，device_id和create_timestamp可用，
// create_timestamp 不可用，若有高频的全局时间范围查询（如不指定 device_id 查全表数据），此时需额外创建
db.device_data.createIndex({ 
  device_id: 1,          // 升序（等值查询字段放前面）
  create_timestamp: -1   // 降序（时间范围+倒序排序）
})
   
// 删除所有文档
db.device_data.deleteMany({})
```



删除集合

```shell
use("syd_app");
db.dropDatabase();
```



插入一条文档

```javascript
// 创建集合（表），并插入一条数据，
// 如果文档未指定 _id 字段，MongoDB 会将具有 ObjectId 值的 _id 字段添加到新文档中。
db.device_data.insertOne({
  device_id: '34CDB06B20DE',
  payload:'1204000000640000000500000000000000000000000000000000000001F400000000000000000000000000000000000000000000000000000000000000000000000000002F800000000000000000040331A200000000000000000000000000000000000300000000000000000000000000000000000000000000000000000467000000FFFFFF00000000051C00000000000000000000000000000000008000000000000000000000000000000000000000000000000000000000000000000000000000000000000019080C0F2C2C0E6A',
  create_timestamp: Long(1) //使用Long() 构造函数可用于显式指定 64 位整数。
});
```



删除所有文档

```javascript
// 删除所有文档
db.device_data.deleteMany({})
```



统计个数

```
db.device_data.distinct("device_id").length
```



查看mongoDB状态

```javascript
use("syd_app");

db.device_data.stats()
```



重复查询

```javascript
use("syd_app");
db.dropDatabase();
const database = "syd_app";
use(database);

db.device_data.aggregate([
  {
    $group: {
      _id: { device_id: "$device_id", create_timestamp: "$create_timestamp" },
      count: { $sum: 1 },
      docs: { $push: "$_id" }  // 把所有重复文档的 _id 收集起来
    }
  },
  {
    $match: {
      count: { $gt: 1 } // 只要出现超过 1 次的
    }
  }
])

```



去重查询

```javascript
use("syd_app");
db.device_data.aggregate([
  { $group: { _id: "$device_id" } },
  { $count: "uniqueDeviceCount" }
])
```





# 磁盘空间以及内存

**统计磁盘空间**

详细解释见以下链接

https://www.mongodb.com/zh-cn/docs/manual/reference/method/db.collection.stats/

```
db.device_data.stats()
```

查询出的一部分结果如下所示

```shell
  "sharded": false,
  "size": 27673988000,// 逻辑数据大小  25.8 GB
  "count": 49684000,
  "numOrphanDocs": 0,
  "storageSize": 2825949184, // 数据存储空间 2.63 GB
  "totalIndexSize": 2641104896, // 索引存储空间 2.46 GB
  "totalSize": 5467054080, // 总磁盘空间占用 5.09 GB
  "indexSizes": {
    "_id_": 769077248,
    "device_id_1_create_timestamp_-1": 2934345728 
  },
```

由此可见即使有了压缩，也要考虑**数据和索引**所占用的存储空间。



**压缩**

存储文档时存储引擎（WiredTiger）会默认启用压缩

> 利用 WiredTiger，MongoDB 能够支持对所有集合和索引进行压缩。压缩能够最大限度减少存储使用量，但会消耗额外的 CPU 资源。

- 默认情况下，WiredTiger 对所有集合使用 **Snappy 区块压缩**
- 对所有索引使用**前缀压缩**



**内存使用**

借助 WiredTiger，MongoDB 可同时利用 WiredTiger 内部缓存和文件系统缓存。

默认 WiredTiger 内部缓存大小为以下两者中的较大者：

- （RAM 大小 - 1 GB）的 50%，或
- 256 MB.

例如，在总 RAM 为 4GB 的系统上，WiredTiger 缓存使用 1.5GB RAM (`0.5 * (4 GB - 1 GB) = 1.5 GB`)。相反，在总 RAM 为 1.25GB 的系统上，WiredTiger 为 WiredTiger 缓存分配了 256 MB，因为这大于总 RAM 的一半减去 1 GB (`0.5 * (1.25 GB - 1 GB) = 128 MB < 256 MB`)。



# java使用连接mongoDB

```yaml
spring:
  data:
    mongodb:
#      uri: mongodb://admin:syd123456@ip:27017/syd_app?authSource=admin
      host: ip
      database: syd_app
      authentication-database: admin
      username: syd
      password: syd123456
```



# 问题排查

## 重启失败排除

启动失败

```shell
root@iZwz9fh86qytc9uotx7wucZ:/tmp# sudo systemctl status mongod
● mongod.service - MongoDB Database Server
     Loaded: loaded (/lib/systemd/system/mongod.service; enabled; vendor preset: enabled)
     Active: failed (Result: exit-code) since Thu 2025-08-28 17:23:24 CST; 3s ago
       Docs: https://docs.mongodb.org/manual
    Process: 825347 ExecStart=/usr/bin/mongod --config /etc/mongod.conf (code=exited, status=14)
   Main PID: 825347 (code=exited, status=14)
```

查看日志，报错为没有`mongodb-27017.sock`这个的权限

```shell
sudo tail -n 50 /var/log/mongodb/mongod.log

{"t":{"$date":"2025-08-28T17:22:04.557+08:00"},"s":"E",  "c":"NETWORK",  "id":23024,   "ctx":"initandlisten","msg":"Failed to unlink socket file","attr":{"path":"/tmp/mongodb-27017.sock","error":"Operation not permitted"}}
{"t":{"$date":"2025-08-28T17:22:04.557+08:00"},"s":"F",  "c":"ASSERT",   "id":23091,   "ctx":"initandlisten","msg":"Fatal assertion","attr":{"msgid":40486,"file":"src/mongo/transport/asio/asio_transport_layer.cpp","line":1050}}
{"t":{"$date":"2025-08-28T17:22:04.557+08:00"},"s":"F",  "c":"ASSERT",   "id":23092,   "ctx":"initandlisten","msg":"\n\n***aborting after fassert() failure\n\n"}
```

修改权限

```shelll
sudo chmod 1777 /tmp/mongodb-27017.sock
```

修改后还是启动失败

最后，删掉`mongodb-27017.sock`才启动成功。



## 无法启动排除

权限问题

```
sudo chown -R mongodb:mongodb /var/lib/mongodb
sudo chown -R mongodb:mongodb /var/log/mongodb
sudo systemctl start mongod
```



数据文件损坏，**注意：这将清空所有剩余数据！**

```
sudo systemctl stop mongod
sudo rm -rf /var/lib/mongodb/*
sudo systemctl start mongod
```











