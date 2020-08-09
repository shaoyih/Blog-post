# Data Models and Query Language



​	数据模型决定问题解决的方式，不仅是其写法。不同的数据模型，通常用于解决不同的问题，应用于不同的场景根据其优劣。我们现今数据模型结构通常是层层传递。比如书中所说  ***API 对象 -> 数据库类型  -> 存储格式 -> 硬件存储*** ， 可能结构会比这个层数要多但是每层都会为上一层隐藏当前层的细节使之简单化。

> Most Application are built by layering one data on top of another.

所以对于 data model选择甚是重要，它决定了我们程序性能如何。 当下比较流行的主要是两种模型 Relational model 和 document model。

## Relational Model  VS.  Document Model

### SQL 起源

-  起初只是一个理论“把数据按照其关系存储”，没人知晓成功与否。

-  起初的应用场景是

   - “transaction process” : 银行交易， 航空预约 等

   - “batch processing”： 工资单

-  起初目的就是为了隐藏细节， 使操作清晰

-  后面就随着电脑发展一起发展了起来

### NoSQL 诞生

* 可以被称为 Not only SQL， 别误解 其名字。

* 2010年后 NoSQL 开始尝试推翻颠覆 SQL。 起初是为了满足几项需求， ***开源， 分布式，非关系型***。

* 以下是NoSQL 能发展起来的驱动力在我开来都是其发展起来的点

  * 可以处理的更大规模数据集，支持更大的通量
  * 开源+免费
  * 可以写一些骚操作的query，SQL 很难或者说是没法做到 因为限制太多了
  * 更灵活，更好表达的数据模型

  

### 应用代码层 与 relational model

- OOP 里 object 与 relational model 属实有隔阂。
- 虽然ORM 减少了这种隔阂，但还是有些差别。深有体会。。。

### one to many

#### 	SQL 

1. 早期的SQL只是单纯的支持 separate tables 和 foreign keys

2. 后面的SQL衍生出了 column 里 放 multi value 的 datatype

3. 把 muti-value 放到一个 json 或者 xml 的文件

4. 需要多个 query， 处理不便

     

#### 	Document

1. 直接放到 json

2. 很多时候 一个 query 搞定

   

### document 和 relational 差别

relational db 里分表的好处显而易见。 用作者的一个比喻就是数据只存在一个地方，大家需要的时候就根据refrence来找他，记得大学里学过的 1NF， 2NF, 3NF, BCNF。这些normalization的一个作用就是为了更好的去重，但同样的因为分表会越变越多，所以会耗费很大一部分的精力在join上当我们需要搜寻的时候。

相比之下 Document DB 差别就在于数据的重复性， 以及必要的时候需要update。因为相关数据很多时候会写在一起所以一些document database 并不支持 join。 所以发生update的时候 需要手动解决 或者 创造 更多重复的数据。

这在读和写上造成了很大的差别。



## Document database 的历史

早期诞生了两个 model

1. Hierarchy model  [one parent]

2. network model （CODAYSL）[multiple parent]

   - 在此 模型下 data point之间的传递更像是 c 里面的 pointer 而并非foreign key
   - 所以多条路径可以通向一个 节点
   - 对于数据库来讲 寻找某个信息 更像是寻找一条路径， 需要perform 一个 cursor 同时 还需要保存各种信息
   - 导致数据库相当的 负责和不灵活

   

相比之下 relational database 就比较容易

- 存在 row 里，并且还没有任何层叠关系。
- 所有东西交给自己的 query optimizer 来处理， 虽然优化器本身很复杂但是设计好后就没有太多后用之忧。
- 如果需要一种新的方式来提取，只需要换一种index的方式。

但是对于 relational 和 document 来讲 面对 many to one 和 many to many 并无本质处理差别。



### 如今document 和 relational的对比

- 与relational 相比 document 更贴近业务代码
- 但需要提取某个特殊数据的时候需要一层层的叫（但不是啥大问题）
- 不支持 join （是不是问题取决于 应用场景 比如 many to many 可能就不太行）
- document 因为没有 relational 严格的schema 限制， 所以很多时候添加比较灵活。被称为schema less
- 所以衍生出两个术语
  - schema on write 在写的时候已经定义好了data 形式 （更像 compile time）
  - schema on read 在读的时候才真正的解释好data形式 （更像 run time）
- 所以写入或者发生更新的时候，relational 就需要大费周章 的去 migrate。
- 本地性为 nosql 带来了 太多便利，主要的好处就是 在这个 doc上 如果在同一时间 需要很大一部分的数据，就可以满足。但是写起来就很痛苦。
- 所以小小建议是 尽量做小的改变， 消减document size。

但随着时间推移 慢慢 一些relational mode 也 支持locality的做法 比如 

- 谷歌 的 spanner database 允许一些row 和其它table的交织，具体得研究下。
- oracle的 milti-table index cluster
- column family家族 如 cassandra 就是一种 big table model， partition只搞专门的 column。（好好研究下）

所以说啊这俩越来越像取长补短相得益彰。



## Query Language 



### declarative query

- relation algebra 写过太多了
- 相对固定化的规则，需要提供条件。
- 需要决定数据按照什么形式输出
- 功能上的限制比较多，英文要交给优化器来处理。
- 好看但是限制多

### Imperative query

- IMS & CODASYL
- Perform 某些操作 按着某些顺序, 类似于写段代码进行提取
- 很难对其进行并行操作，因为像application code 一样有顺序的关系。
- 灵活但是难看

###  Map Reduce

- 通常分为两步 Map 和 Reduce.
- Map 就是收集符合相关条件的数据.
- Reduce 就是合并减少.
- Map reduce functions 通常情况下就是纯方程，不允许多余query。
- 属于相对底层的model
- 需写两个function所以不太好用，但是类似mongo db有declarative的 aggregation pipeline可以做类似的事情。

> The moral of the story is that a NoSQL system may find itself accidentablly reinventing SQL, albeit in disguise.



## Graph Like Model

多对多的关系不管是在 SQL 还是 NoSQL都一样难搞，此时graph database 就出现了。其中很重要的一个点就是graph的数据类型没有限制，所以节点和节点之间的连接就很灵活。几个比较重要的特性

1. 没有shcema的限制
2. 很容易找到related edges
3. 一个graph可以存多种data type的数据



### Cypher QL

一种由Neo4j 创造的 declarative ql, 支持一些recursive的call 可以一直沿着path寻找数据点

与之相反

如果把类似recursive的操作 在SQL里会写的又臭又长。



### Triple stores QL

类似于 【subject， predicate， object】

predicate 可以是 

- property （George age 10）object 会变成 value

- edge (Alex likes Sara) object 会变成 vertex



#### semantic web && RDF

- 曾应用于website 为了保证一致性的输出格式，这样其他的网站也可以接受相同格式的数据，产生一个 “database of everything”
- 和triple store并不是一个东西
- 失败了
- SparkQL 是作用于RDF model 的三段式QL
- SparkQL 也是cypher QL的借鉴源泉， 所以两者写起来结构性很相似。
- 

#### Datalog

- 比sparql还要古老的三段式 predicate （subject， object）
- 相对复杂写个 recursion都需要定义非常严格的rules
- 看看就行了



### Network Model VS. Graph DB

看似结构性都是多对多但是还是有很多差别的。

1. schma的限制
2. network根据path 搜寻， graph 根据id
3. network children会是一个ordered set， graph 完全没有
4. network只支持imperative， 但 graph两个都支持















 







 







## 







