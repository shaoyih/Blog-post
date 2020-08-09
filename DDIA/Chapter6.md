# Partitioning

上一章刚刚讲过复制，这章主讲分区。相比去复制来讲，分区是吧整个dataset 切分成多个不同的 partition, 同样也被称为 sharding， **一般来讲一份数据只会属于一个分区， 他的主要目的就是为了 Scale。**

在各个DB 里类似的概念也有 不同的称呼

- shard : Mongo DB,  Elastic search, solr Cloud
- region: HBase
- tablet: Bigtable
- vnode: Cassandra Riak
- VBucket: CouchBase

这章主要就是讲不同的sharding 策略 以及他们是如何rebalance这些partition的。

相比于 replication 而言， partition 有些 自己的属性

- 一个node 可能存有多个 partition
- 即使一个数据属于 一个partition，但也可以用 replication 的策略让多个相同的partition 存在于不同的node。



## Partition 策略

partition 有 partition的问题 通常情况下

理想： 当分了多个parition 在不同的node的时候， query对这10个的请求量应该是平均的。

现实：很多种原因造成这些请求可能会不是平均分配的 书中也称为 Skwed，load 特别大的node也被称为 热点（hot spot）

最直观的方法就是 随机分配数据到 partition但是会有一个 很大的 con， 读的时候需要全读。听起来这样分区的意义就会变得很小。

所以之后会有两种方式：

1. by key range
2. by key hash



### key Range

这种方式顾名思义，就是按照 key 的 范围来排列，比如一个词典我们有 5个 partition，可能会平均分配一下 A - E （1） F-J (2)...  但是这个boundary 可能并一定要按 key 本身的数量来排，举个极端的例子比如 60%的单词都聚集在A， 如果按上述方式分的话可能会把第一个node 变成一个热点。我们可以按数据量来决定 他们的 boundary，比如像刚刚的情况， 我们 可以把一个 A 切分到 3个node去，大概道理就是这样。



好处： 因为是range排的，非常的好range scan

坏处： key range 还是有很大几率（选不好key）造成 hot spot， 比如我们有一个数据系统是按月分的，那这个月的partition可能会爆掉，但我们在前面加个其它有标识性的就可以分配到其它partition。



### Hash of key

Hash 前面提到过很多次了， 就是无非通过key 生成的 Hash 来决定分区。

因为本身的目标只是为了分区所以hash function不需要很强的加密行。比如Mongo DB就是简简单单的用 MD5. 但还是有些hash function不太可用，比如java 或 ruby里不同的process 通过 object.hashcode() 可能会拿到不同的hashcode

这里提到一个很重要的观点 **Consistent Hashing**,细致的后面 rebalancing的时候再总结吧。



好处： 更好的 distribute 到 partitions

坏处： 只是相对 key range 来讲没法 range scan了



### 各个DB 的parition策略

Mongo DB: https://docs.mongodb.com/manual/core/sharding-data-partitioning/

Couch DB：https://docs.couchdb.org/en/master/partitioned-dbs/index.html#partitions-by-example

Cassandra：https://www.instaclustr.com/cassandra-data-partitioning/

Riak:  https://docs.riak.com/riak/kv/latest/learn/concepts/clusters/index.html

Voldmort: https://www.project-voldemort.com/voldemort/rebalance.html



### 解决 倾斜 workload 和 缓解 热点

即使是用了hash的方式解决hotspot的可能还是有很多 热点 partition可能会发生。比如名人的数据，热门的流量等。 但这种事情在任何系统都比较常见，所以每个系统都有自己handle的方式比如在key前 加random number， 但是同理也会增加写的难度，比如需要bookkeeping一下。所以相对合理的方式是用少量的随机key。也许将来可以很好的automate但是目前来看还是需要一些 trade off



### Secondary index 的 partition

首先提个前提背景， secondary index 一般在relational database 比较流行，但是后面因为这种方式对data modeling 和 query有很大的帮助于是很多 document 也用起来了。

但是partition的时候，和key-val pair 一样也需要partition。两种方式

- partition by document
- partition by term



#### Paritition by document

就是每个 secondary index 只负责自己 parition的 index。好处当然就是只用在当前partition找就好了，所以也被称为 local index。

但是查找的时候就需要特殊标记化id 来表示存在在哪个partition了。但是如果我们想找寻某个特殊index就需要找所有partition 然后 结合一下。

这种partition 方式也被称为 **scatter / gather**（map reduce？？？lol），所以如果收集所有的的话可能就会有拖累现象，比如tail latency。



#### Partition by term

相比于之前的local 形式 其实我们可以把它global 化， 换句话就是 partition by term，比如我们按照字母顺序把 secondary index 存在不同的 partion。

好处比起前面的by document 当然就是读起来更方便了， 但是反关写很有可能就是牵一发而动全身。

同样的这种写也是异步的所以，一旦发生改变很有可能index 并没有来得及被update掉。





## Rebalancing

作者给出的定义如下。

> The process of moving load from one node in the cluster to another is called rebalancing.



#### 机器可能发生的问题

1. 更大的throughput，加cpu
2. dataset增加 disk，ram
3. 机器fail， 用其他take over



#### 最低要求

1. 的的确确

















