# Mysql主从复制集群搭建-基于DockerCompose

## 软件安装
1. 软件列表

| 名称             | 版本            |
|----------------|---------------|
| Docker         | 1\.14\.0\-rc2 |
| Mysql          | 5\.7\.17      |
| Docker compose | 19\.03\.1     |

2. 目录结构
```
./
  - docker-compose.yml
  - master/
    - my.cnf
    - Dockerfile
  - slave/
    - my.cnf
    - Dockerfile
```

## 配置docker-compose.yml
docker-compose.yml
```
version: '2'
services:
  mysql-master:
    build:
      context: ./
      dockerfile: master/Dockerfile
    environment:
      - "MYSQL_ROOT_PASSWORD=root"
      - "MYSQL_DATABASE=replicas_db"
    links:
      - mysql-slave
    ports:
      - "33065:3306"
    restart: always
    hostname: mysql-master
  mysql-slave:
    build:
      context: ./
      dockerfile: slave/Dockerfile
    environment:
      - "MYSQL_ROOT_PASSWORD=root"
      - "MYSQL_DATABASE=replicas_db"
    ports:
      - "33066:3306"
    restart: always
    hostname: mysql-slave
```
## master mysql容器配置
1. 配置Dockerfile

```
FROM mysql:5.7.17
MAINTAINER harrison
ADD ./master/my.cnf /etc/mysql/my.cnf
```
2. 配置my.cnf

```
[mysqld]
## 设置server_id，一般设置为IP，注意要唯一
server_id=100  
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql  
## 开启二进制日志功能，可以随便取，最好有含义（关键就是这里了）
log-bin=replicas-mysql-bin  
## 为每个session分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M  
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed  
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7  
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062
```
## slave mysql容器配置
1. 配置Dockerfile

```
FROM mysql:5.7.17
MAINTAINER harrison
ADD ./slave/my.cnf /etc/mysql/my.cnf
```
2. 配置my.cnf

```
[mysqld]
## 设置server_id，一般设置为IP，注意要唯一
server_id=101  
## 复制过滤：也就是指定哪个数据库不用同步（mysql库一般不同步）
binlog-ignore-db=mysql  
## 开启二进制日志功能，以备Slave作为其它Slave的Master时使用
log-bin=replicas-mysql-slave1-bin  
## 为每个session 分配的内存，在事务过程中用来存储二进制日志的缓存
binlog_cache_size=1M  
## 主从复制的格式（mixed,statement,row，默认格式是statement）
binlog_format=mixed  
## 二进制日志自动删除/过期的天数。默认值为0，表示不自动删除。
expire_logs_days=7  
## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
slave_skip_errors=1062  
## relay_log配置中继日志
relay_log=replicas-mysql-relay-bin  
## log_slave_updates表示slave将复制事件写进自己的二进制日志
log_slave_updates=1  
## 防止改变数据(除了特殊的线程)
read_only=1  
```
## 启动容器
在当前目录，运行 docker-compose 启动命令。
```
docker-compose up -d
```
如下面输出，则表明容器已经启动成功
```
[root@localhost my-mysql]# docker-compose up -d
Building mysql-slave
Step 1/3 : FROM mysql:5.7.17
 ---> 9546ca122d3a
Step 2/3 : MAINTAINER harrison
 ---> Running in df2ad9059e34
Removing intermediate container df2ad9059e34
 ---> eb0aacfa9ec1
Step 3/3 : ADD ./slave/my.cnf /etc/mysql/my.cnf
 ---> c6b14fcc1fcd
Successfully built c6b14fcc1fcd
Successfully tagged mymysql_mysql-slave:latest
WARNING: Image for service mysql-slave was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Building mysql-master
Step 1/3 : FROM mysql:5.7.17
 ---> 9546ca122d3a
Step 2/3 : MAINTAINER harrison
 ---> Using cache
 ---> eb0aacfa9ec1
Step 3/3 : ADD ./master/my.cnf /etc/mysql/my.cnf
 ---> 193dc5745984
Successfully built 193dc5745984
Successfully tagged mymysql_mysql-master:latest
WARNING: Image for service mysql-master was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating mymysql_mysql-slave_1 ... 
Creating mymysql_mysql-slave_1 ... done
Creating mymysql_mysql-master_1 ... 
Creating mymysql_mysql-master_1 ... done
```
使用数据库管理工具，可以看到，单独连接master，slave数据库都是能连接成功。

没有成功的请假差配置文件是否输入正确，这一步成功才能记录后续操作。



## 从数据库配置
走到这一步去，表明主从数据库的安装已经是成功了，接下来把slave数据库关联上master，就能让我们的mysql主从集群搭建成功。
那要怎么关联上master呢？就是在**从数据配置主数据库`binary-log`的文件名称和数据同步位置** 

1. 首先在主数据库查询主库的状态

   ``` show master status;```

   可以看到直接结果为：

   `File: replicas-mysql-bin.000003`

   `Position: 154`

2.  在 **从数据库** 上运行 **主数据库** 的相关配置 `sql` 进行主从关联

   ```sql
   CHANGE MASTER TO
   MASTER_HOST='mysql-master',
   MASTER_USER='root',
   MASTER_PASSWORD='root',
   MASTER_LOG_FILE='replicas-mysql-bin.000003', -- File文件名
   MASTER_LOG_POS=154; -- binlog记录位置
   ```

3.  重新启动 `slave` 服务 

   ```
   docker restart 从数据库容器名/容器id
   ```

## 测试

 在 **主数据库** 中创建一张测试数据表`test`

sql 如下：

``` sql
DROP TABLE IF EXISTS `test`;
CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(20) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

然后在从数据刷新，看是否有`test`已经从主数据库同步过来了。看到了则表明我们基于`docker compose`的**mysql主从集群**已经搭建成功！