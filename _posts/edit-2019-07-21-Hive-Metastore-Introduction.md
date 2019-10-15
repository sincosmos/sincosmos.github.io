---
 layout:     post
 title:      Hive Metastore
 subtitle:   学习笔记
 date:       2019-07-21
 author:     sincosmos
 header-img: img/post-bg-BJJ.jpg
 catalog: true
 tags:
    - Hive
---
# Hive metastore
Hive 为存储在分布式文件存储系统上的数据提供一层类似数据库表的结构信息(元信息，metadata)，以便用户以 Hive SQL 的方式访问相应的数据。这里的分布式文件存储系统最基本的是 HDFS，也支持例如 Amazon S3 这样的云存储系统。
Hive 将元信息例如 schema, partition 等存放到一个 hive 自用的数据库中，并通过 metastore service 对外提供服务。这个自用的数据库可以是内嵌的 derby（默认，只能单进程访问，只作测试用）、外部数据库例如 mysql。启动 hive 时，默认在同一个 jvm 中启动了 metastore service，默认状态下，客户端将通过该 metastore service 连接内嵌的 derby 进行服务。 
Hive metastore 有两个核心组成部分：
- 一个将 metadata 暴露出去的服务，metastore service
- 存储 metadata 的数据库
1. 如前述，默认状态下，hive 会使用内置的 metastore service 和 derby 提供服务，同一时间只允许一个客户端连接，这种方式称为**内嵌式 Metastore**
2. 如果使用外部数据库（通过配置）替换掉内置的 derby，可以允许多个客户端同时连接 hive 服务，这种方式称为**Local Metastore**。使用 Mysql 时，首先要把 mysql JDBC dirver 放入到 hive 的 lib 下。然后更改 hive-site.xml 配置，将 `javax.jdo.option.ConnectionURL` 改为 `jdbc:mysql://host/dbname? createDatabaseIfNotExist=true`，将 `javax.jdo.option.ConnectionDriverName` 改为 `com.mysql.jdbc.Driver`，另外也需要配置访问数据的用户名和密码。
3. 实际应用中，我们一般采用**Remote Metastore**的运行模式，这种模式下 Metastore servcie 运行在单独的 JVM 中，不和 Hive Service 服务进程在一起。客户端通过 metastore service 提供的 thrift 服务连接 metastore。metastore server 的隔离可以通过配置生效，更改 hive-site.xml 将 `hive.metastore.uris` 设置为对应 service 的 `thrift://host:port` 形式。在部署 metastore service 的机器上执行 `hive --service metastore -p <port>` 命令即可启动 metastore service 并提供服务。

hive 客户端基于 thrift；beeline 客户端是 JDBC 客户端，需要连接 hiveserver2 才能执行查询


fs.defaultFS 指定 hive 数据默认存储系统，可以是 hdfs://, file:// (default), s3a:// 等等,  hive.metastore.warehouse.dir 指定 hive 数据在上述存储系统的存储路径。
