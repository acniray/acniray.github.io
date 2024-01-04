---
layout: post
title: PostgreSql 跨版本数据迁移
categories: PostgreSql
description: 跨版本数据迁移。
keywords: DB, PostgreSql

---

# PostgreSql 跨版本数据迁移

## 安装
服务器是centos7.6的，虽然在pg官网还能看到安装指引，但是在rhel7的包里面，已经没办法找到pg16的安装包了
```bash
# Install the repository RPM:
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Install PostgreSQL:
sudo yum install -y postgresql16-server

# Optionally initialize the database and enable automatic start:
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb
sudo systemctl enable postgresql-16
sudo systemctl start postgresql-16
```
所以上述命令执行完毕之后，会得到如下输出
```bash
No package postgresql16-server available.
Error: Nothing to do
```
只能装15了，省事点
```bash
# Install PostgreSQL:
sudo yum install -y postgresql15-server

# Optionally initialize the database and enable automatic start:
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
sudo systemctl enable postgresql-15
sudo systemctl start postgresql-15
```

## 修改数据目录

```bash
# 停止服务
sudo systemctl stop postgresql-15.service

# 创建目录
sudo mkdir /data/postgresql

# 变更权限
sudo chown -R postgres.postgres /data/postgresql/

# 移动data目录
sudo mv /var/lib/pgsql/15/data /data/postgresql/

# 修改服务
sudo vim /usr/lib/systemd/system/postgresql-15.service

# Environment=PGDATA=/var/lib/pgsql/15/data/
Environment=PGDATA=/data/postgresql/data/

# 重载配置
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start postgresql-15.service
```

## 物理复制（失败）
编辑主库的pg_hba.conf，增加当前机器的ip地址。

剩下的尝试跨版本物理复制，会提示版本不一致，所以还是要考虑逻辑复制的方式迁移数据

```bash
sudo pg_basebackup -h 172.32.0.6 -p 5432 -U postgres -D /data/postgresql/data  -Fp -P -Xs -R -v

sudo pg_rewind -D /data/postgresql/data -P --debug --source-server='host=172.32.0.6 port=5432 user=postgres password=123  dbname=db'

```

## 逻辑复制（pglogical）
尝试使用pglogical，安装方式如下：

pg12的老机器执行：

```bash
# 补充安装contrib包
sudo yum install postgresql15-contrib

# 安装2ndQuadrant’s General Public仓库
sudo curl https://techsupport.enterprisedb.com/api/repository/dl/default/release/15/rpm | bash

# 安装pglogical
sudo yum install postgresql15-pglogical

# 执行完毕后，service文件可能会被替换，导致无法启动，重新修改数据目录
sudo vim /usr/lib/systemd/system/postgresql-12.service

# Environment=PGDATA=/var/lib/pgsql/12/data/
Environment=PGDATA=/data/postgresql/data/

# 重载配置
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start postgresql-12.service

```

pg15的机器执行：

```bash
# 补充安装contrib包
sudo yum install postgresql15-contrib

# 安装2ndQuadrant’s General Public仓库
sudo curl https://techsupport.enterprisedb.com/api/repository/dl/default/release/15/rpm | bash

# 安装pglogical
sudo yum install postgresql15-pglogical

```

pglogical设置

修改postgresql.conf:

```ini
wal_level = 'logical'
max_worker_processes = 10   # one per database needed on provider node
                            # one per node needed on subscriber node
max_replication_slots = 10  # one per node needed on provider node
max_wal_senders = 10        # one per node needed on provider node
shared_preload_libraries = 'pglogical'
```

通过命令行工具执行sql

```bash
su - postgres

psql
```

```sql
--切换到指定db
\c db

--在发布节点和订阅节点都创建扩展模块
CREATE EXTENSION pglogical;

--创建发布节点
SELECT pglogical.create_node(
    node_name := 'provider1',
    dsn := 'host=172.32.0.6 port=5432 dbname=db user=provider_user password=provider_pass'
);

--将所有 public 模式下表都添加到默认复制集。(注意：所有表必须要有主键)
SELECT pglogical.replication_set_add_all_tables('default', ARRAY['public']);

--创建订阅节点
SELECT pglogical.create_node(
    node_name := 'subscriber1',
    dsn := 'host=172.32.0.242 port=5432 dbname=db user=subscriber_user password=subscriber_pass'
  );

--订阅
SELECT pglogical.create_subscription(
    subscription_name := 'subscription1',
    synchronize_structure := true, --自动同步结构需要加上这一句，不然得手动建好表才能开始同步
    provider_dsn := 'host=172.32.0.6 port=5432 dbname=db user=provider_user password=provider_pass'
);

--等待订阅同步完成
SELECT pglogical.wait_for_subscription_sync_complete('subscription1');

--查询复制集中的表
select * from pglogical.replication_set_table ;

--查询指定表的同步状态
select * from pglogical.show_subscription_table('subscription1','cpe');

--查询订阅状态
select * from pglogical.show_subscription_status();

--查询表中记录数
select relname as TABLE_NAME, reltuples as rowCounts
from pg_class
where relkind = 'r'
  and relnamespace = (select oid from pg_namespace where nspname = 'public')
order by rowCounts desc;
```

订阅方法参数
```sql
select pglogical.create_subscription(
    subscription_name name, 
    provider_dsn text, 
    replication_sets text[], 
    synchronize_structure boolean, 
    synchronize_data boolean, 
    forward_origins text[], 
    apply_delay interval
    ) 
```

* subscription_name - 订阅的名称，必须是唯一的
* provider_dsn - 提供者的连接字符串
* replication_sets - 要订阅的复制集数组，这些必须已存在，默认为“{default，default_insert_only，ddl_sql}”
* synchronize_structure - 指定是否将提供者与订阅者之间的结构同步，默认为false
* synchronize_data - 指定是否将数据从提供者同步到订阅者，默认为true
* forward_origins - 要转发的原始名称数组，当前只支持的值是空数组，意味着不转发任何不是源自提供者节点的更改，或“{all}”这意味着复制所有更改，无论它们的来源是什么，默认是全部}”
* apply_delay - 延迟复制多少，默认为0秒

## 参考资料
* [pglogical官方github](https://github.com/2ndQuadrant/pglogical)
* [PostgreSQL逻辑复制之pglogical篇](https://www.cnblogs.com/lottu/p/10972773.html)