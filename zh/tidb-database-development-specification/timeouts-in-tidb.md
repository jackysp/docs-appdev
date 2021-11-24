---
title: TiDB 数据库开发规范 - TiDB 中的各种超时
summary: 介绍 TiDB 中的各种超时设置和使用方式。
---

# TiDB 中的各种超时

## 1. GC 超时

TiDB 的事务的实现采用了 MVCC（多版本并发控制）机制，当新写入的数据覆盖旧的数据时，旧的数据不会被替换掉，而是与新写入的数据同时保留，并以时间戳来区分版本。TiDB 通过定期 GC 的机制来清理不再需要的旧数据。

默认配置下 TiDB 可以保障每个 MVCC 版本（一致性快照）保存 10 分钟，读取时间超过 10 分钟的事务，会收到报错 `GC life time is shorter than transaction duration`

当用户确信自己需要更长的读取时间时，比如在使用了 Mydumper 做全量备份的场景中（Mydumper 备份的是一致性的快照），可以通过调整 TiDB 中
`mysql.tidb` 表中的 `tikv_gc_life_time` 的值来调大 MVCC 版本保留时间，需要注意的是 tikv_gc_life_time 的配置是立刻影响全局的，调大它会为当前所有存在的快照增加生命时长，调小它会立即缩短所有快照的生命时长。过多的 MVCC 版本会拖慢 TiKV 的处理效率，在使用 Mydumper 做完全量备份后需要及时把 tikv_gc_life_time 调整回之前的设置。

更多关于 GC 的信息，请参考官网文档：<https://pingcap.com/docs-cn/stable/reference/garbage-collection/overview/>

## 2. 事务超时

含 DML 语句的事务，除了受 `tikv_gc_life_time` 限制之外，还受到另外一个参数 `max-txn-time-use` 的影响，这个参数位于 tidb-server 的配置文件 tidb.toml 中，用于控制单个事务允许的最大执行时间。该参数的默认值为 590（秒），需要注意必须控制该参数的值小于 `tikv_gc_life_time` 的值。

形如 `insert into t10 select * from t1` 的 SQL 语句，即使执行时间没有达到 `tikv_gc_life_time` 限制，但超过了 `max-txn-time-use` 的限制，会由于超时而回滚。

## 3. SQL 执行时间超时

TiDB 还提供了一个系统变量来限制单条 SQL 语句的执行时间：max_execution_time，它的默认值为 0，表示无限制。`max_execution_time` 目前对所有类型的 statement 生效，并非只对 SELECT 语句生效。其单位为 ms，但实际精度在 100ms 级别，而非更准确的毫秒级别。

## 4. JDBC 查询超时

MySQL jdbc 的查询超时设置 `setQueryTimeout()` 对 TiDB 不起作用。这是因为现实客户端感知超时时，向数据库发送一个 KILL 命令。但是由于 tidb-server 是负载均衡的， 为防止在错误的 tidb-server 上终止连接， tidb-server 不会执行这个 KILL。这时就要用 `MAX_EXECUTION_TIME` 实现查询超时的效果。

TiDB 提供了三个与 MySQL 兼容的超时控制参数：

1. `wait_timeout`,控制与 Java 应用连接的非交互式空闲超时时间, 默认值为0，即允许连接无限闲置。

2. `interactive_timeout`，控制与 Java 应用连接的交互式空闲超时时间，默认值为8小时。

3. `max_execution_time`，控制连接中 SQL 执行的超时时间，默认值是 0，即允许连接无限忙碌（一个 SQL 语句执行无限的长的时间）。

但在实际生产环境中，空闲连接和一直无限执行的 SQL 对数据库和应用都有不好的影响。你可以通过在应用的连接字符串中配置这两个 session 级的变量来避免空闲连接和执行时间过长的 SQL 语句。例如，设置 `sessionVariables=wait_timeout=3600（1 小时）`和  `sessionVariables=max_execution_time=300000（5 分钟）`。