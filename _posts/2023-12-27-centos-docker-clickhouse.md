---
layout: post
title: Centos多机Docker部署Clickhouse踩坑记
categories: Linux, Docker, DB
description: 避坑指南
keywords: Linux, Docker,Zookeeper,Clickhouse

---

# Centos多机Docker部署Clickhouse踩坑记
==================

## 前言
有了之前单机部署的经验，本来以为就是洒洒水的东西，没想到踩了一堆的坑，要记录一下！！

## 主要问题
1、容器间要互通，如果采用端口映射的模式，cluster是跑不起来的！！
2、zookeeper别用太高的版本，也别太低。。目前最好是官方推荐的3.5.10！！
3、default用户别设置密码，有密码cluster也跑不起来！！

## 不同机器上的Docker互通的问题
看了一堆的方案，感觉改动最小也最快的方式，应该是设路由表，加几条转发的方案。
网上找了个帖子参考，[地址在这](https://www.cnblogs.com/xiao987334176/p/10049844.html)，都处理完本来以为好了，谁知道iptables的规则有问题，并不能做到真正的nat转发，只是把所有流量转到了宿主机而已。
一通搜索以后，发现采用这个方案的文章基本都是引用的这篇，所以只能自己来了。

先描述一下环境，三台机器分别是
```
node1:10.96.28.60;
node2:10.96.28.165;
node3:10.96.28.182.
```
没有采用网上的修改docker本身网络的方案，新建了一个docker中的桥接网络

```bash
# node1的网段划分为172.18.0.0/24，并固定虚拟网卡名字为zkchnet0
docker network create -d bridge --subnet=172.18.0.0/24 --gateway=172.18.0.1 -o com.docker.network.bridge.name=zkchnet0 zkchnet

# node2的网段划分为172.18.1.0/24，并固定虚拟网卡名字为zkchnet0
docker network create -d bridge --subnet=172.18.1.0/24 --gateway=172.18.1.1 -o com.docker.network.bridge.name=zkchnet0 zkchnet

# node3的网段划分为172.18.2.0/24，并固定虚拟网卡名字为zkchnet0
docker network create -d bridge --subnet=172.18.2.0/24 --gateway=172.18.2.1 -o com.docker.network.bridge.name=zkchnet0 zkchnet
```

建立完桥接网络之后，设置对端路由表

```bash

# node1
route add -net 172.18.1.0/24 gw 10.96.28.165 dev eth0
route add -net 172.18.2.0/24 gw 10.96.28.182 dev eth0

# node2
route add -net 172.18.0.0/24 gw 10.96.28.60 dev eth0
route add -net 172.18.2.0/24 gw 10.96.28.182 dev eth0

# node3
route add -net 172.18.0.0/24 gw 10.96.28.60 dev eth0
route add -net 172.18.1.0/24 gw 10.96.28.165 dev eth0

```

设置完路由表之后，其实本地就已经会进行nat操作了，所以并不需要在iptables的nat表中进行操作，
通过抓包以及观察iptables流量等方式，最终确定流量到了eth0的主网卡之后，因为没有转发到docker生成的虚拟网卡，最终被丢弃，定位到问题就简单了

```bash
# 对到达目标地址的网络包转发到指定网卡，知道为什么前面网卡要制定名字了吧
# node1执行
iptables -I FORWARD -d 172.18.0.0/24 -i eth0 -o zkchnet0 -j ACCEPT

# node2执行
iptables -I FORWARD -d 172.18.1.0/24 -i eth0 -o zkchnet0 -j ACCEPT

# node3执行
iptables -I FORWARD -d 172.18.2.0/24 -i eth0 -o zkchnet0 -j ACCEPT

```

正常设置完毕以后，在zkchnet网络下的容器，应该就能互相ping通了

```bash
# 指定网络跑起来试试，每台机器默认应该都是获取到.2的地址
docker run --rm --network zkchnet -it 10.96.28.163:50000/library/busybox:latest
```

至此，完成了网络互通的操作，且没有多任何的管理模块以及外部项目，速度上测试了一下，也是非常快的

## zookeeper安装

这是个大坑，莫名其妙的整了四五个小时，3888端口愣是不通，服务跑不起来，
整得我几度怀疑是docker网络配置的问题，是不是还有什么协议没支持。。。

最后没辙，怕别是镜像包有问题，试一下更新镜像到zk的最新版本，当然，也清理了之前的数据目录，
然后，它就成功跑起来了，贼顺溜。。。

此处记录一下，踩坑版本的镜像：apache-zookeeper-3.8.0-bin+jdk-11.0.17+8，sha：8e28e2da

升级到最新版本之后，zk是没问题了，但是ck连不上这个版本。。zk日志一直报Connection request from old client。。
好吧，太新了也不行，[一番查找](https://clickhouse.com/docs/en/operations/tips)，官方推荐是3.5.1，找了3.5的最后一个版本，
3.5.10，镜像跑起来，参数啥的都没换，啥问题都没有了。。

大写的服气。。。

## 最后是clickhouse的配置了

别看remote_servers节点的配置里面，好像有个user和password，配上去并没有什么用处，而且host使用宿主机的ip地址，通过端口去区分也没有作用，
虽然看起来select * from system.clusters会有返回，但是一直不显示is_local字段，说明当前服务器并没有检测到集群中的哪一个是自己。

最终明确，不能配密码，不能配外部的ip地址或者hostname，只能配docker中的地址及对应的hostname，附上对应配置

顺便说下，这次没有修改ck生成的默认配置，所有修改项目都在config.d目录下新建对应配置进行覆盖

remote_servers.xml

```xml
<clickhouse>
    <remote_servers>
        <cluster_3s_1r>
            <!-- 数据分片1  -->
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch-node1</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>ch-replica1</host>
                    <port>9000</port>
                </replica>
            </shard>

            <!-- 数据分片2  -->
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch-node2</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>ch-replica2</host>
                    <port>9000</port>
                </replica>
            </shard>

            <!-- 数据分片3  -->
            <shard>
                <internal_replication>true</internal_replication>
                <replica>
                    <host>ch-node3</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>ch-replica3</host>
                    <port>9000</port>
                </replica>
            </shard>
        </cluster_3s_1r>
    </remote_servers>
</clickhouse>

```

zookeeper.xml

```xml
<clickhouse>
    <!-- ZK  -->
    <!-- zookeeper所有实例配置都一样，注意新版本不要带index，不然会报错 -->
    <zookeeper>
        <node>
            <host>zk-node1</host>
            <port>2181</port>
        </node>
        <node>
            <host>zk-node2</host>
            <port>2181</port>
        </node>
        <node>
            <host>zk-node3</host>
            <port>2181</port>
        </node>
    </zookeeper>
</clickhouse>

```

docker-compose.yaml(node1的)

```yaml
version: "3"
services:
  zookeeper:
    image: '10.96.28.163:50000/library/zookeeper:3.5.10'
    container_name: zk-node1
    hostname: zk-node1
    restart: always
    privileged: true
    networks:
      zkchnet:
        ipv4_address: 172.18.0.30
    ports:
    - '2181:2181'
    # - '2888:2888'
    # - '3888:3888'
    extra_hosts:
    - "clickhouse-001:10.96.28.60"
    - "clickhouse-002:10.96.28.165"
    - "clickhouse-003:10.96.28.182"
    - "zk-node1:172.18.0.30"
    - "zk-node2:172.18.1.30"
    - "zk-node3:172.18.2.30"
    volumes:
    - /etc/localtime:/etc/localtime
    - /data/zookeeper-cluster/node1/volumes/data:/data
    - /data/zookeeper-cluster/node1/volumes/datalog:/datalog
    - /data/zookeeper-cluster/node1/volumes/logs:/logs
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zk-node2:2888:3888;2181 server.3=zk-node3:2888:3888;2181
  ch-node:
    image: '10.96.28.163:50000/library/clickhouse-server:latest'
    container_name: ch-node1
    hostname: ch-node1
    networks:
      zkchnet:
        ipv4_address: 172.18.0.10
    restart: always
    ports:
    - '9000:9000'
    - '8123:8123'
    # - '9009:9009'
    cap_add:
    - SYS_NICE
    - NET_ADMIN
    - IPC_LOCK
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    extra_hosts:
    - "clickhouse-001:10.96.28.60"
    - "clickhouse-002:10.96.28.165"
    - "clickhouse-003:10.96.28.182"
    - "zk-node1:172.18.0.30"
    - "zk-node2:172.18.1.30"
    - "zk-node3:172.18.2.30"
    - "ch-node1:172.18.0.10"
    - "ch-node2:172.18.1.10"
    - "ch-node3:172.18.2.10"
    - "ch-replica1:172.18.1.20"
    - "ch-replica2:172.18.2.20"
    - "ch-replica3:172.18.0.20"
    volumes:
    - /etc/localtime:/etc/localtime
    - /data/clickhouse-cluster/node1/data/:/var/lib/clickhouse/
    - /data/clickhouse-cluster/node1/log/:/var/log/clickhouse-server/
    - /data/clickhouse-cluster/node1/config/:/etc/clickhouse-server/
    - /data/clickhouse-cluster/node1/initdb/:/docker-entrypoint-initdb.d/
    depends_on:
    - zookeeper
  ch-replica:
    image: '10.96.28.163:50000/library/clickhouse-server:latest'
    container_name: ch-replica3
    hostname: ch-replica3
    networks:
      zkchnet:
        ipv4_address: 172.18.0.20
    restart: always
    ports:
    - '9001:9000'
    - '8124:8123'
    # - '9010:9009'
    cap_add:
    - SYS_NICE
    - NET_ADMIN
    - IPC_LOCK
    ulimits:
      nofile:
        soft: 262144
        hard: 262144
    extra_hosts:
    - "clickhouse-001:10.96.28.60"
    - "clickhouse-002:10.96.28.165"
    - "clickhouse-003:10.96.28.182"
    - "zk-node1:172.18.0.30"
    - "zk-node2:172.18.1.30"
    - "zk-node3:172.18.2.30"
    - "ch-node1:172.18.0.10"
    - "ch-node2:172.18.1.10"
    - "ch-node3:172.18.2.10"
    - "ch-replica1:172.18.1.20"
    - "ch-replica2:172.18.2.20"
    - "ch-replica3:172.18.0.20"
    volumes:
    - /etc/localtime:/etc/localtime
    - /data/clickhouse-cluster/replica3/data/:/var/lib/clickhouse/
    - /data/clickhouse-cluster/replica3/log/:/var/log/clickhouse-server/
    - /data/clickhouse-cluster/replica3/config/:/etc/clickhouse-server/
    - /data/clickhouse-cluster/replica3/initdb/:/docker-entrypoint-initdb.d/
    depends_on:
    - zookeeper

networks:
  zkchnet:
    name: zkchnet
    driver: bridge
    ipam:
      config:
      - subnet: "172.18.0.0/24"
        gateway: "172.18.0.1"
    driver_opts:
      com.docker.network.bridge.name: zkchnet0

```