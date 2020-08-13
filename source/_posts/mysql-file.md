---
title: MySQL文件
date: 2020-04-15 18:48:38
categories:
- MySQL
tags:
- MySQL
---

## 参数文件
当MySQL实例启动，数据库会先去读一个配置参数文件，用来寻找数据库的各种文件所在位置以及部分初始化参数。

可以通过`SHOW VARIABLES`查看数据库中所有参数，可以通过`LIKE`过滤参数名。

### 参数类型
MySQL中参数分为两类：
- 动态参数
    - 在MySQL实例运行中进行更改。 
- 静态参数
    - 在实例的整个生命周期内都不得进行更改。

可以通过set命令对动态参数进行修改，例如`SET read_buffer_size=524288`。
对变量的修改，在这次的实例生命周期内有效，下次此洞MySQL实例还是会读取参数文件。

## 日志文件

### 错误日志
错误日志对MySQL的启动、运行、关闭过程进行了记录，该文件不仅记录了所有的错误信息，也记录了一些警告信息或正确的信息。可以通过`SHOW VARIABLES LIKE 'log_error'`来定位该文件。

```log
2020-04-13T03:04:23.391925Z 75 [Note] Aborted connection 75 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:04:23.391954Z 76 [Note] Aborted connection 76 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.802373Z 77 [Note] Aborted connection 77 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.802390Z 82 [Note] Aborted connection 82 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.809111Z 79 [Note] Aborted connection 79 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.809298Z 81 [Note] Aborted connection 81 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.809495Z 80 [Note] Aborted connection 80 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.809647Z 83 [Note] Aborted connection 83 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.818503Z 84 [Note] Aborted connection 84 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.820436Z 85 [Note] Aborted connection 85 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.822052Z 86 [Note] Aborted connection 86 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:08:39.809996Z 78 [Note] Aborted connection 78 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.627802Z 87 [Note] Aborted connection 87 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.627848Z 88 [Note] Aborted connection 88 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.627864Z 89 [Note] Aborted connection 89 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.627891Z 90 [Note] Aborted connection 90 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.627917Z 91 [Note] Aborted connection 91 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.627949Z 92 [Note] Aborted connection 92 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.627994Z 93 [Note] Aborted connection 93 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.628018Z 94 [Note] Aborted connection 94 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.628034Z 95 [Note] Aborted connection 95 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T03:17:12.628055Z 96 [Note] Aborted connection 96 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T05:29:24.573237Z 34 [Note] Aborted connection 34 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T05:29:24.573406Z 35 [Note] Aborted connection 35 to db: 'dev' user: 'root' host: 'localhost' (Got an error reading communication packets)
2020-04-13T09:49:37.346162Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 4528ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-04-14T01:35:39.190069Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 43183898ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-04-14T07:30:03.500507Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 9873804ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
2020-04-15T02:39:20.019686Z 0 [Note] InnoDB: page_cleaner: 1000ms intended loop took 46274005ms. The settings might not be optimal. (flushed=0 and evicted=0, during the time.)
```

### 慢查询日志
慢查询日志可以帮助DBA定位存在问题的SQL语句，从而实现SQL语句层面的优化。可以通过`long_query_time`来设置慢查询阈值。默认值为10.
默认情况下，MySQL不开启慢查询日志，需要手动将`log_slow_queries`设置为ON。另一个和慢查询相关的参数`log_queries_not_using_indexes`，这个参数如果是ON，就会将运行的SQL语句没有使用索引的，记录到慢查询日志中。
MySQL 5.6.5中新增一个参数`log_throttle_queries_not_using_indexes`，用来表示每分钟允许记录到慢查询日志且未使用索引的SQL语句次数。默认为0，表示没有限制。

如果用户洗完得到执行时间最长的10条SQL语句，可以使用`mysqldumpslow -s -al -n 10 slow.log`。

InnoSQL加强了对SQL语句的捕获方式，在原版的基础上在慢查询日志上增加了对于逻辑读取和物理读取的统计。逻辑读取是指从磁盘进行IO读取的次数，逻辑读取包含磁盘和缓冲池的读取。可以通过参数`long_query_io`将超过指定逻辑IO次数的SQL语句记录到慢查询日志中，该值默认为100。为了兼容原MySQL运行方式，增加了参数`slow_query_type`，用来表示启用慢查询日志的方式，可选值如下：
- 0表示不将SQL语句记录到slow log
- 1表示根据运行时间将SQL语句记录到slow log
- 2表示根据逻辑IO次数将SQL语句记录到slow log
- 3表示根据运行时间以及逻辑IO次数将SQL语句记录到slow log


### 查询日志
查询日志记录了所有对MySQL请求的信息，无论请求是否正确执行。默认文件名：主机名.log。


### 二进制日志
二进制日志记录了对MySQL执行更改的所有操作，但不包括SELECT和SHOW这类操作。
二进制日志包括了执行数据库更改操作的时间等其他额外信息，二进制日志主要有以下几种作用：
- 恢复：某些数据的恢复需要二进制日志，例如，一个数据库全备文件恢复后，用户通过二进制日志进行point-in-time的恢复
- 复制：通过复制和执行二进制日志使一台远程的MySQL数据库与另一台进行实时同步。
- 审计：用户通过二进制日志中的信息来进行审计，判断是否有对数据库进行注入攻击。

通过配置参数`log-bin[=name]`可以启动二进制日志，如果不指定name，则默认名为主机名，后缀名为二进制日志的序列号，所在路径为数据库所在目录。
二进制日志文件在默认情况没有启动，需要手动指定参数来启动。开启二进制日志对性能下降1%左右。以下配置的参数影响二进制日志记录的信息和行为：

- max_binlog_size
    - 指定了单个二进制日志文件的最大值，如果超过该值，则产生新的二进制日志文件，后缀名+1,MySQL5.0开始默认值为1073741824(1G)
- binlog_cache_size
    - 当使用事务的引擎（如InnoDB）,所有未提交的二进制日志会被记录到一个缓存中去，等事务提交时将缓存中的日志写入二进制日志文件，缓冲大小由该参数指定，默认为32K，该参数基于会话，一个事务分配一个缓存。
- sync_log
    - 默认情况下，二进制日志并不是每次写的时候同步到磁盘（缓冲写），因此如果数据库宕机，还会有一部分数据没有写入日志文件。该参数值N，表示每缓冲写多少次就同步到磁盘。默认值为0，设置为1表示不用缓存，直接进行磁盘写，但是有个问题，在COMMIT之前，部分操作已经写入日志，此时宕机，那么下次启动，由于COMMIT未发生，这个事务会被回滚，但是日志文件不能回滚。该问题通过`innodb_support_xa`设置为1来解决
- binlog-do-db/binlog-ignore-db
    - 表示需要写入或忽略写入哪些库的日志，默认为空，表示需要同步所有库的日志到二进制日志。 
- log-slave-update
    - 如果当前数据库为slave角色，则它不会将从master取得并执行的二进制日志写入自己二进制日志文件，如需写入，设置该参数。
- binlog_format
    - 影响记录二进制日志的格式，可设置的值有`STATEMENT`、`ROW`、`MIXED`
    - `STATEMENT`记录的是日志的逻辑SQL语句
    - `ROW`记录行更改情况，设置为`ROW`可以将InnoDB的事务隔离级别基本设为`READ_COMMITED`，以获得更好的并发性。
    - `MIXED`默认使用`STATEMENT`格式存储，会在一些情况下使用`ROW`格式存储，可能的情况有：
        - 表存储引擎为NDB，
        - 使用了UUID()、USER()、CURRENT_USER()、FOUND_ROWS()、ROW_COUNT()等不确定函数
        - 使用了INSERT DELAY语句
        - 使用用户定义函数
        - 使用了临时表
    - 通常情况下，采用`ROW`格式存储，这可以为数据库的恢复和复制带来更好的可靠性，但是会导致文件大小增加

二进制日志文件采用`mysqlbinlog`查看日志内容。


### 套接字文件

在UNIX系统下本地连接MySQL可以采用UNIX域套接字方式连接，一般位于`/tmp`目录下，名为`mysql.sock`。

### pid文件
当MySQL实例启动后，会将自己的进程ID写入一个文件，该文件文件名为`主机名.pid`

### 表结构定义文件
MySQL数据的存储时根据表进行的，每个表都有对应的文件，frm后缀的文件记录了该表的表结构定义，frm文件哈用来存放视图的定义。


### InnoDB存储引擎文件

#### 表空间文件
InnoDB采用将存储的数据俺好表空间进行存放的设计，在默认配置下会有一个初始大小为10MB，名为ibdata1的文件，该文件就是默认的表空间文件，可以通过`innodb_data_file_path`进行设置。如果设置了`innodb_file_per_table`，则可以将每个基于InnoDB引擎的表产生一个独立表空间。命名规则为`表名.ibd`。


#### 重做日志文件
在默认情况下，InnoDB存储引擎的目录下会有名为`ib_logfile0`和`ib_logfile1`的文件，这就是重做日志文件（redo log file）。它们记录了对于InnoDB存储引擎的事务日志。当数据库因为掉电导致实例失败，InnoDB存储引擎可以通过重做日志恢复到掉电前的时刻，来保证数据完整性。


每个InnoDB存储引擎至少有一个重做日志组，每个文件组下至少有两个重做日志文件，设置多个日志组，可以提高可靠性。

以下参数影响重做日志的属性：
- innodb_log_file_size
    - 指定每个重做日志文件的大小。 
- innodb_log_files_in_group
    - 指定了日志文件组中重做日志文件的数量，默认为2 
- innodb_mirrored_log_groups
    - 指定了日志镜像文件组的数量，默认为1，表示只有一个日志文件组，没有镜像 
- innodb_log_group_home_dir
    - 指定了日志文件所在的路径，默认为`./`

重做日志的大小设置对于InnoDB存储引擎的性能有着很大的影响，过大的话再恢复的时候需要很长时间，过小的话，会导致一个事务的日志需要切换重做日志文件与频繁地发生`async checkpoint`。

重做日志文件与二进制日志文件的区别：
1. 二进制日志会记录所有于MySQL数据库有关的日志记录，包括所有引擎的日志，而重做日志文件只存储和InnoDB相关的事务日志。
2. 二进制日志文件记录的是关于一个事务的具体操作内容，重做日志记录的是关于每个页的更改的物理情况。
3. 写入的时间也不同，二进制日志文件仅在事务提交前进行提交，只写磁盘一次。重做日志在事务期间会不断的写。

重做日志的条目结构由四部分组成：
- redo_log_type占用1字节，表示重做日志的类型。
- space表示表空间的ID，采用压缩的方式，占用空间小于4字节
- page_no表示页的偏移量，采用压缩的方式
- redo_log_body表示每个重做日志的数据部分，恢复时需要调用相应的函数进行解析。

写入重做日志文件不是直接写，而是写入一个日志缓冲，然后按照顺序写入日志文件。流程如图所示：
![重做日志写入过程](https://blog-1251613845.cos.ap-shanghai.myqcloud.com/mysql/file/redo%E8%BF%87%E7%A8%8B.png)

从缓冲往磁盘写入时，写入最小单位为扇区，也就是512字节，因此可以保证写入是成功的。
触发磁盘写的过程由`innodb_flush_log_at_trx_commit`控制，表示在提交操作时，处理重做日志的方式，共有0，1.2三种取值。
- 0：代表提交事务时，并不将事务的重做日志写入磁盘上的日志文件，而是等待主线程每秒的刷新。
- 1：表示在执行commit时间重做日志缓冲同步写道到磁盘
- 2：表示间重做日志异步写到磁盘

为了保证事务的ACID中的持久性，必须将该参数设置为1，也就是每当有事务提交，就必须保证事务都已经写入重做日志文件中。如果数据库发生宕机，可以通过重做日志恢复已经提交的事务，如果设置为0或2，会导致部分事务的丢失。


## 参考
《MySQL技术内幕：InnoDB存储引擎(第二版)》