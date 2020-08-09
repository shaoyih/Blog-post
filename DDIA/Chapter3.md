# Storage and Retrieval

根据题目来讲本章重点就放在了，read 和 write，这也决定了每种DB的走向，以及其在不同情况下的performance，本章主要讲了关于DB 的 data structure，也就是不同db下究竟使用了什么样的index 来 加速我们 query。所以说我们每天写些application code但是不知道底层就会毫无进步，最多的就是熟悉工具而不知道怎么trade off，怎么优化。所以这章算是一个关于我们开始认识不同DB的一个重点。



## Data Structure that power your database

这一小节主要是通过一个 bash的简单写法来告诉你没有index的db需要 O（n)的时间可能才能找到自己想要的data，但这效率太差所以需要引进index 作为我们数据库的一种数据结构。用书中的一句话

> Many database allow you to  add and remove indexes, and this doesn't affect the contents of the database; it only affect performance of your query.

这里作者提到的这点就是index就是index和数据本身无关。然后作者又标志性的提到了一句话叫做

> Well chosen indexes speed up read quires but every index slows down write.

这里再次讲到index，所以index 的好坏很多时候也是要based on不同的cases的，合适就好。 之后就开始介绍两种相对重要的 indexes 和其对应的具体实现。



## Hash Index

这个听名就可以猜到一种通过Hash的方式来进行index。 具体的实现方式就是根据key 和 offset像下图一样的结构保存到一个hashmap，当我们加入新的数据时候可能也会insert 对应的 key - offset pair, 当然这是在hard drive的具体实现方式。bitcask 就是用了对应策略所以带来了一些好处，具体研究请看 [bitcask](https://zhuanlan.zhihu.com/p/53682577).

![](https://drive.google.com/uc?export=view&id=1CiSj06Dkq4CNerxkFl7p77ZUGzeY7Hb1)

这种方式看似简单实际上可以有很好的读写功能，主要是因为hashmap 一直都在memory里。

这里可能会引出一个思考update的时候怎么保证高效，因为是只加不改所以我们只需要update hashmap 就好。

接着又是下一个问题，file爆掉怎么办一直加不是个好办法啊，所以我们就需要做compact， 把相同的 key val 按照最新出现的形式保存下来 。

但很多时候我们的log可能还是很巨大，所以我们可能有多个不同的segament file， 我们在上面的基础上还需要加一层 merge， 如下图。

![](https://drive.google.com/uc?export=view&id=1qV0F6eGfNL9mWcI4ClQrJzRnDLAA9I6X)

merge的时候不用担心读的问题， 因为写我们是放到一个新的file，所以merge的同时我们其它thread 读的是旧的文件。

然后当我们下次再读的时候我们会按照 最新的mrege文件向旧的读，因为我们有了merge 这个操作的存在， 所以我们不需要查很多map就可以找到对应val。



当然这种做法不是无敌的，他也会产生很多需要注意的点：

- ___File format___ ： 选择一个合适的文件格式会影响速度， 如 csv 比 binary file 要慢。
- ___Deleting record___ : 因为我们记录可能存在在多个文件，所以我们要为删除加一些特殊记录，这样merge时检查到才会忽略旧的存在的数据。
- __Crash Recovery__ : 因为hashmap 一直在memory里，虽然没了也没事反正disk里文件还在还可以造，但还是有backup 比较好 比如存个snapshot。
- __Partially written record__ : 中途crash 的file， 可以通过checksum来对比，如果是partial 就可以ignore了。和网路传输的那个checksum有点像。
- __Concurrency :__ 因为这种做法需要保证 写入顺序，不然会引起歧义，所以解决方法是可以多个thread 读， 但是只有一个 thread 写

***compact  + merge 的 process 就这样诞生了。这个概念很重要的原因是我们之后通过这种index 作为基础进行优化产生了db界举足轻重的模型***



### append only  好处

1. 写的速度快，特别是转盘式硬盘
2. reovery简单 因为不会overwrite 旧的data
3. 避免文件碎片化



### 坏处

1. key 很多就完了 -> 性能降低，hash collision，很多随机I/O
2. range query gg 你懂的 ：）， 但是还是有办法的，看下面。



## SSTable & LSM - Trees

之前无序的log 为我们造成了头疼的坏处，那我们为什么不把他们排序一下，这样就产生了SSTable - Sorted String table. 对于每个segment来讲也都是有序的。



### SSTable 的好处

1. merge 不需要跳来跳去，按顺序merge 每个 segment （怎们看怎么像k- way merge），只保留最新的 key和上面一样。

2. 我们的hashmap不用再存那么多key了因为是segment排好序的，所以就可以按block block 来存了，如下图。

   ![](https://drive.google.com/uc?export=view&id=1P4XXjUiC2oC7p1nQp4Xcb_CMK6-SgmX9)

   ​						这样上述 hash index 存在的两个问题就不存在了

3. 反正对于读来讲，我们也要扫key - val block range， 所以我们写的时候就可以把上图类似的block 压缩了再存。



### SSTable 的实现 和 维护

因为是memory 所以 datastructure 就很灵活我们可以用 treemap 的 red black tree 也可以用 AVL tree 等等。

__操作顺序 ： __

1. 把新数据先存到memtable
2. table太大的时候，存到file。
3. read 的时候 先读 mem table -> 最新的 file -> ... -> 向旧的读
4. 然后后面的segment merge and compact

是的这么做有关于memory 的问题又出现了，突然没了怎么办所以我们还是要keep 一个 log的 没加一次 table 就 加一次 那个 file， 需要写出的时候这个file也就没用了，取而代之搞一个新的, 主要就是搞个snashot 预防一下。

用到这个 structure 的 DB 有 ： Level DB, Rocks DB, Cassandra，HBase。 Idea from google big table.

__LSM Tree__ 上述所讲到的整个index机制被称为 Log strutured mrege tree。

不仅仅是上述的DB用到的 LSM tree 这种，甚至很多 full text search 的indexing 工具也用到了这个机制。



### 有关性能优化

1. 如果我们找一个不存在的key 我们需要从 mem table 找到 最久的文件才能 确定他是否 存在，效率低下。
   - bloom filter 可以解决， 把一个 string 通过平均分布的hash function 来 分布到不同的bit，这样的做法可能会对判断key 存在的case有误差，但是对于有一部分却可以很肯定通过它的hash来确定不存在， 因此节省一部分力气。
2. 然后其次对于compact and merge 现在比较流行的有两种策略
   - size-tired -> merge small and new to old and big.
   - leveled-compaction -> moved level by level once full, each level has different size.
   - 因为cassadra 都实现了 所以具体可参考 Cassandra docs



## B tree

不想写。。。。。。。真的太熟了，贴张图吧，表示对我教授的尊敬，她教的真的太好！

![](https://drive.google.com/uc?export=view&id=1XDECJLkofGi4fHrqRGylPu_a7I00IeI2)

这是 b tree 主要结构，通过 sort ， range ， ref 穿插 来方便 query

![](https://drive.google.com/uc?export=view&id=1v9MJePx8YmVNXUKg27aB3UNq1VYO94AE)

这是当一个page满了如何分 page

这个提个术语 branching factor，就是每个 page 有多少个 ref 到 子 page。

b tree 的 performance 不用说了， 大部分relational db 用了这个 结构沿用至今是有原因的

> A four level tree of 4kb pages with a branching factor of 500 cna store up to 256 TB

4层就有此威力可想而知，这个四层的query 速度还可



### B tree 的 稳定性

首先的问题就是 b-tree 不是LSM- tree append only 的 方式， 所以考虑到update的时候。我们就需要两步，找到对应位置，update。

问题 1： 比如page 满了 分页过程 crash -> gg -> 然而此时也要引进类似 snapshot的东西在这里叫 **write-ahead log**，巧合的是他是**append only**。

问题 2： concurrency 问题 -> latch lock 解决



### B tree 优化

因为发展历史之久 方法太多了，简直优化到极致，书中说了比较常见的一些。

- copy on write -> 当需要分page genrate 一个新的 page 而不是直接原有的改
- 压缩key size -> 更大的branching factor -> 更少层 -> query faster
- b+tree
- fractal tree





## B tree VS. LSM Tree (核心)

这两个indexing 家族的对比直接奠定了之后数据库类型的对比因为各自的走向决定了DB的强项，所以这里的对比对以后系统数据库的选择有很大的参考价值。

B tree 其最大的优势就是读，但写起来就慢的一匹。

LSM Tree 最大的优势 写，但读起来就慢一些相对来讲。

在证明为什么他们的读写速度前，作者说到了一个东西叫做

**write amplification** 就是在彻底写入数据库前到底写入硬盘几次

**read amplication** 在query到数据前进行了几次read

其实一开始比较质疑lsm tree 为什么 比 b tree少 算了一下就明白了。

试想 Btree 写一次要重写一次要从写page 两次 假设page。 **O(p)**

因为size tired 的 lsm 比较难算而且取决于merge算法，那我们就拿 Level tired 比较。写入到最底层。首先可以假设每层的增长率为 **L**, 数据库的size是 N, 最小segment 的 size 是 s 所以我们最底层的 sement的 size 就是 N/s.一共有 log_L (N / s)层假设我们每次merge到下一层需要 L(只是预估) 因为会被反复写入一层的文件又因为k 是 factor， 所以约等于 我们 最后的写入 **O( log_L (N / s) * L)** 

具体参考下 [rock db level based](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction)

很明显 lsm tree 写赢了

至于read 就不用比了很显而易见, b tree 就像 正常的 bst mutiple st 一样确定range往下跳 easy and fast， 而 lsm tree 每层 bfs一下看看谁快 ：）

b tree缺点：

1. ssd被反复写寿命有限
2. segament part 太多 sst 反复re write 所以不存在这个问题
3. 小change rewrite whole page

lsm tree缺点：

1. compact 可能占用磁盘资源
2. high write throughput
3. 如果操作不好merge 算法，留下太多segment一样会爆
4. 没法很好的最大哦transactional semantic 有很多mutilple val 存在可能



这里延展开能讲很多很多之后慢慢搞懂。



## other indexing

- clustered
- Non clusterd
- secondary index
- mutiple column index
- full text search index
- Mem cached- > 这个是后话之后再说



## OLAP VS. OLTP

**OLTP :** online transactional processing 我们通常的业务都是这种transaction类型

**OLAP :** online anayltic processing 随着大数据时代的崛起，数据就是钱。

分析数据对数据库的需求和通常业务对数据库的需求有巨大的差别。

接下来用书中的一个table来表示下吧

| 性质     | OLTP               | OLAP                          |
| -------- | ------------------ | ----------------------------- |
| 读       | 少量记录，少量row  | 大量记录，大量row，少量column |
| 写       | 随机access，低延迟 | 大量提取，重构，写入，俗称ETL |
| 用户     | 端用户             | 数据科学家，BI那些人          |
| 数据量   | gb，tb             | tb， pb                       |
| 数据表示 | 最新数据           | 历史记录                      |
|          |                    |                               |



### 数据舱

数据舱这个词对于在大公司工作的人肯定不陌生，收集公司里不同的 transaction 系统的数据保存下来，供分析的人可以分析。因为通常情况下分析不会直接从业务数据库提取，即使是数据库管理员也不会允许因为会影响数据库的state，和efficiency。

所以通常都会从业务数据库**extract** 后**transform** 然后 再**load**到数据舱。（ETL)

书中这图真的很好的解释了整个process

![](https://drive.google.com/uc?export=view&id=1kxr1fLlDUoUfjVt2vZuHEDmcGOTgBmJ5)

大多数数据舱使用的还是relational，因为可以预见到他们的读是巨大的，而且需要match analysis的需求。



### stars & snoflake

- 数据分析通常要很好的建立数据库schema，方便分析。

- 通常比较常见的数据舱模型就是 star schema， 把每个单一的event 分成一个table，即使是类似日期可能也会是一个table因为它可能有其他属性类似 情人节，劳动节等。这么做主要是为了方便后面分析。

- 有个变种叫做snowflake 可能会划分的更细，类似normalization也是一层层更细
- fact table 通常情况下会有很多attribute可能是上百个，所以按照我们直接row row 的存储方式可能会引起很大的问题。因为我们读取的时候可能只需要个别column的所有row。



## Column oriented storage

之前按row存的方式每个row 里面的 所有 value 可能都会挨着存。按照上面的需求要的这种做法就爆炸了会慢到爆炸。

既然我们的需求是column为何不把column挨着存，于是column oriented storage就诞生了。

现在的做法是

- 把每个column的val存进一个file，有多少个column就会有多少个file。
- 每个file里的数据都是按顺序放进去的，如第一个文件里的第三个value 和 第五个文件里的第三个 value属于一个row。



然而我们可以预见正常的relational db 数据量大了，每个column会有重复的值。所以说我们就可以用一个技巧俗称bitmap进行第一次压缩。我们只提取unique value 然后 把他对应出现的row 标成1没出现的标成0. 如下图

Image1:

![](https://drive.google.com/uc?export=view&id=1fZkmdgdDidHirxHlgqob6A8BVjMEOklW)



但是如果数据量巨大的时候还用相同的办法可能有点蠢，把这个看成一个matrix的话就会感受到matrix非常稀疏。和我头顶的头发一样。

我们可以再次压缩对他进行encode如下图一样，下图的encode是对应上图的做法。换句话说我们把头发集中起来从某个角度看我的头发就不稀疏了。

Image2:

![](https://drive.google.com/uc?export=view&id=1uTMMqvXkCz9A974sp-0csEV5R_da6akT)



有很多扩展如 cassandra， hbase的 column family 概念也是沿用了这种做法。



- warehouse局限
  1. 把data 从 disk load 到memory， memory 到 cpu cache。
  2. column oriented 结局了这个问题不用load 所有 columns
  3. 还带来了个好处：很好的使用 cpu cycle， 具体就是一个loop 肯定比各种function call 加起来快，然后再 process一下。



## column sort

插入的时候集体sort 进去，有点猥琐。。

好处：

1. query 条件方便，比如 比某个价格贵。
2. 更好的压缩空间，相似的都排在一起了。

本来也需要备份附件，所以存这些备份的时候可以是不一样sort order， 所以提取的时候就找最适合自己query的版本。



说了这么多的好，但都是在读的方面写起来就难受了。还好前面讲到了lsm tree了解一下。





### Materialized view

试想一下我们一直在 read query，但我们有很多query 用了相同的 aggregate function。每次叫每次叫就很浪费。

然后我们就可以把结果cache一下了。。。

和 virtual view 大不相同， virtual view 本身不含数据 只是把复杂的逻辑在view 层建好方便query。 真正到query的时候就是expand query 的一个过程。

而 materized view 就是切实的存下来。update的时候也会随着update。这样写就更贵了。。

如下图作为一个参考的例子，我们原本有三个维度 net price，product，date， 比如我们做了一个sum的aggregate function对net-price存下 三维变二维以后需要类似的aggregate function就直接跳进去提取就好。

![](https://drive.google.com/uc?export=view&id=1BVIMkqmpGykL3nueT8Zy5m2oWyQAwvgC)



好处： 1. 读的更快乐

坏处： 2. 不够灵活，可能会有占空间的问题。



## extension

[level compaction in Cassandra](https://www.datastax.com/blog/2011/10/leveled-compaction-apache-cassandra)

[When to use leveled compaction](https://www.datastax.com/blog/2011/10/when-use-leveled-compaction)



## 反思

这章很重要可以说是对整体的data model类型的一个展望。了解了行业背后对数据库不同的需求，以及其不同的发展趋势吧。对之后的 system design 很是重要。回去dive deep 的看下big table 还有 column family 的那些数据库们。

Now this is not the end. It is not even the beginning of the end. But it is, perhaps, the end of the beginning. --Winston Churchill









