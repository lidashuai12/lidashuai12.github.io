---
title: docker部署redis主从集群
date: 2023-07-14 18:14:40
tags: Redis
categories: 后端学习笔记
description: |
    **_CentOS7安装docker_**
---

​	本文是我自己实习的时候，公司丢给我了个活，要求把项目的memcached更换为redis。谨慎起见我自己搭建了一个**一主二从**的redis集群，另外由于我的电脑内存不足，所以考虑使用三个docker容器来组件集群。（docker真是yyds！）。

# **1.创建Redis网卡**

​	该网卡为连通各个容器用，可自定义网段，容器连接网卡后可以自定义配置自己的静态IP。后面配置redis集群的时候就使用到了自己指定的IP。

```bash
#配置docker网桥，指定局域网段172.12.0.0/16
docker network create redisCluster --subnet 172.12.0.0/16
```

# **2.拉取镜像**

```bash
docker pull redis
```

# **3.配置3个reidis服务**

​	由于操作繁琐，使用shell脚本进行配置。为三个节点分别创建文件夹用于挂载数据卷（文件路径和文件名按自己喜好修改~），名称为`node01`,`node02`,`node03`，每个节点文件夹下面分别有`conf`,`data`,`log`三个文件夹（`log`没有用到，但是也不影响）。

​	该shell脚本为三个节点创建了`redis.conf`文件，从节点与主节点的配置文件有少许的不同，脚本运行后可以手动修改~、

```bash
#!/bin/bash

for port in $(seq 1 3); \
do \
mkdir -p /docker-volume/redisCluster/node0${port}/conf
mkdir -p /docker-volume/redisCluster/node0${port}/data
mkdir -p /docker-volume/redisCluster/node0${port}/log
touch /docker-volume/redisCluster/node0${port}/conf/redis.conf
cat << EOF >/docker-volume/redisCluster/node0${port}/conf/redis.conf
port 6379
bind 0.0.0.0
protected-mode no
databases 1
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
appednonly no
appendfilename "appendonly.aof"
appendfsync everysec
#主节点可根据需要设置密码
#requirepass 123456
#主节点设置密码后，从节点写主服务器设置的密码
#masterauth 123456 
EOF
done
```

​	如果希望主从关系永久有效，运行后分别在从节点的`redis.conf`中添加一行：

```
#主节点ip，主节点端口号
replicaof 172.12.0.11 6379
```

​	否则在启动redis服务以后通过命令行进行主从关系绑定

​	创建好的文件夹是下面这样的：

![挂载数据卷树形目录](image-20230714183027176.png)



# **4.启动Redis服务**

​	同样使用shell脚本进行批量启动docker服务：

```bash
#!/bin/bash

for port in $(seq 1 3); \
do \
docker run -p $((port+6400)):6379 -p 1637${port}:16379 --name redis-${port} \
-v /docker-volume/redisCluster/node0${port}/data:/data \
-v /docker-volume/redisCluster/node0${port}/conf/redis.conf:/etc/redis/redis.conf \
-d --net redisCluster --ip 172.12.0.1${port} redis redis-server /etc/redis/redis.conf
done
```

- **-p 端口映射**
- **-v 数据卷挂载**
- **-d 后台运行**
- **–net 指定网络(与docker network一致)  --ip指定容器IP地址**
- **redis-server /etc/redis/redis.conf redis-server指向配置文件启动**

​	之后运行`docker ps`查看容器运行情况：

![查看三个redis容器运行情况](image-20230714183238319.png)

# **5.绑定主从关系**

进入master节点,连接redis：

```bash
#进入master节点
docker exec -it redis-1 /bin/bash
#通过容器ip地址连接redis从节点（redis-2）
redis-cli -h 172.12.0.12
#声明redis-2的主节点为redis-1
replicaof 172.12.0.11 6379

#通过容器ip地址连接redis从节点（redis-3）
redis-cli -h 172.12.0.13
#声明redis-3的主节点为redis-1
replicaof 172.12.0.11 6379
```

成功运行命令`info replication`以后查看集群情况：

![查看集群运行情况](image-20230714183328918.png)



# **6.使用get/set测试集群服务**

​	在redis-1执行`set name lja`命令，可在三台redis容器中都看到存入的数据

![测试是否主从复制](image-20230714183351288.png)
