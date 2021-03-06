---
title: Storm Hive 集成
layout: documentation
documentation: true
---

  Hive 提供了 streaming API, 它允许将数据连续地写入 Hive.
  传入的数据可以用小批量 record 的方式连续提交到现有的 Hive partition 或 table 中.
  一旦提交了数据，它就可以立即显示给所有的 hive 查询.
  有关 Hive Streaming API 的更多信息请参阅 <https://cwiki.apache.org/confluence/display/Hive/Streaming+Data+Ingest>

  在 Hive Streaming API 的帮助下, HiveBolt 和 HiveState 允许用户将 Storm 中的数据直接传输到 Hive 中.
  要使用 Hive streaming API, 用户需要创建一个使用了 ORC 格式的 bucketed table.
  如下所示
  
  ```sql
  create table test_table ( id INT, name STRING, phone STRING, street STRING) partitioned by (city STRING, state STRING) stored as orc tblproperties ("orc.compress"="NONE");
  ```
  

## HiveBolt (org.apache.storm.hive.bolt.HiveBolt)

HiveBolt 控制 tuples 直接流入到 Hive 中.
使用 Hive 事务写入 Tuples.
HiveBolt 将流式传输的分区可以创建或预先创建，或者也可用 HiveBolt 来创建它们，如果它们不存在的话.

```java
DelimitedRecordHiveMapper mapper = new DelimitedRecordHiveMapper()
            .withColumnFields(new Fields(colNames));
HiveOptions hiveOptions = new HiveOptions(metaStoreURI,dbName,tblName,mapper);
HiveBolt hiveBolt = new HiveBolt(hiveOptions);
```

### RecordHiveMapper
   该 class 将 Tuple 的字段名映射到 Hive table 的列名.
   
   + DelimitedRecordHiveMapper (org.apache.storm.hive.bolt.mapper.DelimitedRecordHiveMapper)
   + JsonRecordHiveMapper (org.apache.storm.hive.bolt.mapper.JsonRecordHiveMapper)
   
   ```java
   DelimitedRecordHiveMapper mapper = new DelimitedRecordHiveMapper()
            .withColumnFields(new Fields(colNames))
            .withPartitionFields(new Fields(partNames));
    or
   DelimitedRecordHiveMapper mapper = new DelimitedRecordHiveMapper()
            .withColumnFields(new Fields(colNames))
            .withTimeAsPartitionField("YYYY/MM/DD");
   ```

|Arg（参数） | Description（描述） | Type（类型）
|--- |--- |---
|withColumnFields| tuple 中要被映射到 table 列名的字段名称 | Fields (必需的) |
|withPartitionFields| tuple 中要被映射到 hive table partition 的字段名称 | Fields |
|withTimeAsPartitionField| 用户可以使用系统时间作为 hive table 的 partition | String . Date format|

### HiveOptions (org.apache.storm.hive.common.HiveOptions)

HiveBolt 将 HiveOptions 作为一个构造参数.

  ```java
  HiveOptions hiveOptions = new HiveOptions(metaStoreURI,dbName,tblName,mapper)
                                .withTxnsPerBatch(10)
                				.withBatchSize(1000)
                	     		.withIdleTimeout(10)
  ```


HiveOptions 参数

|Arg（参数）  |Description（描述） | Type（类型）
|---	|--- |---
|metaStoreURI | hive meta store URI (可以在 hive-site.xml 中找到) | String (必需的) |
|dbName | 数据库名 | String (必需的) |
|tblName | 表名 | String (必需的) |
|mapper| Mapper class, 映射 Tuple 的字段名称到 Table 的列名称 | DelimitedRecordHiveMapper 或 JsonRecordHiveMapper (必需的) |
|withTxnsPerBatch | Hive 向 HiveBolt 的流客户端授予 *一批事务* 而不是单个事务. 此设置配置每个事务批处理所需的事务数. 来自单个批次中所有事务的数据最终在单个文件中. Flume 将在批处理中的每个事务中写入最大的 batchSize 事件. 与 batchSize 配合使用的设置可以控制每个文件的大小. 请注意, 最终 Hive 将透明地将这些文件压缩成较大的文件. | Integer . 默认 100 |
|withMaxOpenConnections| 只允许这个数量的 open connections. 如果超过该数量, 则最近最少使用的 connection 将被 closed.| Integer . 默认 100|
|withBatchSize| 在单个 Hive 事务中写入 Hive 的最大事件数 | Integer. 默认 15000|
|withCallTimeout| (In milliseconds) 针对 Hive & HDFS I/O operations 的超时, 例如 openTxn, write, commit, abort. | Integer. 默认 10000|
|withHeartBeatInterval| (In seconds) Interval between consecutive heartbeats sent to Hive to keep unused transactions from expiring. Set this value to 0 to disable heartbeats.| Integer. 默认 240 |
|withAutoCreatePartitions| HiveBolt 将自动创建必要的 Hive partition 以流式传输. |Boolean. 默认 true |
|withKerberosPrinicipal| Kerberos user principal 用于安全的访问 Hive | String|
|withKerberosKeytab| Kerberos keytab 用户安全的访问 Hive | String |
|withTickTupleInterval| (In seconds) 如果 > 0, 则 Hive Bolt 将定期刷新事务批次. 建议启用此功能, 以避免在等待批次阻塞时出现元组超时. | Integer. 默认 0|


 
## HiveState (org.apache.storm.hive.trident.HiveTrident)

Hive Trident state 也遵循 HiveBolt 类似的模式, 它以 HiveOptions 作为参数.

```java
   DelimitedRecordHiveMapper mapper = new DelimitedRecordHiveMapper()
            .withColumnFields(new Fields(colNames))
            .withTimeAsPartitionField("YYYY/MM/DD");
            
   HiveOptions hiveOptions = new HiveOptions(metaStoreURI,dbName,tblName,mapper)
                                .withTxnsPerBatch(10)
                				.withBatchSize(1000)
                	     		.withIdleTimeout(10)
                	     		
   StateFactory factory = new HiveStateFactory().withOptions(hiveOptions);
   TridentState state = stream.partitionPersist(factory, hiveFields, new HiveUpdater(), new Fields());
 ```