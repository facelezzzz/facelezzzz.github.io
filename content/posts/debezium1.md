---

title: CDC: debezium 常见问题
description: debezium 可能会碰到的问题和对应的解决方式
date: "2021-06-18"
tags: ["Debezium","数据一致性"]

----

## 1.Mysql HA拓扑故障转移后，debezium失效

严格意义讲这其实不算是debezium的问题。因为从原理上，debezium只是订阅了某个节点的binlog文件，并重放到kafka。
所以对于HA拓扑下，如果没有启用mysql的GTID，则debezium始终跟随其中的一个节点(binlog加position(可以简单理解为物理点位))，如果该节点失效，需要等待该节点
恢复。

在生产上可以启动mysql的GTID（全局事物标识）,这样debezium可以使用事务标识（可以简单理解为逻辑点位）而非之前的position,当debezium当前跟随的
节点故障后，随着故障转移，可以通过重启debezium任务让debezium在另一个活跃的节点上正常工作（通过GTID匹配另一节点的binlog）。

## 2.DB实例锁定
默认的debezium在第一次初始化时，执行
snapshot.mode=initial

在官网的文档中，其第一步就是”Grabs a global read lock that blocks writes by other database clients. “，所以在
initial模式且进行的是第一次初始化时，debezium首先会获取全局读锁一直阻塞客户端对整个db的写，直到读取需要监视的表的schema后
才会释放锁。

在启用GTID的情况下可以为需要捕获的节点新增一个slave,等slave同步后，让debezium订阅该slave，当snapshot执行完成后，下掉这个临时的节点，并
重启启动debezium任务。

## 3.重启失败
之前我们碰到过一个特别的情况，日志如下

---
[2021-11-22 03:52:40,126] INFO Snapshot step 2 - Determining captured tables (io.debezium.relational.RelationalSnapshotChangeEventSource)
[2021-11-22 03:52:40,126] INFO Read list of available databases (io.debezium.connector.mysql.MySqlSnapshotChangeEventSource)
[2021-11-22 03:52:40,128] INFO Read list of available tables in each database (io.debezium.connector.mysql.MySqlSnapshotChangeEventSource)
...
[2021-11-22 03:52:40,155] INFO Snapshot step 4 - Determining snapshot offset (io.debezium.relational.RelationalSnapshotChangeEventSource)
[2021-11-22 03:52:40,157] INFO Read binlog position of MySQL primary server (io.debezium.connector.mysql.MySqlSnapshotChangeEventSource)
[2021-11-22 03:52:40,159] INFO Snapshot - Final stage (io.debezium.pipeline.source.AbstractSnapshotChangeEventSource)
[2021-11-22 03:52:40,159] ERROR Producer failure (io.debezium.pipeline.ErrorHandler)
io.debezium.DebeziumException: io.debezium.DebeziumException: Cannot read the binlog filename and position via 'SHOW MASTER STATUS'. Make sure your server is correctly configured
at io.debezium.pipeline.source.AbstractSnapshotChangeEventSource.execute(AbstractSnapshotChangeEventSource.java:82)
at io.debezium.pipeline.ChangeEventSourceCoordinator.lambda$start$0(ChangeEventSourceCoordinator.java:110)
at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
at java.base/java.lang.Thread.run(Thread.java:829)
Caused by: io.debezium.DebeziumException: Cannot read the binlog filename and position via 'SHOW MASTER STATUS'. Make sure your server is correctly configured
at io.debezium.connector.mysql.MySqlSnapshotChangeEventSource.lambda$determineSnapshotOffset$4(MySqlSnapshotChangeEventSource.java:293)
at io.debezium.jdbc.JdbcConnection.query(JdbcConnection.java:557)
at io.debezium.jdbc.JdbcConnection.query(JdbcConnection.java:498)
at io.debezium.connector.mysql.MySqlSnapshotChangeEventSource.determineSnapshotOffset(MySqlSnapshotChangeEventSource.java:276)
at io.debezium.relational.RelationalSnapshotChangeEventSource.doExecute(RelationalSnapshotChangeEventSource.java:119)
at io.debezium.pipeline.source.AbstractSnapshotChangeEventSource.execute(AbstractSnapshotChangeEventSource.java:71)
... 6 more
[2021-11-22 03:52:40,594] INFO WorkerSourceTask{id=connector-xxx} flushing 0 outstanding messages for offset commit (org.apache.kafka.connect.runtime.WorkerSourceTask)
[2021-11-22 03:52:40,595] ERROR WorkerSourceTask{id=connector-xxx} Task threw an uncaught and unrecoverable exception. Task is being killed and will not recover until manually restarted (org.apache.kafka.connect.runtime.WorkerTask)
org.apache.kafka.connect.errors.ConnectException: An exception occurred in the change event producer. This connector will be stopped.
at io.debezium.pipeline.ErrorHandler.setProducerThrowable(ErrorHandler.java:42)
at io.debezium.pipeline.ChangeEventSourceCoordinator.lambda$start$0(ChangeEventSourceCoordinator.java:127)
at java.base/java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:515)
at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
at java.base/java.lang.Thread.run(Thread.java:829)
Caused by: io.debezium.DebeziumException: io.debezium.DebeziumException: Cannot read the binlog filename and position via 'SHOW MASTER STATUS'. Make sure your server is correctly configured
at io.debezium.pipeline.source.AbstractSnapshotChangeEventSource.execute(AbstractSnapshotChangeEventSource.java:82)
at io.debezium.pipeline.ChangeEventSourceCoordinator.lambda$start$0(ChangeEventSourceCoordinator.java:110)
... 5 more
Caused by: io.debezium.DebeziumException: Cannot read the binlog filename and position via 'SHOW MASTER STATUS'. Make sure your server is correctly configured
at io.debezium.connector.mysql.MySqlSnapshotChangeEventSource.lambda$determineSnapshotOffset$4(MySqlSnapshotChangeEventSource.java:293)
at io.debezium.jdbc.JdbcConnection.query(JdbcConnection.java:557)
at io.debezium.jdbc.JdbcConnection.query(JdbcConnection.java:498)
at io.debezium.connector.mysql.MySqlSnapshotChangeEventSource.determineSnapshotOffset(MySqlSnapshotChangeEventSource.java:276)
at io.debezium.relational.RelationalSnapshotChangeEventSource.doExecute(RelationalSnapshotChangeEventSource.java:119)
at io.debezium.pipeline.source.AbstractSnapshotChangeEventSource.execute(AbstractSnapshotChangeEventSource.java:71)
... 6 more
[2021-11-22 03:52:40,595] INFO Stopping down connector (io.debezium.connector.common.BaseSourceTask)
---
当前是运维在维护mysql时进行了误操作，将某个mysql的新的binlog给删了，然后节点也临时下了个，执行debezium重启后报错，实际上报错就是
在新的节点上找不到GTID对于的binlog记录，可以通过创建连接器或者恢复binlog来修复这个问题。
