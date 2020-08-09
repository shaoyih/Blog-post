# Replication

作者在这章开始前先简单介绍了一下Distributed data 的一些特性 比如为了 提高

1. Scale
2. Fault tolerance / 高可用
3. Latency

在开始进入正式话题前， 作者先提了两种 vertical scaling 的方式：

1. shared memory architecture， 很有限的容错率。
2. shared disk architecture，通过一个快速的 网络来连接 disk， 有些data warehouse 就是这种做法。

因为vertical scaling 的cost 真的是直线上升而且还存在很多其它问题，所以慢慢的horizontal scaling (shared nothing architecture) 就成为了一种主流也就是我们这个 topic 很在乎也很 重要的东西。

> if your data is distributed across multiple nodes, you need to be aware of the constraints and trade-offs that occur in such a distributed system - the database cannot magically hide these from you.

正是因为横向扩展的出现， 很多组件部位的选择也就多了, 加之很多操作的空间。所以trade-off也就变的异常重要。然后就出现了两种选择 复制（replication） 和 分割 （partition）。接着也就蹦到了这章的replication。



## 简介

Replication 存在有这样的几个原因：

1. 如果用户遍布全球，数据只存在一个地方就会很慢对于地球另一端的用户。(for latency)
2. 当有一个机器或者部分机器宕机的时候，其它也还是可以用的。（for availability)
3. 通过load balance 所有read request，读取带宽会变大。（for scalability）

但是这么想复制 就不存在什么问题吗？ 并不是这样的问题可能并不会小，比如当数据发生改变的时候。

所以文中介绍了三种主流的算法：

1. single-leader
2. multiple-leader
3. leaderless-replication

在每种不同机制之下我们也可以进行不同的trade off: 同步或者异步， 如何处理失败的case发生时。作者在书中强调了 一下 replication的目的和很多developer理解的 eventual consistency并不是指一个东西。



## Leaders and followers

这种 replication 做法也被称之为 master - slave 做法，简单讲一下就是从 mater 写入， 但是从master 或者slave 读 都可以。 这种replication的策略不仅可以在database上进行复制，同样的策略也可以应用于 message broker 或者 network filesystems。



### Syncronous VS. Asynchronous Replication

这里讲的同步或者异步是指在我们relicate 过程中使用同步或者异步的策略，下图很好解释了同步异步。

![](https://drive.google.com/uc?export=view&id=10KBXpenZXfo-FBC-cW4TNMLEr917Y-6y)

Follower 2 是同步 3是异步。

Synchronous 最明显的

- 问题：就是需要等待follower都接受到了才会process所以可能会等很久。
- 好处：gurantee data up to date

所以说一般所说的synchronous就只有一个follower是同步的也被称为 semi-synchronous。

所以-> 一般情况下都会使用异步

- 问题：durability 会下降，因为有node会fail
- 好处：无需等待一只写就好

所以通常情况下会使用async 因为可以用其他办法补救后面会讲。



### setting up new followers

当我们设置新的 follower时候首先要考虑的问题就是 新follower 如何 能跟上 leader的进度，因为数据是流动性的，leader也会不断被插。 加lock破坏了我们 leader follower的意义。所以正确的做法应该是。

1. snapshot leader
2. copy snapshot to follower
3. 找到 leader 里 replication log 里 之前拷走的位置
4. 然后 follower 根据 backlog里 新加的行 去 catch up



### handling node outrage

当有 任意node不可用的时候，问题就很大了，不过问题也分是leader 还是 follower。

follower fail ：根据log catch up。

leader fails ： 会经历倒台三步曲，通常的倒台做法就是选个新的leader，这个事情可以手动做，也可以自动化做。

**自动的步骤大概是 ：**

1. 确定leader gg了，因为导致 node gg的原因太多了所以通常用一个timeout来确定是否dead。
2. 通过election 来选举下一个leader， 通常的指标就是哪个node更up to date的data。
3. 然后重新config 新的leader 让其他node 去 consume它。

**但是如果此时旧的leader 死灰复燃 会产生很多问题 ：**

1. 旧的leader 有些 数据没被 replicate， sol： 忽视old的那些没被replcate的 record
2. 但是上述solution 很危险，作者文中给了一个导致db 和 redis不一致，引发了一些隐私问题。
3. 旧的回来后，有可能会有两个leader。sol：此时只能shutdown一个但处理不好两个可能都会关了。
4. 如果没有选择合适的timeout让旧的倒台的话，可能会让high load系统变得更糟。



### Implemetation of replication logs

因为上面我们讲到了可以用log 来做recover，然后作者在这一小节就给了一个 写 log的方式。



#### statement based（声明式）

和名字一样是靠 声明作为log 然后让 follower 根据log来跑这些声明。但是有几个问题。

1. 有一些 nodeterministic function 例如 NOW( )，可能会产生问题。

2. 声明必须按顺序来，没法concurrency

3. 有很多side effect 而且在不同的replica 会不一样因为（trigger， stored procedure， udf等）

   

#### write-ahead log （WAL）shipping

也和名字一样，是写前加入的log，append only，并且是squence of byte， 所以说follower 和 leader 必须要用一样的data structure。这么做最大的问题就是太replication 太 closely couple store engine了。所以有version 不同问题的时候就需要考虑replication协议允不允许不允许的化，就需要等待upgrade。



#### Logical(row based) log replication

前面那种log的方式粒度太细导致会有version问题，但是logical log的力度是每个row。

- 首先对于每个row一定要包含所有的column 新 value
- 其次要有足够的信息表明那些delete row，比如用primary key
- 最后要有足够的信息表明那些update row，比如用primary key， 还有所有column的新value



#### Trigger based replication

如果是部分replicate的话或者复制一种数据库的到另外一种，这时候可能要上升到 application code的层面去处理replication 了，比如trigger 就可以 注册一些udf 去做对于replication。



### Problems with replication lag

replication lag 存在很多很严重的问题，书中提出了一些比较有意思的讨论。作者首先提出了 synchronous 是多么的不实际。

首先如果假设我们是一种很理想的模式 **read - scaling architecture**, 我们有很大的read load，如果我们synchronous， 所有的 follower 可能也要等到 leader反应，所以这种case并不是很切合实际。

虽然 Asynchronous 会导致 eventual-consitency 的问题但是avaliability看起来更符合实际场景。用户们不可能为了等一个写入，甚至是其他用户的写入 等很久吧。

因此异步产生的 replication lag 需要想办法解决



#### *Reading your own write*

首先我们先上图，可以看到如果用户刚刚写入了一条数据，然后从有延迟的follower读的时候就会发现数据不对。

![](https://drive.google.com/uc?export=view&id=1MYNb5Zf4RQ-_kDmMvSyKwyuslPsbC31g)



第一个出现在书中的策略就是读自己写的， 虽然对于其他用户来讲可能会有延迟，但是可以首先确保自己看到这个数据的准确性。比如我们提交了某个刚刚更改的东西但是刷新页面发现没有更改就会产生很大的问题。但我们能如何执行这个策略呢作者给出了几种方法

- 当读用户可改的信息要从leader读，读别人可改自己不可改的时候就可以从follower。但是很多时候我们没法确定可不可以改的token， 所以我们就选那些100%确定的 比如自己的 profile。
- 但如果一个程序里大部分东西都可以被更改的时候我们上面的策略就没用了，我们就可以考虑些其他的标准了比如track一下上次更改的时间是多少然后根据时间来选择从哪里读。
- 实际的做法就是用户记住最后一次写的timestamp， 然后再次发送读的请求的时候确认某一replica已经到了规定时间才读，不然就换。
- 但是如果这些replicas 不在一个数据中心就需要多走一层proxy可能。



此时又蹦出了一个问题如果是一个用户 多个设备怎么办，我们就需要 perform cross-device的read after write consistency了。

- 上述说的timestamp的元数据可能就要 centralize了。
- 上述提到的最后一个问题在这里同样变得复杂了如手机和电脑去了不同的 datacenter，所以这里就要控制他们去一个了。



### Monotonic Reads

如果一个用户插入了一条数据如下图， 同时另外一个用户同时发起了两次读的请求我们可以看到 数据回流的情况，就是两次数据并不统一的情况。

![](https://drive.google.com/uc?export=view&id=13sML32nFNU9z6n_LhoKLE96AyAueuZcZ)



这种时候我们可能就要用单体读法了，就是之前读了哪个现在还从哪个读。其操作方法就是

- 让一个用户总是从一个replica做读请求

- 比如用用户id的hash来做等等。。

  

### Consistent Prefix Reads

这里我们要再次上图了。不得不感叹作者的图有多么关键。这里我们又加了一个用户，作者还用了情景对话的模式来体现他又多么奇怪。

<img src="https://drive.google.com/uc?export=view&amp;id=1KpuVeuvCLkZ7u57_dEfg3Pi195LLif9D" style="zoom:80%;" />

首先poons 先写入了一条数据， 然后 cake写入了一条，但是理论来讲cake写的数据是based on poons前一条“how far...” 然后回复了一句 “about ten ...”，但是因为一部分replication lag的原因所以观察者看到的顺序是反的，他先看到了回答然后才是提问。

避免这类问题就有一个很重要的方法叫consitent prefix read, 用作者的话就是

> This gurantee says that if a sequence of writes happens in a certain order,  then anyone reading those write will see them appear in the same order.

但不同的 parition 可能会按照自己的轨迹操做可能没法保证这个事情， 后面会提到一个解决的算法叫 “happens before”。



最后作者说对于这类问题的解决方案最好不要放在application code的层面。



## Multi-Leader Replication

相比于之前的single现在就会有多个 leader node了，在行业里也被称为 master-master 或者 active/active replication 模式。但是这种情况会增加很多复杂性如作者所说。但我们还是有一些case 可能会用到这种模式（不得已。。）



### Use Cases

#### Mutilple datacenter

正如作者前面所说到的如果write 是跨 datacenter的话对于可能就要重新route到另外一个 datacenter，不过让我们来对比一下比较直观。



**Performance** :

单点-> 延迟高因为所有write 都要到一个 datacenter 然后 再replicate到其它。

多点-> 可以在本地的datacenter write 然后再异步的 replicated 到其它datacenter 用异步的方式，这样用户就不会很直观的感受到延迟。



**tolerance of datacenter outrage:**

单点-> 此时需要另外的datacenter 的 node 成为leader了， 还得调各种config 包括跨datacenter的write。

多点-> 挂就挂吧， 挂了的回来继续catch up



Tolerance of network problem：

单点 -> 因为replicate 到其他的datacenter 需要同步的在link 上传输， 所以对网络稳定有很大的影响。

多点->  因为是 Async 的情况所以没有单点那么敏感。



说了这么多感觉多点貌似没什么缺点但实际上不是的，其中一个最大的缺点就是同步的发生更改但是在不同的data-center 这样问题就可以变得很棘手。所以这种做法需要尽量避免



#### Clients with offline operation

当断网了的时候，本地设备和云端并没有sync的时候也可以被算作这种case。



#### collabrative editing

很多类似于google doc 这样的程序，允许用户同时编辑，这里提到主要是为了提async在这种上面的问题，另外他和上面的offline 其实很像。其中一个解决方法就是上锁，把上锁的单位调的很小很小。下面一节会讲如何 handle conflict。



### Handling write conflicts

当然在不同的master node同步的写入到就会产生很多冲突问题。同步问题如果发生在single master 的case通常要么打断之前的transaction，要么就是等前面写完才能再写。

作者提了一句其实可以把conflict detection这个行为做成同步的，但实际上这么做很蠢丧失了mutiple的意义还不如直接单点。



#### 避免

这种方法 最被推荐，因为可以早一点解决问题。

1. 可以是某种特殊的记录 去同一个leader
2. 可以是 user-based， user 去 固定的datacenter

但他不是无敌的还是有很多exception cases, 比如user换了地方， 某个datacenter 被炸了。。



#### 朝统一努力

单点一般最后写入才会被作为最后的value

1. 根据其uuid来决定winner也算作为一个last writer win。
2. 给replica本身一个 unique id，由高者决胜
3. mrege 两个 value
4. 用一个数据结构来保存其冲突然后交给appliaction code处理。



#### 自制 conflict resolution

上面第四个不是说交给application code 来解决所以有两种

on write -> write的时候 detect 到了 conflict 在background 解决

on read -> read 到几份不统一的数据 既可以交给用户解决也可以自行解决。

couchDB 已经被吹爆了。



#### 自动化解决conflict

书中介绍了三种自动化的 算法 可以去研究下，但是没有细讲就提了一嘴。

- conflict free replicated datatypes
- Mergeable persistent data structure
- Operational transformation



### replication topologies

看图说话，这里有三种主要的多leader拓扑。

![](https://drive.google.com/uc?export=view&id=16G1nq3_Dr32BsLAL01Wo1xWcwXJ_84lR)

- 处理重复的node 走两边的办法就是用每个 node的id，一边走一边加到write tag里。

- 前两种会导致一个node fail 其它可能也会fail， 所以越是细密的 structure 越没有单点fail的case。

- 但是all to all 会有的一个问题是 抢先 replicate到某些replica这样就会导致出错。如下图

  ![](https://drive.google.com/uc?export=view&id=1MMWko3j10fVPFRb3roH9oZaV14yELa3l)

timestamp 在这个 case 并不好用，作者在这里没细说他为什么不靠谱。

但是可以用 version vector 来解决，细节请继续往下看。





## Leaderless Replication

顾名思义就是没有绝对的node 作为一个 leader的角色在我们所有的 nodes里。 用户直接发送写的请求给其它的replicas 具体的操作一会儿再讲。 

其实这种方式很早的时候就已经存在了，但是因为 relational DB的崛起，这种方式就被搁置了一段日子。但后面出现了Dynamo DB 类似的数据库这种方式才被重新引回来，这类 lealerless replication 的DB 是被 Dynamo所启发，所以也被称为 Dynamo style.



### node outrage

首先可以回想一下，如果有写入的node不可用情况下有master case 通常会用 failover 来做write重启。但是在leaderless的话这种解决方法就不存在。

两种解决方式分别是：

- Read Repair

  当读的时候，用户会像写的时候一样，把并行的request 发给不同的node，然后根据不同的version number 来判断最新的值，之后再发送repair的请求，如果读的操作很频繁这种方式就很好。

- Anti Entropy

  有些数据库会有一些 background process 去查replica 之间数据的差异。然后把差异的值copy到miss value的弄得，但是因为leaderless写入的时候没什么特殊顺序，所以在做copy的时候可能延迟很高。



可以看下图是一种用 Read repair 的方式 来解决的。当第二个 user 读了三次发现有些 outdated replica，然后就发送了请求去repair。

![](https://drive.google.com/uc?export=view&id=1RpLv6aZ3qQEeQ6-yAypTT-S7UCDfhv8J)



### Quorums

这里用到另一个词叫Quorums, 中文是法定人数。在这里的意思是我们的 read 和 write request必须达到某个规定的数量才可以。书中给了个简单公式

> *valid Write = valid Read = (number of nodes + 1) / 2*      ->  w + r  >  n

这里我们可以看出来至少有一个share common的node，这样才能解决解决问题。下面这张图就很好的解释了Quorum 是怎么做的。

![](https://drive.google.com/uc?export=view&id=12SfmRoJXdQSMKinZd7VITa9ib4nlejvK)

对于node down掉了的情况，我们无需知道他down掉的原因只需要知道我们的 w 或者 r有没有成功发到那个数量，如果没有 的话就要返还错误。



#### Quorum限制

首先不一定非要要求到达w + r > n, 但还是会同样发 n 个出去， 这么做 可能会大大的提高可用性，降低延迟，但是结果就不一定能得到保证了。

但是 需要注意的一点 即使是 w + r > n 的情况也不能完全保证不会staled value 比如下面这些就是可能会出现的问题

- sloppy quorum
- writes occurs concurrently
- write occurs with read
- write failed cases 就没法保证w了 但又不能让其他roll back
- 可能会有timing的问题，作者之后会讲。

虽然 dynamo会做到 eventual consistence 但是Quorum并不一定会做到绝对的保证。

对于监测stalness 在 leaderless 这里目前并没有很好的办法，因为他们写入不按顺序没法检测，而且用read repair的方式并不知道value会老到什么程度。



### Sloppy quorum and Hinted handoff

对于quorum 来讲某个node down掉 或者很慢并不会产生很大的影响， 但是不要高估他fault tolerant 的能力。如果quorum不达标的话通常有两个选项：

1.  report error
2. 接受 write 写到那些 可用的 node

第二个情况就是 sloppy， 这些可用node 可能并不在自己的管控的n的范围内，但是我们会找那些范围之外的node，一旦我们范围内的node可用了，那些在范围外的node又会把之前那些请求pass回来。增加了可用，但是 这样 即使达到了 quorum 我们还是无法确认读了最新的值，因为一部分w 可能并不在 n的范围之内。

这样的做法并不局限于一个 data center， 也可以跨 data center *Cassandra 和 伏地魔* 就可以设置这些。



### 检测 concurrent write

concurrent write 不管在什么里都是一个棘手的问题。即使 quorum applied 也没有办法解决这种问题。首先我们还是看下图。

![](https://drive.google.com/uc?export=view&id=1ziKR75rlpfVZKDCI5L0QCc_ukJTvey_v)

这张图 说明了很多问题。如何解决呢，只能找解决conflict 的方法 前面作者有提到。

作者还提了一句很警醒的句子（先搞懂手头的这一套）

> if you want to avoid losing data, you--- the application developer --- need to know a lot about the internal of your database handling.

之后作者就div deep了一些conflict slove 方法

#### last write win

因为请求是 concurrent 所以我们只能用timestamp决定先后，而且这竟然是cassandra里 唯一有的一种策略。 但正如作者所说如果丢失data不可接受，那么这种方法就很不好。

#### Happens before and concurrency

作者首先告诉我们什么是 concurrent， 如果 一个 写入 casually dependent on 另外一个， 就不太算。但两个用户同时操作但不知道彼此的存在就叫并行，所以 haapnes before 和 concurrency 示范一次。



但但但但但但是这两者有时候很难分辨和判定所以我们还是需要一个算法帮我们做这个事情。

首先先上两个图吧，仔细观察一下他们就是happend before关系。

![](https://drive.google.com/uc?export=view&id=1Ia132xupzlSnTmI3VMfviKCp73HN4K6p)

我们可以看到这个系统里做了这样一件事情。

1. 维护了一个 version number，每次写入就会增加
2. 每次用户写入前会先会先读取一下latest version
3. 每次用户写入的时候读了latest version 还要有自己 之前的 read 的version然后需要merge一下
4. 最后如果要 做overwrite的选择也只会保留更高的 version

简单的看下上面 version 3， 用户先读到了自己之前的read，但他还不知道用户二在version 2加了个 eggs， 然后又读了 latest version， 之后merge 了一下 就变成了 milk flour eggs。这个需要自己看一下思考一下。

这种算法可以确保没有data会被drop掉。但用户需要做些多于操作， 比如合并后clean up 和去重，比如上面的例子我们就可以看到一些存在的重复的value。

但是这种方法如果放在删除上就不太好用了，因为我们可能并删不干净所以可能还是要某个marker 和之前提到的一样叫 tombstone。

 Riak里面有一种data type 叫 CRDTS 可以自动merge 和删除。有空研究下



#### Version vector

如果有多个 replica 我们可能用的就不是version number 而是一个 version vector， 同样的也还是要merge 所以想知道细节就去研究吧。



## 感想

这章读完，确确实实开始了所谓的 system design。里面讲的多种replication 策略都值得多多的思考和反复的想。就比如很久以前就写过类似master slave 的DB replication 作为load balancer， 但是从未想过那么多可能发生的问题以及trade off，这章读完确确实实够我喝一壶了。一边思考一边践行吧，里面还有些细节没有记下来。同样的也有很多的点可以继续dive deep 之后再回看吧。



























