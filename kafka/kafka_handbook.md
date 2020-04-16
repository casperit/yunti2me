# kafka handbook

[1. topic](#1-topic)

* [1.1 topic相关操作](#11-topic相关操作)
  * [1.1.1 创建topic](#111-创建topic)
  * [1.1.2 list topic](#112-list-topic)
  * [1.1.3 describe topic](#113-describe-topic)
  * [1.1.4 为topic增加partition](#114-为topic增加partition)
  * [1.1.5 Balancing leadership](#115-balancing-leadershipkafka-preferred-replica-election-replica-election)
  * [1.1.6 Reassign Partitions](#116-reassign-partitions)
  * [1.1.7 删除topic](#117-删除topic)
* [1.2 topic配置选项操作](#12-topic配置选项操作)
* [1.3 topic相关配置选项](#13-topic相关配置选项)

[2. consumer相关操作](#2-consumer相关操作)

* [2.1 console consume](#21-console-consumer)
* [2.2 Checking consumer position](22-checking-consumer-position)
* [2.3 Managing Consumer Groups](#23-managing-consumer-groups)

[3. producer相关操作](#3-producer相关操作)

* [3.1 console producer](#31-console-producer)

[4. 集群运维相关操作](#4-集群运维相关操作)

* [4.1 启动单个kafka进程](#41-启动单个kafka进程)
* [4.2 启动多broker kafka集群](#42-启动多broker-kafka集群)
* [4.3 优雅的停止Broker](#43-优雅的停止broker)
* [4.4 Balancing leadership](#44-balancing-leadershipkafka-preferred-replica-election-replica-election)
* [4.5 Checking consumer position](45-checking-consumer-position)
* [4.6 Managing Consumer Groups](#46-managing-consumer-groups)



## 1. topic<a name="1. topic"></a>

### 1.1 topic相关操作<a name="1.1 topic相关操作"></a>

默认情况下，如果`auto.create.topics.enable`在server的设置中为true，那么当向kafka中不存在的topic写入数据的时候会自从创建topic。自动创建的topic会包含默认数量的partition数，replication factor，并使用kafka默认的schema来做replica的assignment。

#### 1.1.1 创建topic<a name="1.1.1 创建topic"></a>

```bash
# 创建一个包含1个partition，每个partition只有1个replica(replication-factor)的topic
> bin/kafka-topics.sh --create \
                      --zookeeper localhost:2181 \
                      --replication-factor 1 
                      --partitions 1 \
                      --topic test

# 创建一个包含1个partition，每个partition只有3个replica(replication-factor)的topic
> bin/kafka-topics.sh --create \
                      --zookeeper localhost:2181 \
                      --replication-factor 3 \
                      --partitions 1 \
                      --topic my-replicated-topic
```



#### 1.1.2 list topic<a name="1.1.2 list topic"></a>

```bash
> bin/kafka-topics.sh --zookeeper localhost:9092 \
                      --list
test
```



#### 1.1.3 describe topic<a name="1.1.3 describe topic"></a>

```bash
# describe一个包含3个partition，每个partition有3个replica（replication-factor）的topic
> bin/kafka-topics.sh --zookeeper localhost:2181 \
                      --describe \
                      --topic test
Topic:test	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: test	Partition: 1	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
	Topic: test	Partition: 2	Leader: 0	Replicas: 0,1,2	Isr: 0,1,2
```



#### 1.1.4 为topic增加partition<a name="1.1.4 为topic增加partition"></a>

在kafka中，partition数量决定了其并发度，既一个topic中的messages是被分布在多个partitions中的，这些partition分别被不同的broker server存储。当创建一个topic的时候，该topic的partition数就已经被确定了。但当这个topic中的数据持续增加，后续可能需要为该topic增加更多的partitions。对topic进行增加partition的操作如下：

```shell
# Increase number of partitions for topic
> bin/kafka-topics.sh --alter --zookeeper localhost:2181 --topic topic1 --partitions 4
  
# Increase number of partitions with specific replica assignment
> bin/kafka-topics.sh --alter --zookeeper localhost:2181 --topic topic1 --replica-assignment 0:1:2,0:1:2,0:1:2,2:1:0 --partitions 4
```

这里需要注意的是：

```diff
- 增加partition并不会改变已经写入到原partitions中的数据的分布，也就是说并不会导致数据被重新shuffle。
- 所以如果消费者依赖于类似于`hash(key) % number_of_partitions`的数据分布策略，
- 那么可能会对增加partition后的数据消费产生疑惑。如果数据是按照这种取模这种算法方式向partitions中进行分布写入，
- 那么新的数据会按照新的partition数进行分布，但原来的数据是不会做任何的redistribution的。```
```



#### 1.1.5 Balancing leadership（kafka-preferred-replica-election-replica-election）<a name="1.1.5 Balancing leadership"></a>

当一个broker宕机，或重启以后，原先以这台broker为leader的partitions将会被转移到其他的broker上去。当这台broker重启以后，就没有任何一个partition的leader在这台机器上，也就不会服务于任何从client（producer或comsumer）来的读写操作，这样会造成这台机器过闲导致的负载不均衡问题。

为了避免这种不平衡，Kafka中有一个概念叫做`preferred replicas`，比如当一个partition的replicas列表是1，5，9的时候，node 1就是这个partition的preferred，因为它出现在replicas列表的第一个。当出现某个partition的`leader replica`跟replicas列表中的第一个broker id不一致的时候，说明现在这个partition的leader不是`preferred replica`，如以下情况：

```
Topic:test	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 1	Replicas: 0,1,2	Isr: 2,1,0
	Topic: test	Partition: 1	Leader: 1	Replicas: 0,1,2	Isr: 2,1,0
	Topic: test	Partition: 2	Leader: 1	Replicas: 0,1,2	Isr: 2,1,0
```

可以用一下命令来触发kafka集群对leadership的重新分配。

```shell
> bin/kafka-preferred-replica-election.sh --zookeeper zk_host:port/chroot
```

但是由于这是一个集群层面的操作，通常会运行的很慢，所以也可以通过在服务配置中加入以下配置来对这个过程进行自动执行：

```properties
auto.leader.rebalance.enable=true
```

另外，这个工具还可以通过增加`--path-to-json-file`参数来对控制对哪些preferred replica进行leader选举。这个选项后面需要跟一个json文件，该json文件中定义了哪些topics的哪些partition信息。如下示例：

```json
{
  "partitions": [
    {"topic": "foo","partition": 1},
    {"topic": "foobar","partition": 2}
  ]
}
```

或者

```json
{
 "partitions":
  [
    {"topic": "topic1", "partition": 0},
    {"topic": "topic1", "partition": 1},
    {"topic": "topic1", "partition": 2},
    {"topic": "topic2", "partition": 0},
    {"topic": "topic2", "partition": 1}
  ]
}
```

假设该json文件名为`preferred_replica_example_test.json`，那么运行命令如下；

```shell
bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181 \
                                        --path-to-json-file preferred_replica_example_test.json

Created preferred replica election path with test-0,test-1,test-2
Successfully started preferred replica election for partitions Set(test-0, test-1, test-2)
```

这里需要注意的是：当这个命令运行以后，并不是立刻就会让`preferred replica election`完成，而只是触发了这个进程开始，其后台工作步骤如下：

1. 这个命令更新zookeeper中的`/admin/preferred_replica_election`节点，将需要做leader调整到`preferred replica`的partition写入到这个节点中。
2. Controller会监听这个zk path，当有数据更新到这个zk path的时候就会触发选举，controller读取这个节点中的partition list
3. 对每一个partition，controller会读取它的preferred replica（assigned replica列表中的第一个），如果其`prefeered replcia`不是leader，并且是在isr列表中，那么controller就会发起一个request到hold preferred replica的broker，让其变成是leader。

* 如果preferred replica不在ISR列表中怎么办？

这种情况下controller的move leadership任务会失败。通常需要查看是否该replica在preferred broker上是否有数据丢失。当恢复到ISR列表后可以重新运行

* 如何确认操作的运行结果

可以使用list topic来查看topic和partitions的状态（leader，assigned replicas，in-sync等）如果每个partition的leader都跟其assigned replica中的第一个broker id一致，则表示成功。否则失败。

* 注意：`kafka-preferred-replica-election.sh`只是尝试去调整每个partition的preferred replica，并没有对replica的broker顺序或者broker列表（`Replicas: 0,1,2`）进行修改，如果想要对replica list进行修改，需要用`kafka-reassign-partitions.sh`工具



#### 1.1.6 Reassign Partitions<a name="1.1.6 Reassign Partitions"></a>

`Reassign partition`跟上述`Preferred Replica election`有点相似，都是为了对集群内的读写流量进行负载均衡。但和Preferred Replica election只对leader replicas进行均衡不一样的是，Reassign partition是可以对每个partition的assigned replicas（也就是partition的replicas broker是列表`Replicas: 0,1,2`）进行重新分配。这样做的原因是，虽然leader replica承担了数据的写入和被消费的流量，但其他的副本也是需要从leader进行数据的sync的，因此有时候仅仅对leader的分布进行balance还不够。

```shell
bin/kafka-reassign-partitions.sh
 
Option                                 Description
------                                 -----------
--bootstrap-server <String: Server(s)  the server(s) to use for
  to use for bootstrapping>              bootstrapping. REQUIRED if an
                                         absolution path of the log directory
                                         is specified for any replica in the
                                         reassignment json file
--broker-list <String: brokerlist>     The list of brokers to which the
                                         partitions need to be reassigned in
                                         the form "0,1,2". This is required
                                         if --topics-to-move-json-file is
                                         used to generate reassignment
                                         configuration
--disable-rack-aware                   Disable rack aware replica assignment
--execute                              Kick off the reassignment as specified
                                         by the --reassignment-json-file
                                         option.
--generate                             Generate a candidate partition
                                         reassignment configuration. Note
                                         that this only generates a candidate
                                         assignment, it does not execute it.
--reassignment-json-file <String:      The JSON file with the partition
  manual assignment json file path>      reassignment configurationThe format
                                         to use is -
                                       {"partitions":
                                        [{"topic": "foo",
                                          "partition": 1,
                                          "replicas": [1,2,3],
                                          "log_dirs": ["dir1","dir2","dir3"]
                                         }],
                                       "version":1
                                       }
                                       Note that "log_dirs" is optional. When
                                         it is specified, its length must
                                         equal the length of the replicas
                                         list. The value in this list can be
                                         either "any" or the absolution path
                                         of the log directory on the broker.
                                         If absolute log directory path is
                                         specified, it is currently required
                                         that the replica has not already
                                         been created on that broker. The
                                         replica will then be created in the
                                         specified log directory on the
                                         broker later.
--throttle <Long: throttle>            The movement of partitions will be
                                         throttled to this value (bytes/sec).
                                         Rerunning with this option, whilst a
                                         rebalance is in progress, will alter
                                         the throttle value. The throttle
                                         rate should be at least 1 KB/s.
                                         (default: -1)
--timeout <Long: timeout>              The maximum time in ms allowed to wait
                                         for partition reassignment execution
                                         to be successfully initiated
                                         (default: 10000)
--topics-to-move-json-file <String:    Generate a reassignment configuration
  topics to reassign json file path>     to move the partitions of the
                                         specified topics to the list of
                                         brokers specified by the --broker-
                                         list option. The format to use is -
                                       {"topics":
                                        [{"topic": "foo"},{"topic": "foo1"}],
                                       "version":1
                                       }
--verify                               Verify if the reassignment completed
                                         as specified by the --reassignment-
                                         json-file option. If there is a
                                         throttle engaged for the replicas
                                         specified, and the rebalance has
                                         completed, the throttle will be
                                         removed
--zookeeper <String: urls>             REQUIRED: The connection string for
                                         the zookeeper connection in the form
                                         host:port. Multiple URLS can be
                                         given to allow fail-over.
```

该工具有多种用法，比如：

* 将已有的topics/partitions分配到新加入集群的brokers上（或者集群中负载比较低的brokers上）

通常当给一个kafka集群增加brokers后，这些新加入的brokers上并没有任何流量，此时就可以用该工具进行partition reassign，将已有的topics/partitions分配到这些新brokers机器上。此时可以提供两个选项：1，需要进行move的topics；2，需要接受数据的brokers列表。使用这两个选项，此工具就会自动的选择相应的topics的partitions到新的brokers上，并产生一个`JSON`文件用来描述分配的详细策略。该`JSON`文件将要在下一个步骤（包含`--reassignment-json-file` 选项）

```shell
$ bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --broker-list "0,2,3" --topics-to-move-json-file /tmp/topic-to-move.json  --generate

Current partition replica assignment
{"version":1,"partitions":[{"topic":"test","partition":1,"replicas":[0,1,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":0,"replicas":[0,1,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":2,"replicas":[0,1,2],"log_dirs":["any","any","any"]}]}

Proposed partition reassignment configuration
{"version":1,"partitions":[{"topic":"test","partition":1,"replicas":[2,3,0],"log_dirs":["any","any","any"]},{"topic":"test","partition":0,"replicas":[0,2,3],"log_dirs":["any","any","any"]},{"topic":"test","partition":2,"replicas":[3,0,2],"log_dirs":["any","any","any"]}]}

$ cat /tmp/topic-to-move.json
{
  "topics": [
    {"topic": "test"}
  ],
  "version": 1
}
```

* 选择性的对一些partition调整到一些特定的brokers上

该partition movement工具还可以选择性的对一些partition的replica进行分配到特定的brokers上去。通常如果集群存在不均衡的情况，就可以使用这个工具进行人为的负载均衡调整。这里需要用到的是一个用来描述哪些topic的哪些partition需要调整到哪些brokers上的JSON文件，也可以在上一步generate的JSON文件的基础上做调整。

比如如下命令，对原本分布在“0,1,2”三台broker上的topic test进行重新分配，分配到“0,2,3”上，并进行leader的分布。

```shell
$ bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test
Topic:test	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 0	Replicas: 0,1,2	Isr: 2,0,1
	Topic: test	Partition: 1	Leader: 0	Replicas: 0,1,2	Isr: 2,0,1
	Topic: test	Partition: 2	Leader: 0	Replicas: 0,1,2	Isr: 2,0,1
	

$ cat partitions-to-move.json
{
  "version": 1,
  "partitions": [
    {
      "topic": "test",
      "partition": 1,
      "replicas": [2,3,0],
      "log_dirs": ["any","any","any"]
    },
    {
      "topic": "test",
      "partition": 0,
      "replicas": [0,2,3],
      "log_dirs": ["any","any","any"]
    },
    {
      "topic": "test",
      "partition": 2,
      "replicas": [3,0,2],
      "log_dirs": ["any","any","any"]
    }
  ]
}

$ bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --reassignment-json-file partitions-to-move.json --execute
Current partition replica assignment

{"version":1,"partitions":[{"topic":"test","partition":1,"replicas":[0,1,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":0,"replicas":[0,1,2],"log_dirs":["any","any","any"]},{"topic":"test","partition":2,"replicas":[0,1,2],"log_dirs":["any","any","any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started reassignment of partitions.

$ bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test
Topic:test	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 0	Replicas: 0,2,3	Isr: 2,0,3
	Topic: test	Partition: 1	Leader: 0	Replicas: 2,3,0	Isr: 2,0,3
	Topic: test	Partition: 2	Leader: 0	Replicas: 3,0,2	Isr: 2,0,3
	
# 此时发现test topic的replicas列表已经从原来的0,1,2变成了0,2,3，且不同partition的preferred replica也已经均匀分布在0，2和3的brokers上。只不过leader还都是在0上，现在只需要对test topic再进行一次preferred replicas election，就能把leader也调整过来。

$ cat preferred_replica_example_test.json
{
  "partitions": [
    {"topic": "test","partition": 0},
    {"topic": "test","partition": 1},
    {"topic": "test","partition": 2}
  ]
}

$ bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181 --path-to-json-file preferred_replica_example_test.json
Created preferred replica election path with test-0,test-1,test-2
Successfully started preferred replica election for partitions Set(test-0, test-1, test-2)

$ bin/kafka-topics.sh --zookeeper localhost:2181 --describe --topic test
Topic:test	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 0	Replicas: 0,2,3	Isr: 2,0,3
	Topic: test	Partition: 1	Leader: 2	Replicas: 2,3,0	Isr: 2,0,3
	Topic: test	Partition: 2	Leader: 3	Replicas: 3,0,2	Isr: 2,0,3

# 此时发现，test topic的三个partition，leader分别是0，2和3， replicas列表的排列也是按照相应顺序排列的了。
```

**注意该命令跟kafka-preferred-replica-election.sh一样，指示修改了zookeeper path的内容和存在性，后续的调整执行是有Controller来对partition的replicas进行异步的重新分配。**

**另外，该命令的默认run model是dry-run，并不是真正执行该操作，只有当加了`--execute`参数的时候才开始真正执行。**



#### 1.1.7 删除topic<a name="1.1.7 删除topic"></a>

当服务端（broker）设置`delete.topic.enable`为true时，topics是可以被kafka命令行工具删除的：

```shell
# Delete topic named topic1
bin/kafka-topics.sh --delete --zookeeper localhost:2181 --topic topic1
```



### 1.2 topic配置选项操作<a name="1.2 topic配置选项操作"></a>

Topic相关的配置选项可以在server的config中进行默认设置，当客户端发送命令的时候没有提供相应的配置选项的特殊设定，则会使用server端的默认值。

客户端提供相关选项的设置方式是通过`--config` 选项来设置，如下：

```bash
> bin/kafka-topics.sh --bootstrap-server localhost:9092 \
                      --create \
                      --topic my-topic \
                      --partitions 1 \
                      --replication-factor 1 \
                      --config max.message.bytes=64000 \
                      --config flush.messages=1
```

如果需要通过客户端对某些选项进行修改，则使用`kafka-configs.sh`的`--alter` 的`--add-config`设置：

```bash
> bin/kafka-configs.sh --zookeeper localhost:2181 \
                       --entity-type topics \
                       --entity-name my-topic \
                       --alter \
                       --add-config max.message.bytes=128000
```

确认修改是否生效可以用`--describe`如下方式进行查看：

```bash
> bin/kafka-configs.sh --zookeeper localhost:2181 \
                       --entity-type topics \
                       --entity-name my-topic \
                       --describe
```

如果需要对某些选项的设置进行删除，用`--delete-config`的方式，如下：

```bash
> bin/kafka-configs.sh --zookeeper localhost:2181  \
                       --entity-type topics \
                       --entity-name my-topic \
                       --alter \
                       --delete-config max.message.bytes
```



### 1.3 topic相关配置选项<a name="1.3 topic相关配置选项"></a>

kafka中的topic默认都有这些配置选项，如果没有单独做特殊设定，则会使用系统默认值。

* **cleanup.policy**

  ```shell
  A string that is either "delete" or "compact" or both. This string designates the retention policy to use on old log segments. 
  The default policy ("delete") will discard old segments when their retention time or size limit has been reached. 
  The "compact" setting will enable log compaction on the topic.
  
  * Type: list
  * Default: delete
  * Valid Values: [compact, delete]
  * Server Default Property: log.cleanup.policy
  * Importance: medium
  ```

* **compression.type**

```shell
Specify the final compression type for a given topic. 
This configuration accepts the standard compression codecs ('gzip', 'snappy', 'lz4', 'zstd'). 
It additionally accepts 'uncompressed' which is equivalent to no compression; 
and 'producer' which means retain the original compression codec set by the producer.

* Type: string
* Default: producer
* Valid Values: [uncompressed, zstd, lz4, snappy, gzip, producer]
* Server Default Property: compression.type
* Importance: medium
```

* **delete.retention.ms**

```shell
The amount of time to retain delete tombstone markers for log compacted topics. 
This setting also gives a bound on the time in which a consumer must complete a read 
if they begin from offset 0 to ensure that they get a valid snapshot of the final stage 
(otherwise delete tombstones may be collected before they complete their scan).

* Type: long
* Default: 86400000
* Valid Values: [0,...]
* Server Default Property: log.cleaner.delete.retention.ms
* Importance: medium
```

* **file.delete.delay.ms**

```shell
The time to wait before deleting a file from the filesystem

* Type: long
* Default: 60000
* Valid Values: [0,...]
* Server Default Property: log.segment.delete.delay.ms
* Importance: medium
```

* **flush.messages**

```SHELL
This setting allows specifying an interval at which we will force an fsync of data written to the log. 
For example if this was set to 1 we would fsync after every message; 
if it were 5 we would fsync after every five messages. 
In general we recommend you not set this and use replication for durability and allow the operating system's background flush capabilities 
as it is more efficient. This setting can be overridden on a per-topic basis (see the per-topic configuration section).

* Type: long
* Default: 9223372036854775807
* Valid Values: [0,...]
* Server Default Property: log.flush.interval.messages
* Importance: medium
```

* **flush.ms**

```shell
This setting allows specifying a time interval at which we will force an fsync of data written to the log. 
For example if this was set to 1000 we would fsync after 1000 ms had passed. 
In general we recommend you not set this and use replication for durability and allow the operating system's background flush capabilities as it is more efficient.

* Type: long
* Default: 9223372036854775807
* Valid Values: [0,...]
* Server Default Property: log.flush.interval.ms
* Importance: medium
```

* **follower.replication.throttled.replicas**

```shell
A list of replicas for which log replication should be throttled on the follower side. 
The list should describe a set of replicas in the form [PartitionId]:[BrokerId],[PartitionId]:[BrokerId]:... 
or alternatively the wildcard '*' can be used to throttle all replicas for this topic.

* Type: list
* Default: ""
* Valid Values: [partitionId]:[brokerId],[partitionId]:[brokerId],...
* Server Default Property: follower.replication.throttled.replicas
* Importance: medium
```

* **index.interval.bytes**

```shell
This setting controls how frequently Kafka adds an index entry to its offset index. 
The default setting ensures that we index a message roughly every 4096 bytes. 
More indexing allows reads to jump closer to the exact position in the log but makes the index larger. 
You probably don't need to change this.

* Type: int
* Default: 4096
* Valid Values: [0,...]
* Server Default Property: log.index.interval.bytes
* Importance: medium

```

* **leader.replication.throttled.replicas**

```shell
A list of replicas for which log replication should be throttled on the leader side. 
The list should describe a set of replicas in the form [PartitionId]:[BrokerId],[PartitionId]:[BrokerId]:... 
or alternatively the wildcard '*' can be used to throttle all replicas for this topic.

* Type: list
* Default: ""
* Valid Values: [partitionId]:[brokerId],[partitionId]:[brokerId],...
* Server Default Property: leader.replication.throttled.replicas
* Importance: medium
```

* **max.compaction.lag.ms**

```shell
The maximum time a message will remain ineligible for compaction in the log. 
Only applicable for logs that are being compacted.

* Type: long
* Default: 9223372036854775807
* Valid Values: [1,...]
* Server Default Property: log.cleaner.max.compaction.lag.ms
* Importance: medium
```

* **max.message.bytes**

```shell
The largest record batch size allowed by Kafka. 
If this is increased and there are consumers older than 0.10.2, the consumers' fetch size must also be increased so that the they can fetch record batches this large. 
In the latest message format version, records are always grouped into batches for efficiency. 
In previous message format versions, uncompressed records are not grouped into batches and this limit only applies to a single record in that case.

* Type: int
* Default: 1000012
* Valid Values: [0,...]
* Server Default Property: message.max.bytes
* Importance: medium

```

* **message.format.version**

```shell
Specify the message format version the broker will use to append messages to the logs. 
The value should be a valid ApiVersion. Some examples are: 0.8.2, 0.9.0.0, 0.10.0, check ApiVersion for more details. 
By setting a particular message format version, the user is certifying that all the existing messages on disk are smaller or equal than the specified version. 
Setting this value incorrectly will cause consumers with older versions to break as they will receive messages with a format that they don't understand.

* Type: string
* Default: 2.4-IV1
* Valid Values: [0.8.0, 0.8.1, 0.8.2, 0.9.0, 0.10.0-IV0, 0.10.0-IV1, 0.10.1-IV0, 0.10.1-IV1, 0.10.1-IV2, 0.10.2-IV0, 0.11.0-IV0, 0.11.0-IV1, 0.11.0-IV2, 1.0-IV0, 1.1-IV0, 2.0-IV0, 2.0-IV1, 2.1-IV0, 2.1-IV1, 2.1-IV2, 2.2-IV0, 2.2-IV1, 2.3-IV0, 2.3-IV1, 2.4-IV0, 2.4-IV1]
* Server Default Property: log.message.format.version
* Importance: medium
```

* **message.timestamp.difference.max.ms**

```shell
The maximum difference allowed between the timestamp when a broker receives a message and the timestamp specified in the message. 
If message.timestamp.type=CreateTime, a message will be rejected if the difference in timestamp exceeds this threshold. 
This configuration is ignored if message.timestamp.type=LogAppendTime.

* Type: long
* Default: 9223372036854775807
* Valid Values: [0,...]
* Server Default Property: log.message.timestamp.difference.max.ms
* Importance: medium
```

* **message.timestamp.type**

```shell
Define whether the timestamp in the message is message create time or log append time. 
The value should be either `CreateTime` or `LogAppendTime`

* Type: string
* Default: CreateTime
* Valid Values: [CreateTime, LogAppendTime]
* Server Default Property: log.message.timestamp.type
* Importance: medium
```

* **min.cleanable.dirty.ratio**

```shell
This configuration controls how frequently the log compactor will attempt to clean the log (assuming log compaction is enabled). 
By default we will avoid cleaning a log where more than 50% of the log has been compacted. 
This ratio bounds the maximum space wasted in the log by duplicates (at 50% at most 50% of the log could be duplicates). 
A higher ratio will mean fewer, more efficient cleanings but will mean more wasted space in the log. 
If the max.compaction.lag.ms or the min.compaction.lag.ms configurations are also specified, 
then the log compactor considers the log to be eligible for compaction as soon as either: 
(i) the dirty ratio threshold has been met and the log has had dirty (uncompacted) records for at least the min.compaction.lag.ms duration, or 
(ii) if the log has had dirty (uncompacted) records for at most the max.compaction.lag.ms period.

* Type: double
* Default: 0.5
* Valid Values: [0,...,1]
* Server Default Property: log.cleaner.min.cleanable.ratio
* Importance: medium
```

* **min.compaction.lag.ms**

```shell
The minimum time a message will remain uncompacted in the log. 
Only applicable for logs that are being compacted.

* Type: long
* Default: 0
* Valid Values: [0,...]
* Server Default Property: log.cleaner.min.compaction.lag.ms
* Importance: medium
```

* **min.insync.replicas**

```shell
When a producer sets acks to "all" (or "-1"), this configuration specifies the minimum number 
of replicas that must acknowledge a write for the write to be considered successful. 
If this minimum cannot be met, then the producer will raise an exception (either 
NotEnoughReplicas or NotEnoughReplicasAfterAppend).
When used together, min.insync.replicas and acks allow you to enforce greater durability guarantees. 
A typical scenario would be to create a topic with a replication factor of 3, set 
min.insync.replicas to 2, and produce with acks of "all". 
This will ensure that the producer raises an exception if a majority of replicas 
do not receive a write.

* Type: int
* Default: 1
* Valid Values: [1,...]
* Server Default Property: min.insync.replicas
* Importance: medium

```

* **preallocate**

```shell
True if we should preallocate the file on disk when creating a new log segment.

* Type: boolean
* Default: false
* Valid Values:
* Server Default Property: log.preallocate
* Importance: medium
```

* **retention.ms**

```shell
This configuration controls the maximum time we will retain a log before we will discard old 
log segments to free up space if we are using the "delete" retention policy. 
This represents an SLA on how soon consumers must read their data. If set to -1, 
no time limit is applied.

* Type: long
* Default: 604800000  #(默认7天)
* Valid Values: [-1,...]
* Server Default Property: log.retention.ms
* Importance: medium
```

* **segment.bytes**

```shell
This configuration controls the segment file size for the log. 
Retention and cleaning is always done a file at a time so a larger segment size means fewer files but less granular control over retention.

* Type: int
* Default: 1073741824
* Valid Values: [14,...]
* Server Default Property: log.segment.bytes
* Importance: medium
```

* **segment.ms**

```shell
This configuration controls the period of time after which Kafka will force the log to roll 
even if the segment file isn't full to ensure that retention can delete or compact old data.

* Type: long
* Default: 604800000
* Valid Values: [1,...]
* Server Default Property: log.roll.ms
* Importance: medium
```

*  **unclean.leader.election.enable**

```shell
Indicates whether to enable replicas not in the ISR set to be elected as leader as a last resort, even though doing so may result in data loss.

* Type: boolean
* Default: false
* Valid Values:
* Server Default Property: unclean.leader.election.enable
* Importance: medium
```





## 2. consumer相关操作<a name="2. consumer相关操作"></a>

##### 2.1 console consumer<a name="2.1 console consumer"></a>

```bash
> bin/kafka-console-consumer.sh  --topic test \
                                 --bootstrap-server localhost:9094 \ #0.10.0及以后的版本这里可以直接提供一台或者多台的broker:port即可，kafka足够智能，知道去哪里找对应的meta信息。不需要写--zookeeper
                                 --from-beginning
```

##### 2.2 Checking consumer position<a name="2.2 Checking consumer position"></a>

对consumer group的消费进度查询，能够查询到当前的消费offset，离end还有多远，有多大gap。

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --list
Note: This will not show information about old Zookeeper-based consumers.
console-consumer-89834
console-consumer-52688
console-consumer-87916

$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --describe --group console-consumer-52688
Note: This will not show information about old Zookeeper-based consumers.

TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                     HOST            CLIENT-ID
tpch.part       0          3394600         20000000        16605400        consumer-1-9eecce42-95f8-4069-9a42-8c289d68c446 /192.168.1.71   consumer-1
```

也可以使用该工具来对consumer的offset进行reset。如下。



##### 2.3 Managing Consumer Groups<a name="2.3 Managing Consumer Groups"></a>

使用consumer group管理命令，可以对consumer group进行list，describe，delete。consumer group可以在kafka端被手动的删除，或者当该group的最后一次offset的commit时间过期以后，被自动删除。只有当consumer group中没有active的consumer手动删除能成功。

对consumer group list操作如下：

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --list
Note: This will not show information about old Zookeeper-based consumers.
console-consumer-89834
console-consumer-52688
console-consumer-87916
```

要查看consumer group的消费offset详情，用describe：

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --describe --group console-consumer-52688
Note: This will not show information about old Zookeeper-based consumers.

TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                     HOST            CLIENT-ID
tpch.part       0          3394600         20000000        16605400        consumer-1-9eecce42-95f8-4069-9a42-8c289d68c446 /192.168.1.71   consumer-1
```

describe有一些命令行辅助参数能够帮助提供更多详细的信息：

* `--members`: 输出consumer group中的active consumers列表

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group --members
 
CONSUMER-ID                                    HOST            CLIENT-ID       #PARTITIONS
consumer1-3fc8d6f1-581a-4472-bdf3-3515b4aee8c1 /127.0.0.1      consumer1       2
consumer4-117fe4d3-c6c1-4178-8ee9-eb4a3954bee0 /127.0.0.1      consumer4       1
consumer2-e76ea8c3-5d30-4299-9005-47eb41f3d3c4 /127.0.0.1      consumer2       3
consumer3-ecea43e4-1f01-479f-8349-f9130b75d8ee /127.0.0.1      consumer3       0
```

* `--members --verbose`： 除了在members的输出信息基础上，进一步输出每个partition被assign到每个consumer的详情

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-group --members --verbose
 
CONSUMER-ID                                    HOST            CLIENT-ID       #PARTITIONS     ASSIGNMENT
consumer1-3fc8d6f1-581a-4472-bdf3-3515b4aee8c1 /127.0.0.1      consumer1       2               topic1(0), topic2(0)
consumer4-117fe4d3-c6c1-4178-8ee9-eb4a3954bee0 /127.0.0.1      consumer4       1               topic3(2)
consumer2-e76ea8c3-5d30-4299-9005-47eb41f3d3c4 /127.0.0.1      consumer2       3               topic2(1), topic3(0,1)
consumer3-ecea43e4-1f01-479f-8349-f9130b75d8ee /127.0.0.1      consumer3       0               -
```

* `--state`：该选项提供一些group level的信息

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --describe --group console-consumer-52688 --state
Note: This will not show information about old Zookeeper-based consumers.
Consumer group 'console-consumer-52688' has no active members.

COORDINATOR (ID)          ASSIGNMENT-STRATEGY       STATE                #MEMBERS
192.168.1.71:9092 (0)                               Empty                0
```

如果要对consumer group在kafka中的信息进行删除，使用`--delete`：

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --delete --group console-consumer-87916  --group console-consumer-89834
Note: This will not show information about old Zookeeper-based consumers.
Deletion of requested consumer groups ('console-consumer-89834', 'console-consumer-87916') was successful.
```

Consumer Group工具还可以对consumer group的消费offset进行reset。使用`--reset-offsets`选项。reset offsets时必须选择一下两个范围的其中一个：`--all-topics` 或 `--topic`，除非使用了`--from-file`来进行精准定义（这里的file需要是一个CSV文件，用来提供topics和offsets的详细配置）。另外，需要对consumer group的offsets进行reset，需要确保consumer instances都已经处于inactive状态，也就是已经没有消费者在进行消费了。

`--reset-offsets`有三种执行模式：

- --dry-run(default)： 仅仅用来显示哪些offsets需要被reset
- --execute : 执行--reset-offsets
- --export : 输出执行结果到一个CSV文件

`--reset-offsets`有一下一些可供选择的执行场景（执行时至少选择一种）:

- --to-datetime <String: datetime> : 将offsets设置到某一个时间戳. Format: 'YYYY-MM-DDTHH:mm:SS.sss'
- --to-earliest : reset到最开头.
- --to-latest : reset到末尾.
- --shift-by <Long: number-of-offsets> : 将offset调整到当前位置的N步以外，可以是之前或者之后，分别用正数和负数表示。
- --from-file : 用一个CSV文件，用来提供topics和offsets的详细信息
- --to-current : reset offset到current。
- --by-duration <String: duration> : 将offset设置到当前时间的某个时间间隔以外. Format: 'PnDTnHnMnS'
- --to-offset : 直接设置到某个offset.

如果将offset设到了超出范围，则会被自动调整到末尾。比如：

```shell
$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --reset-offsets --group console-consumer-52688 --topic tpch.part --to-latest
WARN: In a future major release, the default behavior of this command will be to prompt the user before executing the reset rather than doing a dry run. You should add the --dry-run option explicitly if you are scripting this command and want to keep the current default behavior without prompting.
Note: This will not show information about old Zookeeper-based consumers.

TOPIC                          PARTITION  NEW-OFFSET
tpch.part                      0          20000000

$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --describe --group console-consumer-52688
Note: This will not show information about old Zookeeper-based consumers.
Consumer group 'console-consumer-52688' has no active members.

TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
tpch.part       0          3887600         20000000        16112400        -               -               -
# 以上describe会发现实际的offset并没有被重置，这是因为默认的运行方式是--dry-run(没有加--execute)，如果需要真正执行reset，需要加上--execute，如下例

$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --reset-offsets --group console-consumer-52688 --topic tpch.part --to-latest --execute
Note: This will not show information about old Zookeeper-based consumers.

TOPIC                          PARTITION  NEW-OFFSET
tpch.part                      0          20000000

$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9095  --describe --group console-consumer-52688
Note: This will not show information about old Zookeeper-based consumers.
Consumer group 'console-consumer-52688' has no active members.

TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
tpch.part       0          20000000        20000000        0               -               -               -
# 加了--execute后，offset被重置
```





## 3. producer相关操作<a name="3. producer相关操作"></a>

#####  3.1 console producer<a name="3.1 console producer"></a>

```bash
> bin/kafka-console-producer.sh --broker-list localhost:9092 \
                                --topic test
This is a message
This is another message
```

## 4. 集群运维相关操作<a name="4. 集群运维相关操作"></a>

##### 4.1 启动单个kafka进程<a name="4.1 启动单个kafka进程"></a>

```bash
# 这里需要确保
# 1, zookeeper服务(单点or集群)已经启动
# 2, config/server.properties中zk相关的配置已经正确设置
bin/kafka-server-start.sh config/server.properties 
```

##### 4.2 启动多broker kafka集群<a name="4.2 启动多broker kafka集群"></a>

这里是用一台机器上不同端口的启动方式来模拟多台broker，在实际分布式环境中通常是所有kafka进程使用相同的端口

```bash
# 拷贝原config/server.properties，作为不同kafka进程启动的配置文件
> cp config/server.properties config/server-1.properties
> cp config/server.properties config/server-2.properties

# 修改两个配置文件中响应的几个配置选项，其中包括log目录（用来存储数据的目录，服务进程端口，以及broker的id）
config/server-1.properties:
    broker.id=1
    listeners=PLAINTEXT://:9093
    log.dirs=/tmp/kafka-logs-1
 
config/server-2.properties:
    broker.id=2
    listeners=PLAINTEXT://:9094
    log.dirs=/tmp/kafka-logs-2

# 然后用每个配置文件分别启动不同的kafka进程
> bin/kafka-server-start.sh config/server-1.properties &
...
> bin/kafka-server-start.sh config/server-2.properties &
...
```

##### 4.3 优雅的停止Broker<a name="优雅的停止Broker"></a>

kafka集群在有broker宕机的时候是能够自动的发现并对在这台宕掉机器上的leader partition进行新的leader选举的。通常broker停止的原因要么是crash了，要么是运维上因为需要进行配置修改或者维护目的进行例行重启。在后者这种情况下，除了直接kill掉broker进程外，是有更加优雅的方式来停止kafka进程的。这样做有两个好处：

1. kafka会把所有还没有sync到磁盘中的数据进行commit，以保证数据不会丢失。
2. kafka会先对当前机器上hold的partition leader转移到其他机器上去，然后再shutdown。这样能够让新的leader选举主动被发起，从而加速新的leader上线的时间，也能够降低对消费端的影响。通常能够将对消费方的影响控制在毫秒级别。

如果采用这种更优雅的方式来停止kafka进程，那么对数据的sync和新的leader选举都将会自动完成，要这样做需要在kafka server config中加入以下配置选项：

```properties
controlled.shutdown.enable=true
```

但这里需要注意的是，这种controlled shutdown只对replica大于1的partition有效。

另外还有两个跟这个相关的配置选项是：

```properties
# controlled shutdown会因为各种原因而失败，这个选项可以控制controlled shutdown的重试次数
controlled.shutdown.max.retries=3

# 每次重试之间的停顿时间，默认5秒
controlled.shutdown.retry.backoff.ms=5000
```

配置了以上选项以后，就可以用如下命令进行broker的停止了：

```shell
> bin/kafka-server-stop.sh
```

##### 4.4 Balancing leadership（kafka-preferred-replica-election-replica-election）<a name="Balancing leadership"></a>

当一个broker宕机，或重启以后，原先以这台broker为leader的partitions将会被转移到其他的broker上去。当这台broker重启以后，就没有任何一个partition的leader在这台机器上，也就不会服务于任何从client（producer或comsumer）来的读写操作，这样会造成这台机器过闲导致的负载不均衡问题。

为了避免这种不平衡，Kafka中有一个概念叫做`preferred replicas`，比如当一个partition的replicas列表是1，5，9的时候，node 1就是这个partition的preferred，因为它出现在replicas列表的第一个。当出现某个partition的`leader replica`跟replicas列表中的第一个broker id不一致的时候，说明现在这个partition的leader不是`preferred replica`，如以下情况：

```
Topic:test	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 1	Replicas: 0,1,2	Isr: 2,1,0
	Topic: test	Partition: 1	Leader: 1	Replicas: 0,1,2	Isr: 2,1,0
	Topic: test	Partition: 2	Leader: 1	Replicas: 0,1,2	Isr: 2,1,0
```

可以用一下命令来触发kafka集群对leadership的重新分配。

```shell
> bin/kafka-preferred-replica-election.sh --zookeeper zk_host:port/chroot
```

但是由于这是一个集群层面的操作，通常会运行的很慢，所以也可以通过在服务配置中加入以下配置来对这个过程进行自动执行：

```properties
auto.leader.rebalance.enable=true
```

另外，这个工具还可以通过增加`--path-to-json-file`参数来对控制对哪些preferred replica进行leader选举。这个选项后面需要跟一个json文件，该json文件中定义了哪些topics的哪些partition信息。如下示例：

```json
{
  "partitions": [
    {"topic": "foo","partition": 1},
    {"topic": "foobar","partition": 2}
  ]
}
```

或者

```json
{
 "partitions":
  [
    {"topic": "topic1", "partition": 0},
    {"topic": "topic1", "partition": 1},
    {"topic": "topic1", "partition": 2},
    {"topic": "topic2", "partition": 0},
    {"topic": "topic2", "partition": 1}
  ]
}
```

假设该json文件名为`preferred_replica_example_test.json`，那么运行命令如下；

```shell
bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181 \
                                        --path-to-json-file preferred_replica_example_test.json

Created preferred replica election path with test-0,test-1,test-2
Successfully started preferred replica election for partitions Set(test-0, test-1, test-2)
```

这里需要注意的是：当这个命令运行以后，并不是立刻就会让`preferred replica election`完成，而只是触发了这个进程开始，其后台工作步骤如下：

1. 这个命令更新zookeeper中的`/admin/preferred_replica_election`节点，将需要做leader调整到`preferred replica`的partition写入到这个节点中。
2. Controller会监听这个zk path，当有数据更新到这个zk path的时候就会触发选举，controller读取这个节点中的partition list
3. 对每一个partition，controller会读取它的preferred replica（assigned replica列表中的第一个），如果其`prefeered replcia`不是leader，并且是在isr列表中，那么controller就会发起一个request到hold preferred replica的broker，让其变成是leader。

* 如果preferred replica不在ISR列表中怎么办？

这种情况下controller的move leadership任务会失败。通常需要查看是否该replica在preferred broker上是否有数据丢失。当恢复到ISR列表后可以重新运行

* 如何确认操作的运行结果

可以使用list topic来查看topic和partitions的状态（leader，assigned replicas，in-sync等）如果每个partition的leader都跟其assigned replica中的第一个broker id一致，则表示成功。否则失败。

* 注意：`kafka-preferred-replica-election.sh`只是尝试去调整每个partition的preferred replica，并没有对replica的broker顺序或者broker列表（`Replicas: 0,1,2`）进行修改，如果想要对replica list进行修改，需要用`kafka-reassign-partitions.sh`工具

另外，kafka中还可以通过设置broker的rack来让kafka自动的对replica尽量的均匀分布在不同的rack，同时也可以避免kafka集群对整个rack的故障能够免疫。要设置broker的rack信息，只需要在server配置里加入如下配置：

```properties
broker.rack=my-rack-id
```

需要注意的是，kafka会尽量保证所有的partition的leader replica均匀的分布在所有的broker上，而不会考虑在各个rack上的分布，以此来确保负载均衡。而且如果不同的rack上有数量不同的broker，那么broker少的rack会被分配更多的数据，所以一个非常重要的点是：**rack规划的时候，需要尽量确保每个rack内包含的broker数量尽量相等 **

