---
layout:     post
title:      MySQL 基础知识汇总
subtitle:   学习笔记
date:       2019-06-10
author:     sincosmos
header-img: img/post-bg-map.jpg
catalog: true
tags:
    - MySQL
---

## 物理文件组成
1. 日志文件：error log (hostname.err), binlog (mysql-bin.******, mysql-bin.index), slow query log (hostname-slow.log), redo log(Inndo 事务日志，配合表中 undo 信息，保证事务安全性）
2. 数据文件
   1) MyISAM 存储引擎：table-name.frm (metadata of the table), table-name.MYD (MyISAM 表数据文件），table-name.MYI（MyISAM 表索引文件，非聚簇索引）；
   2) InnoDB 存储引擎：table-name.frm (metadata of the table), table-name.ibd（表独享数据文件时）或 .ibdata (表共享数据文件时)，InnoDB 的主键索引是聚簇索引，辅助索引是非聚簇索引，索引和数据都存储在表文件里。  
   如果不单独配置，则可以在 mysql 安装目录 $MYSQL_HOME/data 下查看到上述文件
3. Replication 相关文件：master.info (保存在 slave 端数据目录下，存放 master 的主机地址/端口、连接用户/密码，当前日志位置，已读取到的日志位置信息)，mysql-relay-bin.xxxxxn （存放 slave I/O 线程从 Master 端读取的 binlog 信息，然后由 Slave 端的 SQL 线程解析并转化成 Master 端执行的 SQL，从而可以在 slave 同步执行），mysql-relay-bin.index (记录relay log 存放的绝对路径)，relay-log.info  
   一般情况下，需要设置打开相应日志，mysql 才会记录相应的日志。对于 binlog 来说，在 mysql 的配置文件 my.cnf 中添加 
   ```
   log-bin=mysql-bin
   binlog_format="MIXED"
   ```
   即可开启 binlog 记录。目前 mysql 支持三种 binlog 格式，ROW, STATEMENT 和 MIXED。
   master 会在数据目录下生成 mysql-bin.******(* 代表 0～9 数字，表示日志的序号) 日志文件，同时生成 mysql-bin.index 文件，记录所有 binlog 的绝对路径，便于其它线程能顺利找到所需的 binlog。slave 端启动线程从 master 同步 binary log，并存放为 mysql-relay-bin.****** 文件，slave 端的 SQL 线程读取并解析 binlog，转化为可执行的 SQL 进行同步操作。
4. 其它文件：my.cnf, pid file (linux), socket file (linux)
## 逻辑架构
总的来说，Mysql 可以看成是二层架构，第一层通常叫做 SQL layer，在 MySQL 数据库系统处理底层数据之前的所有工作都在这一层完成，包括权限判断、SQL 解析、执行计划优化、Query cache 的处理等；第二层是存储引擎层，也就是底层数据存取操作实现，由多种存储引擎组成。
还可以把 SQL layer 层拆分为服务层和核心层，服务层主要负责和用户的交互工作，例如连接处理、授权认证、安全等；核心层主要负责数据存取预处理，例如SQL 解析、查询缓存、执行计划优化等。
可以通过下图来了解，当一条 SQL 发送到 mysql 服务器后是怎样被执行的。
![sql 的执行过程](https://user-gold-cdn.xitu.io/2018/8/12/1652e56415e9a6f4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
Mysql 缓存由`query_cache_type`参数控制打开，由 `query_cache_size` 控制缓存可用空间大小，`query_cache_limit` 控制每次能够缓存的最大结果集(`show variables like '%cache%'`)。可以通过 `show status like '%cache%'`中 `Qcache_hits` 和 `Qcache_inserts` 等来查看缓存的使用状态。MySQL将缓存存放在一个引用表（不要理解成table，可以认为是类似于HashMap的数据结构），通过一个哈希值索引，这个哈希值通过查询本身、当前要查询的数据库、客户端协议版本号等一些可能影响结果的信息计算得来。所以两个查询在任何字符上的不同（例如：空格、注释），都会导致缓存不会命中。如果查询中包含任何用户自定义函数、存储函数、用户变量、临时表、mysql库中的系统表，其查询结果都不会被缓存。比如函数NOW()或者CURRENT_DATE()会因为不同的查询时间，返回不同的查询结果，再比如包含CURRENT_USER或者CONNECION_ID()的查询语句会因为不同的用户而返回不同的结果，将这样的查询结果缓存起来没有任何的意义。既然是缓存，就会失效，那查询缓存何时失效呢？MySQL的查询缓存系统会跟踪查询中涉及的每个表，如果这些表（数据或结构）发生变化，那么和这张表相关的所有缓存数据都将失效。正因为如此，在任何的写操作时，MySQL必须将对应表的所有缓存都设置为失效。
[MySQL逻辑架构及性能优化原理](https://juejin.im/post/5c3ef9e051882525dc62de87)
## 线程池与连接池
连接池通常实现在 client 端，指应用预先创建一定数量的数据库连接，利用这些连接服务客户端所有的 DB 请求。如果某一时间，应用所需的连接数大于连接池中的连接数，则请求需要排队等待空闲连接。通过连接池可以复用连接，避免应用频繁地创建和释放连接，从而减少请求的平均响应时间。并且在数据库繁忙时，可以在客户端实现请求排队，减小数据库压力。  
线程池在 server 端实现，数据库服务器通过创建一定数量的线程服务与 DB 连接请求
## 存储引擎与索引原理
[存储引擎与索引原理](sincosmos.github.io/)
## 锁与事务
[锁与事务](sincosmos.github.io/)

## Join 的实现原理
在 MySQL 中，只有一中 Join 算法，即 Nested Loop Join（其它数据库还提供的有 Hash Join 和 Sort Merge Join），实际上就是通过驱动表的结果集（通过 where 条件过滤）作为循环基础数据，然后一条一条的通过该结果集中的数据作为过滤条件到下一个表中查询数据，然后合并结果，如果还有第三个表参与 Join，则再通过前两个表的 Join 结果集作为循环基础数据，再一次通过循环查询条件到第三个表中查询数据，如此往复。
## order by 与 limit
如果根据索引字段进行 order by，mysql 可以不进行排序直接返回有序的结果。
```
# date_created 字段上有索引，
select * from sites order by date_created desc limit 100, 10;
# 如果 category_id 分布均匀，为了避免扫描大量数据，下面的情况下我们最好在 category_id 列上建立索引
select * from sites where category_id=5 order by date_created desc limit 10;
```
如果排序字段上无索引，mysql 提供两种排序实现方式。
1) 首先从表中取出满足过滤条件的用于排序的字段和可以直接定位到行数据的行指针，在 Sort Buffer 中进行实际的排序操作，然后利用排好序的行指针回表查询，取得其它字段，返回客户端
2) 根据过滤条件一次取出排序字段和客户端需要的所有其它字段，将不需要排序的字段数据放在一块内存区域中，然后在 Sort Buffer 中将排序字段和行指针进行排序，最后利用排序后的行指与存放在内存区域中的其它字段进行匹配，将结果按照顺序返回给客户端。这种方式减少了回表磁盘 IO 操作，但会占用更多的内存，是典型的以空间换时间的方式。
如果要对 Join 后的结果进行排序操作，如果排序字段仅在 Join 的驱动表中，那么可以在获得驱动表的结果集时在 Sort Buffer 中进行排序，利用排序后的结果集与第二个表进行 Nested Loop Join。但如果排序的多个字段出现在 join 的两个表中，那就不能利用 Sort Buffer 进行排序，而是必须先将 Join 的结果集存放到一个临时表中，再把临时表中的数据抽取到 Sort Buffer 中进行排序。
在无索引的字段上进行排序并加上 limit 条件限制时，MySQL 5.6 开始采用了堆排序的优化算法，虽然参与排序的元素不会受到影响，但是 sort buffer 所需的空间变小许多。

## 主从数据库
1)  随着用户量的增多，我们可以将数据库的读写分离，使用主库（Master)负责写，若干个从库与主库同步更新数据，用户从从库读数据。写操作发生后，从库同步主库会有一定的延迟。在程序实现方面，可以借助 Spring AOP 组件实现写主库，读请求读从库。当只有读操作的时候，直接操作读库（从库），当在写事务（即写主库）中读时，强制走从库，即先暂停写事务，开启读（读从库），然后恢复写事务。此方案其实是使用事务传播行为为：NOT_SUPPORTS解决的。
## 分库分表
1）随着用户量的大幅度增多和历史数据的积累，一个 master 不能满足高并发的写需求，而全量的数据放在一个表/数据库里也会因为数据量过大导致查询性能下降，这时候我们就要考虑分库分表了。
分库主要解决单个数据库性能问题。垂直分库在微服务盛行的今天非常受欢迎。基本的思路就是按照业务模块划分出来不同的数据库，而不是像早期那样将所有的数据表都放在同一个数据库中。例如电商网站中，订单表、用户表、商品表等都分别放在不同的数据库里。不同的数据库可能位于不同的机器上。
分表一般有水平拆分和垂直拆分两种分法。水平分表就是将某个表的数据按行拆分，分别保存到不同的表中，例如 id（主键） 是 1 ～ 1000万的行在一个表，id 是 1000万 ～ 2000万的行在下一个表，依次拆分。根据表数据的大小，这些表都在同一个数据库（不建议使用），也可以位于不同的数据库中（即分库分表同时使用）。当然，实际水平分表的策略可能是通过主键或者时间等字段进行 hash 和取模后进行拆分。垂直拆分指将数据表的列拆分到不同的表中，使拆分后的每个表拥有较少的列，通常可以把较大的字段单独保存到一个表中，常用的字段和不常用的字段也分开到不同的表中。
如果是因为表多而数据多，使用垂直切分，根据业务切分成不同的库。如果是因为单张表的数据量太大，这时要用水平切分，即把表的数据按某种规则切分成多张表，甚至多个库上的多张表。 分库分表的顺序应该是先垂直分，后水平分。 因为垂直分更简单，更符合我们处理现实世界问题的方式。
分库分表后，应当尽量避免跨库 join，可以采用增加数据冗余的方式来解决该问题。
