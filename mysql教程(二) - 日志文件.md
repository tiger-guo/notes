---
title: mysql教程(二) - 日志文件
date: 2022-02-23
tags:
    - mysql
categories:
    - mysql
---

## Mysql日志文件
Mysql 中有以下几种重要的日志模块，分别是：

- redo log 
- bin log
- undo log

## redo log
### redo log是什么？
Mysql 每一次的更新操作如果都要从磁盘读取数据，并且将每次的更新操作都写入磁盘的话，整个过程的IO成本、查找成本都很高。所以，为了解决这个问题，引入了 WAL 技术（Write-Ahead Logging），它的关键点就是先写日志（redo log），再写磁盘。

具体来说，当有一条记录需要更新的时候，InnoDB引擎就会把记录写到redo log buffer，后续某个时间点再一次性将多个操作记录写到 redo log file 中（这个由参数 'innodb_flush_log_at_trx_commit' 控制），并更新内存，这个时候更新就算执行完成了。同时，InnoDB引擎会在适当的时候，将这个操作更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。

此外 redo log 还有确保事务的持久性的作用，如果发生故障的时间点，尚有脏页未写入磁盘导致数据丢失问题。在重启Mysql服务的时候，会根据redo log进行重做，恢复奔溃时暂未写入磁盘的数据，从而达到事务的持久性这一特性。

redo log 是 InnoDB 引擎特有的；是物理日志，记录的是“在某个数据页上做了什么修改”；redo log 是循环写的，空间固定会用完。

### redo log 工作原理
InnoDB 的 redo log 是固定大小的，比如可以配置为一组4个文件，每个文件的大小是1GB。从头开始写，写到末尾就又回到开头循环写，如下图所示：

![](./images/mysql/redo_log.png)

write pos 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。check point 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。write pos 和 check point 之间是还空着的部分，可以用来记录新的操作。如果 write pos 追上 check point，表示redo log满了，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 check point 推进一下。有了 redo log，InnoDB 就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为 crash-safe。

### innodb_flush_log_at_trx_commit
innodb_flush_log_at_trx_commit 控制 redo log buffer 写入 redo log file 的时机，其参数值含义如下：

| 参数值 | 含义 |
| --- | --- |
| 0 | 事务提交时不会将 redo log buffer 中日志写入到 os buffer ，而是每秒写入 os buffer 并调用 fsync() 写入到 redo log file 中。也就是说设置为0时是(大约)每秒刷新写入到磁盘中的，当系统崩溃，会丢失1秒钟的数据。 |
| 1 | 事务每次提交都会将 redo log buffer 中的日志写入 os buffer 并调用 fsync() 刷到 redo log file 中。这种方式即使系统崩溃也不会丢失任何数据，但是因为每次提交都写入磁盘，IO的性能较差。 |
| 2 | 每次提交都仅写入到 os buffer ，然后是每秒调用 fsync() 将 os buffer 中的日志写入到 redo log file 。 |

![](./images/mysql/redo_log_sync.png)

### 物理文件
默认情况下，对应的物理文件位于数据库的 data 目录下的ib_logfile1、ib_logfile2。

相关参数如下：

- innodb_log_files_in_group – 指定文件个数（默认2）
- innodb_log_file_size – 指定文件大小
- innodb_mirrored_log_groups – 指定日志文件副本个数，主要用于保护数据（默认1）

## bin log
### bin log是什么？
bin log是Mysql Server层的逻辑日志，bin log中存储的内容称之为事件，每一个数据库更新操作（Insert、Update、Delete，不包含Select）等都对应一个时间（event）。简单的说 bin log 中存储的是更新数据库的SQL语句，但又不完全是SQL语句这么简单，bin log 中同时包含了用户执行的SQL语句（增删改）的反向SQL语句。

比如：delete 操作在 binlog 会有 delete本身和其反向的 insert 这两条记录；update 则同时存储着update执行前后的版本的信息；insert则对应着delete和insert本身的信息。

因此可以基于 binlog 做到类似于 oracle 的闪回功能，其实都是依赖于 binlog 中的日志记录。

### 工作原理
事务提交的时候，会一次性将事务中的 SQL 语句（一个事务可能对应多个 SQL 语句）按照一定的格式记录到 binlog 中。但是对 binlog 来说，较大事务的提交可能会变得比较慢一些，这是因为 binlog 是在事务提交的时候一次性写入的。

### 物理文件
binlog 文件存放路径由参数 log_bin_basename 控制，binlog 日志文件按照指定大小，当日志文件达到指定的最大的大小之后，进行滚动更新生成新的日志文件（如：binlog.0001,binlog.0002）。

*对于每个 binlog 文件，会通过一个统一的 index（目录）文件来组织。*

- expire_logs_days - bin log 文件存放时间，默认为0，即不自动移除。
- log_bin_basename – bin log 文件存放路径
- max_binlog_size – 单个 bin log 文件最大大小（默认 1G）

## bin log 与 redo log 区别
**作用层面**
- bin log 是 Mysql Server 层的日志
- redo log 是 InnoDB 引擎层的日志


**存储内容**
- bin log 是逻辑日志，存储了执行了那个更新语句，以及这个更新语句的回滚语句。
- redo log 是物理日志，存储了在某个数据页上面做了什么修改。


**日志覆盖方式**
- bin log 单文件是有最大限制的，达到最大上限后会滚的更新，但整体大小无限制。
- redo log 空间是固定的，只能循环写。

## 参考
- https://time.geekbang.org/column/article/68633
- https://www.lixueduan.com/post/mysql/07-binlog-redolog-undolog/
- https://segmentfault.com/a/1190000023827696