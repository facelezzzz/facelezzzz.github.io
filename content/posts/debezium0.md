---
title: 数据一致性:云上的Debezium实践
description: 在云上部署Debezium避免mysql和redis双重写入问题
date: "2021-09-18"
tags: ["Debezium","数据一致性"]
---



最近在处理呼叫中心的控制组件时（该组件提供简单的API允许用户发起外部呼叫请求（指使用机器人对某人打电话））碰到一个问题，在过去，控制组件接收用户的提交的呼叫数据后写入mysql，然后控制组件从mysql扫描提交的数据并写入redis，在数据量很小的情况下，这种模式工作的很好，然而随着数据量快速上升，扫描变得越来越慢，开始影响到用户。

呼叫数据表一共有15列，其中有些关键的字段，例如:CREATE_TIME（创建时间）/STATE（状态，用来标记是否已经写入redis）/PHONE（用户的号码，可能是加密号）/ID（偏序自增的标识，作为主键，由发号器生成）/PAYLOAD（载体，必要的信息，可能会很长）。

在旧的模式下，定时任务从表中根据STATE扫描出未进入redis的数据，然后提交到redis队列中，当更改成功时修改每条数据的STATE字段，随着业务量上升，数据量开始快速膨胀（大约亿级每月），根据CREATE_TIME扫描的速度开始变慢，同时由于扫描耗时增长（数据库负载增加），也影响到插入，系统开始变得缓慢，需要优化。

有一些看上去不太可靠的方式：a.直接向redis写入呼叫数据，然后从redis同步数据到mysql（写入mysql是必要的，后续有其他组件依赖于此）；b.先向kafka写入呼叫数据，然后分别同步到redis和kafka; c.使用cdc工具 d.写入mysql时同时写入redis

a首先被否决，因为如果用户请求超时，则没有好的办法判断请求的状态（失败？成功写入redis？写入redis成功但是同步mysql异常？），此外从redis同步到mysql并没有合适的解决方案，此外在呼叫中心上，重复呼叫是需要避免的，系统需要提供幂性。

b其实看上去可以解决问题，但是还是缺失事务，因为提交呼叫数据时，有其他的同事务操作，当然可以通过一些手段避免事务，但是会增大复杂性。

d其实最简单，但是存在双写问题。

最后c方案被选择，CDC看上去完美符合需求，保持原来的写入逻辑，将扫描操作移除，改为由CDC捕获binlog同步变化到redis。

存在很多CDC工具，如canal，maxwell，debezium等，在一番测试后，最终确定了debezium。理由是debezium更符合需要，开源，由红帽支持，默认情况下使用kafka connect（解决了高可用的问题），此外写入到kafka能直接被下游的系统消费。

## 在云上部署
官网提供Demo在Docker上运行简单的示例，这里提供在k8s上部署的示例。

### 制作镜像:
在debezium官网下载debezium-connector-mysql-1.5.4.Final-plugin.tar

基于confluentinc公司的kafka connect构建需要的镜像：
DOCKERFILE如下：
```
FROM confluentinc/cp-kafka-connect:6.2.0
ADD debezium-connector-mysql-1.5.4.Final-plugin.tar /plugins/

```
构建完成后推送到必要的仓库。

### 制作Deployment
参照https://docs.confluent.io/platform/current/installation/docker/config-reference.html

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-kafka-connector-debezium
  namespace: kxjl
  labels:
    app: test-kafka-connector-debezium
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-kafka-connector-debezium
  template:
    metadata:
      labels:
        app: test-kafka-connector-debezium
        logging: 'true'
    spec:
      containers:
        - name: test-kafka-connector-debezium
          image: "<your_private_Harbor>/test/cp-kafka-connect-debezium:1.6.1"
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          env:
            - name: CONNECT_BOOTSTRAP_SERVERS
              value: <kafka_cluter_addrs>
            - name: CONNECT_REST_PORT
              value: '<rest_port>'
            - name: CONNECT_GROUP_ID
              value: test-debezium-connect-group
            - name: CONNECT_CONFIG_STORAGE_TOPIC
              value: test-debezium-connect-config-storage
            - name: CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR
              value: '<factor_for_replacation>'
            - name: CONNECT_OFFSET_STORAGE_TOPIC
              value: test-debezium-connect-offset-storage
            - name: CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR
              value: 'factor_for_replacation'
            - name: CONNECT_STATUS_STORAGE_TOPIC
              value: test-debezium-connect-status-storage
            - name: CONNECT_STATUS_STORAGE_REPLICATION_FACTOR
              value: 'factor_for_replacation'
            - name: CONNECT_KEY_CONVERTER
              value: org.apache.kafka.connect.json.JsonConverter
            - name: CONNECT_VALUE_CONVERTER
              value: org.apache.kafka.connect.json.JsonConverter
            - name: CONNECT_INTERNAL_KEY_CONVERTER
              value: org.apache.kafka.connect.json.JsonConverter
            - name: CONNECT_INTERNAL_VALUE_CONVERTER
              value: org.apache.kafka.connect.json.JsonConverter
            - name: CONNECT_REST_ADVERTISED_HOST_NAME
              value: localhost
            - name: CONNECT_PLUGIN_PATH
              value: /plugins
            - name: CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE
              value: 'false'
            - name: CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE
              value: 'false'
```
通过设定CONNECT_REST_ADVERTISED_HOST_NAME的值localhost避免将connect RESTAPI端点暴露出去，如果不需要安全限制，可以去除该配置。

设定CONNECT_KEY_CONVERTER_SCHEMAS_ENABLE和CONNECT_VALUE_CONVERTER_SCHEMAS_ENABLE为false避免在topic中出现冗余的schema信息（可选，视需求而定）。

### 创建debezium connector
将deployment文件应用后，3个节点的kafka connect应该启动完成，接下来将创建debezium connector。

首先更改mysql配置（binlog_format=ROW的情况下才能被debezium正常捕获）
```
server-id         = 2324
log_bin           = mysql-bin
binlog_format     = ROW
binlog_row_image  = FULL
expire_logs_days  = 90
```

然后在mysql中创建debezium用户：
```
CREATE USER debezium IDENTIFIED BY '<your_password>';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'qebezium'@'%';
FLUSH PRIVILEGES;
```

最后根据https://docs.confluent.io/platform/current/connect/references/restapi.html
准备创建debezium connect。

```
{
    "name": "test-kafka-connector-debezium",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "snapshot.locking.mode": "none",
        "transforms.unwarp.add.fields": "op,table",
        "database.user": "debezium",
        "transforms.unwrap.delete.handling.mode": "rewrite",
        "database.server.id": "2134",
        "tasks.max": "1",
        "database.history.kafka.bootstrap.servers": "<kafka_cluter_addrs>",
        "database.history.kafka.topic": "schema-changes.test-kafka-connector-debezium",
        "transforms": "unwarp,Reroute",
        "database.server.name": "test-kafka-connector-debezium-slave",
        "database.port": "3306",
        "database.hostname": "<mysql_host>",
        "database.password": "<mysql_password>",
        "name": "test-kafka-connector-debezium",
        "transforms.unwarp.type": "io.debezium.transforms.ExtractNewRecordState",
        "table.include.list": "test.tb_data.*",
        "database.include.list": "test",
        "transforms.Reroute.type": "io.debezium.transforms.ByLogicalTableRouter",
        "transforms.Reroute.topic.regex": "(.*)_shard_(.*)",
        "transforms.Reroute.topic.replacement": "$1"
    }
}
```
根据debezium文档，connector有几个配置需要注意，1.database.server.id表明debezium作为mysql的slave运行，注意不要和其他的slave id冲突 2.database.history.kafka.topic 保存了DDL和binlog点位信息，不能设置过期否则会报错（可以设定snapshot.mode=schema_only_recovery）3.transforms指定unwarp后可以
丢弃同步到kafka中的一些冗余信息，比如before after等，参考https://debezium.io/documentation/reference/transformations/event-flattening.html
。指定Reroute通常是为了将分表数据推送到相同的topic，以上的配置会将test.tb_data_shard_1,test.tb_data_shard_2...推送到相同的topic。

最后，向kafka connect提交创建请求：
```
curl --location --request POST '<ip>:<port>/connectors' \
--header 'Content-Type: application/json' \
--data '{
    "name": "test-kafka-connector-debezium",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "snapshot.locking.mode": "none",
        "transforms.unwarp.add.fields": "op,table",
        "database.user": "debezium",
        "transforms.unwrap.delete.handling.mode": "rewrite",
        "database.server.id": "2134",
        "tasks.max": "1",
        "database.history.kafka.bootstrap.servers": "<kafka_cluter_addrs>",
        "database.history.kafka.topic": "schema-changes.test-kafka-connector-debezium",
        "transforms": "unwarp,Reroute",
        "database.server.name": "test-kafka-connector-debezium-slave",
        "database.port": "3306",
        "database.hostname": "<mysql_host>",
        "database.password": "<mysql_password>",
        "name": "test-kafka-connector-debezium",
        "transforms.unwarp.type": "io.debezium.transforms.ExtractNewRecordState",
        "table.include.list": "test.tb_data.*",
        "database.include.list": "test",
        "transforms.Reroute.type": "io.debezium.transforms.ByLogicalTableRouter",
        "transforms.Reroute.topic.regex": "(.*)_shard_(.*)",
        "transforms.Reroute.topic.replacement": "$1"
    }
}'
```

现在向表中写入数据后，可以从对应的topic中立即读取。注意例子中的topic由database.server.name、table.include.list、Reroute规则3部分组成，以上例子的topic是“test-kafka-connector-debezium-slave.test.tb_data”。





