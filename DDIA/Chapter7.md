# Transactions

这章的话主要就是相对基础的讲relational database中如果我们按照transaction 的思想可能性会遇到一些问题和bottleneck。以及讲到了我们会用什么样的策略去解决这些问题，同时讲到了这些解决方法的Trade off。 相对来讲这章还是比较基础的当作复习一下另外可以复习一下锁的类型以及总结一下现存的DB里大都采用什么样的一个策略， 因为行业里也没有规定某个isolation level在什么哪个level做到什么。名字都比较乱但我们还是按照统称来。



## ACID

这个概念不用说了吧，只要进入行业就必须了解的一个东西我就按照自己话简单总结下。

**Atomicity**： 原子性 all or nothing。

**consistency**：一致性 不用说了各种算法手段都是为了这个目的。

**isolation**： 每个transaction都是单一独立不互相影响。

**Durability**: 一旦写入db成功，那么操作对于db来说是永久的。

对于上面的 durability来讲作者单拉出了一页 p227写到了和replication一起的话可能会产生的一些问题。



## single and mutiple object operation

顾名思义就是在DB进行transaction的时候是对一个还是多个object进行了操作。object在这里的定义是（rows， documents，recrods）

有太多case需要用到mutiple object operation了所以对于 一些concurrency的trasaction来说可能会有很多问题。



下面用一个table来总结一下这些可能遇到的问题和对应解决的isolation level吧再细分来讲。





## Concurrency problems

| Concurrency problem     | isolation level                              |
| ----------------------- | -------------------------------------------- |
| Dirty Read, Dirty write | read commited (row level lock only on write) |
| read skew               | snapshot isolation                           |
| Lost update             | repeatable read                              |
| write skew              | Serializable, snapshot                       |
| phantom read            | serializable                                 |



**问题类型1:**

**Dirty write:**同理一个 transaction并没有 commit的时候 另一个transaction overwrite 第一个 transaction的write。

**Dirty read:** 一个transaction并没有commit 的时候 另外一个 transaction 可以读到第一个transaction 的 write。

**解决：**

dirty write： 好说 必须要有个 row level 的 lock

dirty read： 不能用read lock了， 通常保留两个value，在写没commited以前只能读old。



**问题类型2:**

**read skew：**一个transaction进行两次读，在两次读的区间另外一个transaction写了这两个object并且commit了。

**解决**

snapshot isolation，就是保证读的那个transaction 的 version是一致的。这个 version control的技术也被称为 **MCVCC**。

- same snap for whole transaction
- visibility rule
  - list 所有正在运行的transaction，任何write 写入都会被ingnored
  - 被aborted 同理
  - write 有更晚的transaction id 同理
  - 其他都可见

- index怎么解这么多version的话 -> point all + filter



**问题类型3：**

**lost update：** read-modify-write 两个这样的cycle同时发生的时候。其中有个 transaction 并没看到另外一个transaction 的 updat。 很多情况会发生类似的问题



**解决：**

1. Atomic write -> remove read-modify-write这个cycle在application code里。

   比如写这样一个query：Update counters set value = value + 1 where key = ‘foo’； 这时候就需要一个互斥锁，就是使用锁的时候其他读无法操作直至写已经结束

2. 外部锁：自己加
3. 自动检测： 通过snapshot
4. compare and set： 对比设定



**问题类型4:**

write skew：同时读，based on result 然后改变了不同的 object。

phantom：一个transaction先读了一个object 然后又读了一个，在此期间另外一个transaction write了 第一个transaction要读的后面的object，导致幻影出现。

特质

1. query到一个或者多个符合条件的结果
2. based on 渠道的结果决定决策
3. 在数据库，写入更改或删除



**解决方法：**

1. 被关锁 make it serialize
2. 乐观锁，错就错吧最后对了就行



## 类型差别

**read skew** vs. **phantom**

Read skew： 两次读理论上不符合要求，但user可能不知道

![image-20200815164222975](/Users/shaoyi/Library/Application%20Support/typora-user-images/image-20200815164222975.png)

phantom read：一个transaction的read 们是有相互联系的，前后不符所以就像看到幽灵一样，user知道。

![image-20200815164311478](/Users/shaoyi/Library/Application%20Support/typora-user-images/image-20200815164311478.png)



**snapshot isolation** vs. **read commited**

Snapshot: 有多个version

read commited： 就俩 version 新旧



## serliazability

### Actual serial excution

- stored procedure
  - 写法不一
  - code 难管理
  - 写不好效率不行

### 2PL 2 phase lokcing（悲观锁）

shared lock：读锁

exclusive lock：写锁

规则可以自己回忆下。

**predicate lock** ： lock related 条件的 columns

**index range lock**： 靠index 进行lock 和 predicate 差不多

### SSI serializable snapshot isolation (乐观锁)

- muti version check， abort based on result affect each other or not







## Reference 

concurrency problems with image: https://blog.acolyer.org/2016/02/24/a-critique-of-ansi-sql-isolation-levels/











