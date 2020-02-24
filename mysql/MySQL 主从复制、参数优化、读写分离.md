# MySQL 主从复制、参数优化、读写分离

## 配置要求

* 主从服务器操作系统版本和位数一致；
* Master 和 Slave 数据库的版本要一致；
* Master 和 Slave 数据库中的数据要一致；
* Master 开启二进制日志， Master 和 Slave 的 server_id 在局域网内必须唯一；

## 步骤概述

* Master 步骤
    1. 安装数据库；
    2. 修改数据库配置文件，设置 server_id，开启二进制日志(log-bin)；
    3. 启动数据库，查看当前日志文件（File）和位置（Position）；
    4. 登录数据库，授权数据复制用户；
    5. 备份主库（记得加锁/解锁）；
    6. 启动数据库；

* Slave 步骤
    1. 安装数据库；
    2. 修改数据库配置文件，设置 server_id，开启二进制日志(log-bin)；
    3. 启动数据库，还原备份；
    4. 设置 Master 的地址、用户、密码等信息；
    5. 开启同步，查看状态。

## 主节点

修改配置文件 my.cnf，在 `[mysqld]` 下加入以下配置，并重启 MySQL：

```bash
# 主库ID
server_id = 1

# 开启二进制日志 
log-bin = mysql-bin

# 需要复制的数据库名，若需复制多个库，重复设置这个选项即可                  
binlog-do-db = db   

# 将从 master 接收的更新也记入二进制日志中
log-slave-updates

# 控制 binlog 的写入频率
sync_binlog = 1

# 二进制日志自动删除的天数，默认值为 0,表示不删除 
expire_logs_days = 7

# 将函数复制到 slave
log_bin_trust_function_creators = 1
```

在主节点创建用于复制操作的用户：

```bash
create user 'sync_user'@'slave_host' identified by 'password';
grant replication slave on *.* TO 'sync_user'@'slave_host';
flush privileges;
```

`若需同步已有数据` 主库锁表：

```bash
flush tables with read lock;
```

获取主节点 binlog 文件（File）和位置（Position）：

```bash
show master status\G;
```

```bash
mysql> show master status\G;
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 2287
     Binlog_Do_DB: db
 Binlog_Ignore_DB: 
Executed_Gtid_Set: 
1 row in set (0.00 sec)
```

`若需同步已有数据` 创建主库的快照文件：

```bash
mysqldump -uroot -p -h master_host -P 3306 --all-databases --triggers --routines --events >>/all.sql
```

`若需同步已有数据` 解锁主库的锁表操作：

```bash
unlock tables;
```

## 从节点

`若需同步已有数据` 将主库的快照文件导入到从库中：

```bash
mysql -uroot -p -h slave_host -P 3306 < /all.sql
```

修改配置文件 my.cnf，在 `[mysqld]` 下加入以下配置并重启 MySQL：

```bash
server_id = 2
log-bin = mysql-bin
log-slave-updates
sync_binlog = 0
innodb_flush_log_at_trx_commit = 0
replicate-do-db = db # 需要复制的数据库名
slave-net-timeout = 60                    
log_bin_trust_function_creators = 1
```

设置主节点参数并执行同步命令：

```bash
change master to master_host='master_host',master_user='sync_user',master_password='password',master_log_file='mysql-bin.000001',master_log_pos=2287;

start slave;
```

查看从节点同步状态：

```bash
show slave status\G;
```

```bash
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.4
                  Master_User: sync_user
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 2287
               Relay_Log_File: c3ed087b6481-relay-bin.000002
                Relay_Log_Pos: 1615
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: db
          Replicate_Ignore_DB: 
          ...
          ...
```

`Slave_IO_Running` 和 `Slave_SQL_Running` 进程必须正常运行，即 `Yes` 状态，否则说明同步失败，若失败查看 mysql 错误日志中具体报错详情来进行问题定位。

最后可以去主节点上的数据库中创建表或者更新表数据来测试同步。

## MySQL 配置优化

## innodb_buffer_pool_size

说明：InnoDB 存储引擎的表数据和索引数据的最大内存缓冲区大小。默认为 128M。

建议：物理内存的 70% - 80%。

修改 my.cnf 并重启

```bash
[mysqld]
innodb_buffer_pool_size=xxx
```

MySQL 5.7 可直接在线修改：

```bash
SET GLOBAL innodb_buffer_pool_size=xxx;
```

*注：buffer_pool 在线调整期间会造成用户请求的阻塞，最好在业务空闲时修改。此外建议同时在 my.cnf 中做同样配置。*

示例：

```bash
innodb_buffer_pool_size=2147483648    # 2G
innodb_buffer_pool_size=2G            # 2G
innodb_buffer_pool_size=2048M         # 2G
```

## tmp_table_size 和 max_heap_table_size

说明：tmp_table_size 控制内部临时表的缓存大小，超出时将写入磁盘，max_heap_table_size 控制内存引擎的表大小。由于内存临时表也属于内存表，因此调整 tmp_table_size 同时也需要调整 max_heap_table_size。

建议：优化 SQL 语句，尽量避免使用内部临时表。tmp_table_size 是针对单张表的，根据实际大小而定，关注 `Created_tmp_disk_tables` 和 `Created_tmp_tables` 两个状态，以 `Created_tmp_disk_tables / Created_tmp_tables < 0.05` 为佳。

修改 my.cnf 并重启

```bash
[mysqld]
tmp_table_size=128M
max_heap_table_size=256M
```

MySQL 5.7 可直接在线修改：

```bash
SET GLOBAL tmp_table_size=128M;
SET GLOBAL max_heap_table_size=256M;
```

以下情况会用到内部临时表：

* 查询系统表数据
* 查询语句中带有 UNION
* 多表更新
* SQL 语句中使用 SQL_BUFFER_RESULT
* SQL 语句中包含了 DERIVED TABLE（FROM 中的子查询）
* SQL 中包含 DISTINCT 且无法被优化掉
* 连接表使用 BNL/BKA 算法且语句中包含 ORDER BY 或者 GROUP BY
* ORDER BY 或者 GROUP BY 的列不属于执行计划中第一个连接表的列
* ORDER BY 的表达式是复杂表达式，如 UDF 或 SP、包含聚集函数、包含 SCALAR SUBQUERY（SELECT 中的子查询）
* SQL 语句中同时包含 ORDER BY 和 GROUP BY 但两者使用的列不同
* 多表外连接且 GROUP BY 中带有 ROLLUP
* ORDER BY 使用的列来自 SCALAR SUBQUERY
* IN 表达式转换为 semi-join，若 semi-join 方式为 Materialization 或 Duplicate Weedout
* 聚合函数中包含：count(distinct *)、group_concat

## key_buffer_size

说明：MyISAM 表的索引缓存大小，决定索引的处理速度，尤其是索引的读取速度。

建议：关注 `Key_reads` 和 `Key_read_requests` 两个状态，以 `Key_reads / Key_read_requests < 0.001` 为佳。

修改 my.cnf 并重启

```bash
[mysqld]
key_buffer_size=1G
```

MySQL 5.7 可直接在线修改：

```bash
SET GLOBAL key_buffer_size=1G;
```

## sharding-proxy 读写分离

[ShardingSphere 官方文档][1]

  [1]: https://shardingsphere.apache.org/document/current/cn/overview/