---
layout: post
title: Centos单机Docker多节点部署Clickhouse记录
categories: Linux, Docker, DB
description: 记录下docker多节点clickhouse部署
keywords: Linux, Docker,Zookeeper,Clickhouse

---

Centos单机Docker多节点部署Clickhouse记录
==================

# 准备工作

## 拉取镜像：

```bash
docker pull clickhouse/clickhouse-server

docker pull zookeeper
```

## 查看镜像：

```bash
docker images
```

```bash
clickhouse/clickhouse-server                latest              e6a6aaca7e9d        2 days ago          910MB
docker.io/zookeeper                         latest              248549379309        4 weeks ago         300MB
```

## 创建子网络：

```bash
docker network create --driver bridge --subnet=172.18.0.0/16 --gateway=172.18.0.1 zkchnet
```

# ZooKeeper

## 创建Zookeeper存储路径：

```bash
mkdir /data1/zookeeper-cluster
mkdir /data1/zookeeper-cluster/node1
mkdir /data1/zookeeper-cluster/node2
mkdir /data1/zookeeper-cluster/node3
```

## 创建Zookeeper容器：

### node1:

```bash
docker run -d -p 2181:2181 --name zookeeper_node1 --privileged --restart always --network zkchnet --ip 172.18.0.2 \
-v /data1/zookeeper-cluster/node1/volumes/data:/data \
-v /data1/zookeeper-cluster/node1/volumes/datalog:/datalog \
-v /data1/zookeeper-cluster/node1/volumes/logs:/logs \
-e ZOO_MY_ID=1 \
-e "ZOO_SERVERS=server.1=172.18.0.2:2888:3888;2181 server.2=172.18.0.3:2888:3888;2181 server.3=172.18.0.4:2888:3888;2181" docker.io/zookeeper
```

### node2:

```bash
docker run -d -p 2182:2181 --name zookeeper_node2 --privileged --restart always --network zkchnet --ip 172.18.0.3 \
-v /data1/zookeeper-cluster/node2/volumes/data:/data \
-v /data1/zookeeper-cluster/node2/volumes/datalog:/datalog \
-v /data1/zookeeper-cluster/node2/volumes/logs:/logs \
-e ZOO_MY_ID=2 \
-e "ZOO_SERVERS=server.1=172.18.0.2:2888:3888;2181 server.2=172.18.0.3:2888:3888;2181 server.3=172.18.0.4:2888:3888;2181" docker.io/zookeeper
```

### node3:

```bash
docker run -d -p 2183:2181 --name zookeeper_node3 --privileged --restart always --network zkchnet --ip 172.18.0.4 \
-v /data1/zookeeper-cluster/node3/volumes/data:/data \
-v /data1/zookeeper-cluster/node3/volumes/datalog:/datalog \
-v /data1/zookeeper-cluster/node3/volumes/logs:/logs \
-e ZOO_MY_ID=3 \
-e "ZOO_SERVERS=server.1=172.18.0.2:2888:3888;2181 server.2=172.18.0.3:2888:3888;2181 server.3=172.18.0.4:2888:3888;2181" docker.io/zookeeper
```

### 启动后，调用容器内命令验证一下：

```bash
docker ps
```

```bash
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                  NAMES
d7c9fb6791bf        248549379309        "/docker-entrypoint.…"   37 minutes ago      Up 37 minutes       2888/tcp, 3888/tcp, 8080/tcp, 0.0.0.0:2183->2181/tcp   zookeeper_node3
87959d4388b3        248549379309        "/docker-entrypoint.…"   37 minutes ago      Up 37 minutes       2888/tcp, 3888/tcp, 8080/tcp, 0.0.0.0:2182->2181/tcp   zookeeper_node2
31af65637340        248549379309        "/docker-entrypoint.…"   37 minutes ago      Up 37 minutes       2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, 8080/tcp   zookeeper_node1
```

```bash
$ docker exec -it zookeeper_node3 ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
$ docker exec -it zookeeper_node2 ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: leader
$ docker exec -it zookeeper_node1 ./bin/zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
Client port found: 2181. Client address: localhost. Client SSL: false.
Mode: follower
```

# ClickHouse

## 获取Clickhouse配制文件样本：

单独执行一次样例之后，将默认配制文件存储在/data/config-demo目录下

```bash
docker run -d --name clickhouse-server --ulimit nofile=262144:262144 clickhouse/clickhouse-server

cd /data

docker cp clickhouse-server:/etc/clickhouse-server/ ./config-demo/

docker stop clickhouse-server

docker rm clickhouse-server
```

## 创建Clickhouse存储路径：

因为是单机部署，分别将数据放在两块不同的硬盘上，然后作为副本：

```bash
mkdir /data1/clickhouse-cluster

mkdir /data1/clickhouse-cluster/node1
mkdir /data1/clickhouse-cluster/node1/data
mkdir /data1/clickhouse-cluster/node1/log
mkdir /data1/clickhouse-cluster/node1/config
mkdir /data1/clickhouse-cluster/node1/initdb

mkdir /data1/clickhouse-cluster/node3
mkdir /data1/clickhouse-cluster/node3/data
mkdir /data1/clickhouse-cluster/node3/log
mkdir /data1/clickhouse-cluster/node3/config
mkdir /data1/clickhouse-cluster/node3/initdb

mkdir /data1/clickhouse-cluster/replica2
mkdir /data1/clickhouse-cluster/replica2/data
mkdir /data1/clickhouse-cluster/replica2/log
mkdir /data1/clickhouse-cluster/replica2/config
mkdir /data1/clickhouse-cluster/replica2/initdb

mkdir /data/clickhouse-cluster

mkdir /data/clickhouse-cluster/replica1
mkdir /data/clickhouse-cluster/replica1/data
mkdir /data/clickhouse-cluster/replica1/log
mkdir /data/clickhouse-cluster/replica1/config
mkdir /data/clickhouse-cluster/replica1/initdb

mkdir /data/clickhouse-cluster/replica3
mkdir /data/clickhouse-cluster/replica3/data
mkdir /data/clickhouse-cluster/replica3/log
mkdir /data/clickhouse-cluster/replica3/config
mkdir /data/clickhouse-cluster/replica3/initdb

mkdir /data/clickhouse-cluster/node2
mkdir /data/clickhouse-cluster/node2/data
mkdir /data/clickhouse-cluster/node2/log
mkdir /data/clickhouse-cluster/node2/config
mkdir /data/clickhouse-cluster/node2/initdb
```

## 修改配置文件

在网上查到的教程中，都会增加一个metrika.xml的配置文件，然后修改config.xml，include_from引用进去，用于分离配置，但是这都是老版本的配置方法了，新版本建议直接在config.xml中完成通用配置，主要修改以下两个配置节点：

```xml
<!--所有实例均使用这个集群配置，不用个性化 -->
<clickhouse>

    <!-- 集群配置 -->
    <!-- remote_servers所有实例配置都一样 -->
    <remote_servers>
        <!-- 集群名称 -->
        <cluster_3s_1r>

            <!-- 数据分片1  -->
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>172.18.0.10</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
                <replica>
                    <host>172.18.0.13</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
            </shard>

            <!-- 数据分片2  -->
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>172.18.0.11</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
                <replica>
                    <host>172.18.0.14</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
            </shard>

            <!-- 数据分片3  -->
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>172.18.0.12</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
                <replica>
                    <host>172.18.0.15</host>
                    <port>9000</port>
                    <user>default</user>
                    <password></password>
                </replica>
            </shard>
        </cluster_3s_1r>
    </remote_servers>

    <!-- ZK  -->
    <!-- zookeeper所有实例配置都一样，注意新版本不要带index，不然会报错 -->
    <zookeeper>
        <node>
            <host>172.18.0.2</host>
            <port>2181</port>
        </node>
        <node>
            <host>172.18.0.3</host>
            <port>2181</port>
        </node>
        <node>
            <host>172.18.0.4</host>
            <port>2181</port>
        </node>
    </zookeeper>

</clickhouse>
```

每个容器中不同的部分只有macros，在config.d中，新增macros.xml，内容如下，里面的shard和replica标签名和内容都是自己命名的，在建表时可以引用，如分片3副本2，每个实例本身也算一个副本。

```xml
<clickhouse>
    <macros>
        <shard>02</shard>
        <replica>rep_02_02</replica>
    </macros>
</clickhouse>
```

新版本在这个基础上，就完成了所有节点的配制，下面是老版本的参考！



PS：注意如果是用网络上其他的一些metrika.xml，老版本是clickhouse_remote_servers，新版本是remote_servers，具体可以查看config.xml中的标签项。

```xml
<include_from>/etc/clickhouse-server/metrika.xml</include_from>
<!-- 如果zookeeper标签使用的是zookeeper-servers，还要加入下面这句-->   
<zookeeper incl=”zookeeper-servers” optional=”true” />
```

## 准备安装命令

根据[官方的Docker安装指南]([Docker](https://hub.docker.com/r/clickhouse/clickhouse-server/))，进行命令行配置，其中对于路径或者端口这些就略过，有一点提到Linux Capabilities涉及到一些Clickhouse特性的节点，这边记录一下，主要涉及到如下三个特性：

> **CAP_SYS_NICE**
>               * Lower the process nice value ([nice(2)](https://man7.org/linux/man-pages/man2/nice.2.html), [setpriority(2)](https://man7.org/linux/man-pages/man2/setpriority.2.html)) and change the nice value for arbitrary processes; 
>               * set real-time scheduling policies for calling process,  and set scheduling policies and priorities for arbitrary processes ([sched_setscheduler(2)](https://man7.org/linux/man-pages/man2/sched_setscheduler.2.html), [sched_setparam(2)](https://man7.org/linux/man-pages/man2/sched_setparam.2.html),  [sched_setattr(2)](https://man7.org/linux/man-pages/man2/sched_setattr.2.html)); 
>               * set CPU affinity for arbitrary processes([sched_setaffinity(2)](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html));
>               * set I/O scheduling class and priority for arbitrary processes ([ioprio_set(2)](https://man7.org/linux/man-pages/man2/ioprio_set.2.html));
>               * apply [migrate_pages(2)](https://man7.org/linux/man-pages/man2/migrate_pages.2.html) to arbitrary processes and allow processes to be migrated to arbitrary nodes;
>               * apply [move_pages(2)](https://man7.org/linux/man-pages/man2/move_pages.2.html) to arbitrary processes; 
>               * use the **MPOL_MF_MOVE_ALL** flag with [mbind(2)](https://man7.org/linux/man-pages/man2/mbind.2.html) and [move_pages(2)](https://man7.org/linux/man-pages/man2/move_pages.2.html).
> **CAP_IPC_LOCK**
>               * Lock memory ([mlock(2)](https://man7.org/linux/man-pages/man2/mlock.2.html), [mlockall(2)](https://man7.org/linux/man-pages/man2/mlockall.2.html), [mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html), [shmctl(2)](https://man7.org/linux/man-pages/man2/shmctl.2.html));
>               * Allocate memory using huge pages ([memfd_create(2)](https://man7.org/linux/man-pages/man2/memfd_create.2.html), [mmap(2)](https://man7.org/linux/man-pages/man2/mmap.2.html), [shmctl(2)](https://man7.org/linux/man-pages/man2/shmctl.2.html)).
> 
> **CAP_NET_ADMIN**
>               Perform various network-related operations:
>               * interface configuration;
>               * administration of IP firewall, masquerading, and accounting;
>               * modify routing tables;
>               * bind to any address for transparent proxying;
>               * set type-of-service (TOS);
>               * clear driver statistics;
>               * set promiscuous mode;
>               * enabling multicasting;
>               * use [setsockopt(2)](https://man7.org/linux/man-pages/man2/setsockopt.2.html) to set the following socket options: **SO_DEBUG**, **SO_MARK**, **SO_PRIORITY** (for a priority outside the range 0 to 6), **SO_RCVBUFFORCE**, and **SO_SNDBUFFORCE**.

在Clickhouse的更新日志中，找到关于SYS_NICE的描述：

> ClickHouse Release 19.11.3.11, 2019-07-18
> 
> * Added `os_thread_priority` setting that allows to control the “nice” value of query processing threads that is used by OS to adjust dynamic scheduling priority. It requires `CAP_SYS_NICE` capabilities to work. This implements [#5858](https://github.com/ClickHouse/ClickHouse/issues/5858) [#5909](https://github.com/ClickHouse/ClickHouse/pull/5909) ([alexey-milovidov](https://github.com/alexey-milovidov))

```bash
docker run -d \
    -p 19000:9000 -p 18123:8123 -p 19009:9009 --network zkchnet --ip 172.18.0.10 \
    --cap-add=SYS_NICE --cap-add=NET_ADMIN --cap-add=IPC_LOCK \
    -v /data1/clickhouse-cluster/node1/data/:/var/lib/clickhouse/ \
    -v /data1/clickhouse-cluster/node1/log/:/var/log/clickhouse-server/ \
    -v /data1/clickhouse-cluster/node1/config/:/etc/clickhouse-server/ \
    -v /data1/clickhouse-cluster/node1/initdb/:/docker-entrypoint-initdb.d/ \
    --name ch-node1 --ulimit nofile=262144:262144 clickhouse/clickhouse-server


docker run -d \
    -p 19001:9000 -p 18124:8123 -p 19010:9009 --network zkchnet --ip 172.18.0.11 \
    --cap-add=SYS_NICE --cap-add=NET_ADMIN --cap-add=IPC_LOCK \
    -v /data/clickhouse-cluster/node2/data/:/var/lib/clickhouse/ \
    -v /data/clickhouse-cluster/node2/log/:/var/log/clickhouse-server/ \
    -v /data/clickhouse-cluster/node2/config/:/etc/clickhouse-server/ \
    -v /data/clickhouse-cluster/node2/initdb/:/docker-entrypoint-initdb.d/ \
    --name ch-node2 --ulimit nofile=262144:262144 clickhouse/clickhouse-server


docker run -d \
    -p 19002:9000 -p 18125:8123 -p 19011:9009 --network zkchnet --ip 172.18.0.12 \
    --cap-add=SYS_NICE --cap-add=NET_ADMIN --cap-add=IPC_LOCK \
    -v /data1/clickhouse-cluster/node3/data/:/var/lib/clickhouse/ \
    -v /data1/clickhouse-cluster/node3/log/:/var/log/clickhouse-server/ \
    -v /data1/clickhouse-cluster/node3/config/:/etc/clickhouse-server/ \
    -v /data1/clickhouse-cluster/node3/initdb/:/docker-entrypoint-initdb.d/ \
    --name ch-node3 --ulimit nofile=262144:262144 clickhouse/clickhouse-server


docker run -d \
    -p 19003:9000 -p 18126:8123 -p 19012:9009 --network zkchnet --ip 172.18.0.13 \
    --cap-add=SYS_NICE --cap-add=NET_ADMIN --cap-add=IPC_LOCK \
    -v /data/clickhouse-cluster/replica1/data/:/var/lib/clickhouse/ \
    -v /data/clickhouse-cluster/replica1/log/:/var/log/clickhouse-server/ \
    -v /data/clickhouse-cluster/replica1/config/:/etc/clickhouse-server/ \
    -v /data/clickhouse-cluster/replica1/initdb/:/docker-entrypoint-initdb.d/ \
    --name ch-replica1 --ulimit nofile=262144:262144 clickhouse/clickhouse-server


docker run -d \
    -p 19004:9000 -p 18127:8123 -p 19013:9009 --network zkchnet --ip 172.18.0.14 \
    --cap-add=SYS_NICE --cap-add=NET_ADMIN --cap-add=IPC_LOCK \
    -v /data1/clickhouse-cluster/replica2/data/:/var/lib/clickhouse/ \
    -v /data1/clickhouse-cluster/replica2/log/:/var/log/clickhouse-server/ \
    -v /data1/clickhouse-cluster/replica2/config/:/etc/clickhouse-server/ \
    -v /data1/clickhouse-cluster/replica2/initdb/:/docker-entrypoint-initdb.d/ \
    --name ch-replica2 --ulimit nofile=262144:262144 clickhouse/clickhouse-server


docker run -d \
    -p 19005:9000 -p 18128:8123 -p 19014:9009 --network zkchnet --ip 172.18.0.15 \
    --cap-add=SYS_NICE --cap-add=NET_ADMIN --cap-add=IPC_LOCK \
    -v /data/clickhouse-cluster/replica3/data/:/var/lib/clickhouse/ \
    -v /data/clickhouse-cluster/replica3/log/:/var/log/clickhouse-server/ \
    -v /data/clickhouse-cluster/replica3/config/:/etc/clickhouse-server/ \
    -v /data/clickhouse-cluster/replica3/initdb/:/docker-entrypoint-initdb.d/ \
    --name ch-replica3 --ulimit nofile=262144:262144 clickhouse/clickhouse-server
```

TIPS: centos版本7.2.1511遇到host与container无法通讯的问题，定位到的问题是3.10.0-327.el7.x86_64内核的网桥核心问题，只能更新内核或者系统解决。

## 安装后验证

```bash
docker ps
```

```bash
CONTAINER ID   IMAGE                          COMMAND                  CREATED          STATUS          PORTS                                                                       NAMES
a19398d4c684   clickhouse/clickhouse-server   "/entrypoint.sh"         5 minutes ago    Up 5 minutes    0.0.0.0:18128->8123/tcp, 0.0.0.0:19005->9000/tcp, 0.0.0.0:19014->9009/tcp   ch-replica3
139865aead16   clickhouse/clickhouse-server   "/entrypoint.sh"         5 minutes ago    Up 5 minutes    0.0.0.0:18127->8123/tcp, 0.0.0.0:19004->9000/tcp, 0.0.0.0:19013->9009/tcp   ch-replica2
6db0ed6d3a7a   clickhouse/clickhouse-server   "/entrypoint.sh"         5 minutes ago    Up 5 minutes    0.0.0.0:18126->8123/tcp, 0.0.0.0:19003->9000/tcp, 0.0.0.0:19012->9009/tcp   ch-replica1
2f411dcf95fa   clickhouse/clickhouse-server   "/entrypoint.sh"         5 minutes ago    Up 5 minutes    0.0.0.0:18125->8123/tcp, 0.0.0.0:19002->9000/tcp, 0.0.0.0:19011->9009/tcp   ch-node3
bd199cbcac4b   clickhouse/clickhouse-server   "/entrypoint.sh"         5 minutes ago    Up 5 minutes    0.0.0.0:18124->8123/tcp, 0.0.0.0:19001->9000/tcp, 0.0.0.0:19010->9009/tcp   ch-node2
da3235668a63   clickhouse/clickhouse-server   "/entrypoint.sh"         5 minutes ago    Up 5 minutes    0.0.0.0:18123->8123/tcp, 0.0.0.0:19000->9000/tcp, 0.0.0.0:19009->9009/tcp   ch-node1
8c59f2402488   zookeeper                      "/docker-entrypoint.…"   58 minutes ago   Up 58 minutes   2888/tcp, 3888/tcp, 8080/tcp, 0.0.0.0:2183->2181/tcp                        zookeeper_node3
684e869fbf87   zookeeper                      "/docker-entrypoint.…"   58 minutes ago   Up 58 minutes   2888/tcp, 3888/tcp, 8080/tcp, 0.0.0.0:2182->2181/tcp                        zookeeper_node2
32ec4ed9e4f9   zookeeper                      "/docker-entrypoint.…"   58 minutes ago   Up 58 minutes   2888/tcp, 3888/tcp, 0.0.0.0:2181->2181/tcp, 8080/tcp                        zookeeper_node1
```

进入ch-node1执行clickhoust-client进行验证

```bash
docker exec -it ch-node1 bin/bash

clickhouse-client --host 172.18.0.11
ClickHouse client version 23.2.4.12 (official build).
Connecting to 172.18.0.11:9000 as user default.
Connected to ClickHouse server version 23.2.4 revision 54461.

Warnings:
 * Linux transparent hugepages are set to "always". Check /sys/kernel/mm/transparent_hugepage/enabled

bd199cbcac4b :) SELECT * FROM system.clusters;

SELECT *
FROM system.clusters

Query id: 6167eef8-0b61-49d8-b118-7eb9f3a08456

┌─cluster───────┬─shard_num─┬─shard_weight─┬─replica_num─┬─host_name───┬─host_address─┬─port─┬─is_local─┬─user────┬─default_database─┬─errors_count─┬─slowdowns_count─┬─estimated_recovery_time─┐
│ cluster_3s_1r │         1 │            1 │           1 │ 172.18.0.10 │ 172.18.0.10  │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
│ cluster_3s_1r │         1 │            1 │           2 │ 172.18.0.13 │ 172.18.0.13  │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
│ cluster_3s_1r │         2 │            1 │           1 │ 172.18.0.11 │ 172.18.0.11  │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │
│ cluster_3s_1r │         2 │            1 │           2 │ 172.18.0.14 │ 172.18.0.14  │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
│ cluster_3s_1r │         3 │            1 │           1 │ 172.18.0.12 │ 172.18.0.12  │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
│ cluster_3s_1r │         3 │            1 │           2 │ 172.18.0.15 │ 172.18.0.15  │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │
└───────────────┴───────────┴──────────────┴─────────────┴─────────────┴──────────────┴──────┴──────────┴─────────┴──────────────────┴──────────────┴─────────────────┴─────────────────────────┘

6 rows in set. Elapsed: 0.001 sec.
```

至此，安装过程完成。
