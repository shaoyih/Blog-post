# Encoding and Evolution

首先开篇作为引子, 拿 schema on write 和 schema on read 为开始然后提到如果数据库的改变 会引发代码的改变, 但很多时候改变没有办法瞬间执行. 所以只能采取一种策略叫rolling upgrade. 在后端程序上我们只能让部分node 执行新的改变,剩下一部分执行旧的. 此时新旧交叠的情况出现就会产生一个问题. 前后, 新旧 兼容的问题.



这章主要是讲，encoding 和 decoding 还有 evolution 的关系， 其中介绍了几种比较常见的 encoding format: Json, XML, Binary Variant. 之后介绍了一些binary 的压缩协议例如 Thrift,  Protocol buffer, Avro.  以及分析了各自的利弊. 让我们看到了 他们在evolution 上的灵活性.  add something later



> **Backward compatibility** : newer code can read data that was written by older code.

>**forward compatibility** : older code read data that was written by newer code.

相对来讲Backward 比 forward 相对更简单 更好实现一些. 这两个概念对于本章比较重要.



## Format for encoding data



### data representation

1. application level, 如code 里的一些 data structure.
2. encode level, 如保存到file 里的 sequence of bytes.

然后作者 把这个 两个level 的transition 称为 encoding 和 decoding 的过程.

作者还把encode 联系到 serialize 还有 decode 联系到 deserialize. 同时留下了一个伏笔, 在contest of transaction 里的 serialization究竟代表什么.



### Language specific formats

很多语言都提供自己的 encoding libraries 但是 他们存在着很多的**问题 :**

1. encoding 工具 和 语言紧密相连. 不方便其它语言使用.
2. decode 过程会产生些随意的 object 可能 被截获了的话会产生一些安全问题.
3. 为了增加速度而不再去关兼容性的问题.
4. 即使是这样, 它们的performance 还是一样的差.

作者最后还形容这些工具包.

> ​	For these reason it's generally a bad idea to user your language's built in encoding for anything other than very transient purposes.



## Json XML and Binary variants



### Json

- 因为其被浏览器支持所以很火
- 无法分辨整数和小数, 更没有精度一说. 给了一个 twitter id 太大需要传两次的例子
- human readable
- 不支持 binary string 的转换, 需要带转换schema size 会加大 33%
- 有个 optional schema 但是太复杂了



### XML

- 被称为废话太多
- 无法分辨 数字 和 string
- human readable
- 不支持 binary string 的转换, 需要带转换schema size 会加大 33%
- 有个 optional schema 但是太复杂了



### CSV

- 无法分辨 数字 和 string
- 没有 schema, 由application 决定 meaning.



想要了解请自行谷歌一下更多细节.

虽然这些format 有各种问题,但是他们依然会流行, 因为他们 作文 一种 data interchange format. 作者还提到一点

> As long as people agree on what the format is, it often doesn't matter how pretty or efficient the format is.



### Binary encoding

这章比较关键的 一个 format 出现了, 去 Figure 4 -1 看图和json 对比.

这章是encode 前后的对比 可以看到他们并没有做到更好, 于是就推出了两种新的 encode 方式.



### Thrift 和 protocol buffers

这是谷歌和脸书推出的两个 encode 方式.

其主要方式是通过提供一个 带着 id schema 来 determine. encoded form.

thrift有两种做法:

1. 把每个部分放进去 比如type, field, length, value. 看 figure 4-2 

 	2. 和上面差不多, 把field 和 tag 拍成 一个byte, 然后让数字使用 variable byte. 看 figure 4-3

protocol buffer做法:

1. 和上面差不多, 把field 和 tag 拍成 一个byte, 然后让数字使用 variable byte. 看 figure 4-4



#### evolution and field tags

- 可以改field 名字 因为他们跟 id 跑
- 旧的 code 读 新的 code 写的东西时, 如果 无法识别 就会忽略 -> 维持了 backward
- 但注意这里我们只能用 optional 当新field 的选项, 因为 旧的code 读 新的code建立的时候会fail 看到 required.
- 新的code 始终可以读 旧的 -> 维持了 forward
- 反过来我们只可以remove optional.



#### evolution and field tags

- 可以改但是会产生问题 比如 32位变64位
- protocol buffer 没有list直接用了 repeated 好处是可以改 field option
- 但是 Thrift 有list 可以支持 nested list



### Avro

与上面两种方式相比Avro 拥有更加灵活的schema配置。首先他有两种shcema。

1. Avro IDL 给人编辑用的
2. 另外一种是machine readable
3. 

他有几个特点：

- 分 reader schema 和 writer schema， 顾名思义。
  - 不一定要一样但一定要可以兼容。
  - 可以顺序不一样。
- 因为这里没有前面所说的那种 optional 所以 只可以加入或者删除field 有 default value的，为了前后兼容。
- 更改type ok， 更改 name half ok。
- 可动态生成schema， 关键就是没id 可 dynamic。



但因为schema 没有id所以我们需要考虑 reader schema 是怎么identify field 被 writer schema写入的。分几种情况。

1. 整个文件只用一种schema， 所以把writer schema 放在开头。
2. 不同writer 写入的时候 就可以用 version number 来 决定。
3. 当有网络交流的时候，可以通过schema version在传输开始前决定。





###  好处

1. compact
2. schema 可以当作doc 一样来使用
3. 支持 前后 兼容
4. 有了schema 可以支持 compile time check.



## Modes of Dataflow

Data flow 是指每次发送数据从一个proces 到另外一个 process， 所以data 从一个 flow 到另外一个，这节主要是讲三个层面的data flow。

1. 数据库
2. service之间的
3. 异步 message 传输



### 数据库

这个很显然就是  数据库 层的传输。所以设计者要自己考虑前后的兼容性。特别需要注意的一个问题就是

新的field 加入 -> 旧的application code 读了 并且 update 了 -> 新的 那部分可能就会被 override了。

通常情况下数据库如果有改动是不会直接彻底迁移而是做一下小改动，比如加了一个column 但是 会自动设置成 null。 

归档保存这些数据库数据的时候通常会用最新的 schema 所以一些 前后兼容的问题需要考虑。



### Service 层的流动

现今 比较流行的有两种方式， 一种就是我们今天广泛使用的 rest 还有 一种 就是 RPC. 前面有关于 Rest 的 介绍就是老生常谈。首先作者花了 一些篇幅来区分 Rest api 和 SOAP api，简而言之就是soap 直接通过WSDL 直接access remote 的 class 和 method call。Rest 就是尽力往 http 协议上靠， 虽然 soap 也是用http 传输 但是他们 的目标是尽量远离http， 后面作者花了一些篇幅介绍了RPC 的问题。因为问题真的很大：

1. 第一就是问题是out of control，首先本地的function出了什么问题不会包含一些网络问题等，所以不可预测。
2. 很多时候远程 拿不到 response 并不知道是什么原因，其实和上面的问题差不多。
3. 有些时候 code 在另一端收到了， perform了 但是传回来的时候 response 丢了， 通常会再次发送请求这种时候需要考虑去重的问题。
4. 慢
5. 在local 可以直接reference 但是远程必须转成byte sequence，但如果object 很大很复杂就很麻烦。
6. 然后因为是不同的sender 和 recipients 很多时候可能并不使用一种语言，所以说 转换也是个问题。
7. 其实存在的问题不仅是这些，应该还有很多。

综上所述就知道为什么rest 会流行了，虽然还有跟多原因。



- 后面作者介绍了 prc 的未来发展，虽然我并看不到它的未来。 比如 gprc， 就是 prc 加上 protocol buffer， 然后 支持stream 的 操作就是同时可以进行多个request和 response的操作， 但还是看不到未来 ：（ 即使是自制的 prc 有更好的效率， 但他们并不支持rest 那种好debug的特性。



- prc 的evloution需要是双向的， provider 和 servicer 都需要 upgrade 才行，但他们又有organizational boundaries， 没法强制别人 upgrade。



- 但rest 就可以直接带version number 在 url里。



所以得出一个结论，内部可以使用prc，但外部就老老实实的使用rest吧！



### Message passing  流动

简而言之就是支持异步的操作，作者说更像 prc 和 数据库间的一种流动。首先介绍了几个好处：

1. 可以当作buffer， 使得程序 更 reliable，message queue里缓冲一下。
2. 可以自动传输到某个 程序 之前crash过也没问题。
3. sender 不需要知道 recipient 的地址信息。
4. 可支持一对多传输
5. 更好的解耦了 sender 和 reciever 的关系。

但它通常有两个 特点 单向， 支持 异步。

通常有两种模式：

1. queue 一对一 
2. topic 一对多



#### actor 框架

作者最后介绍了一个之前没听过的框架 叫 actor， 核心思想就是一样。在一个process 里 可以同步的 communicate， 封装了一个逻辑， 就是 每个 actor 有一些 state 可以互相communicate， 互相传输message。但是有一件事不能保证那就是message可能会丢失。



在分布式actor框架里， 这种model是为了让程序scale 更大。 作者这么定义

> A distributed actor framework essentially integrates a message broker and the actor programming model into a single framework.

但我们有rolling upgrade的情况出现的时候我们还是要考虑 前后兼容的问题。



作者提到了几个框架 之后可以去研究下：

1. AKKA
2. Orleans
3. In Erlang OTP



## 总结

这章总体 来说就是介绍了 data encode 和 evolve 的关系， 通过几种 不同的 模型 还有一些case 解释了 前后兼容的问题。 然后同过介绍了几个不同的data flow 来阐述这些 也可能存在。



## 感想

这章看完， 主要在我脑子里构建了一个data 和 evolve 之间的关系很重要这些都是我们 需要考虑的。在第一章讲到evovlbility 的时候 我并不知道具体需要考虑的 问题。 文中 介绍了几种 binary 的压缩方式是比较特别的， 特别是Avro. 因为感觉其好处很多应该为很多database 结局了 类似前后兼容的问题 之后可以深入研究一下。 后面介绍到data flow 的时候比较浅和老生常谈，顺便加深了我对RPC 的bad impression。这章整体比较短 和 浅，前半部分是重点。后面章节应该会有很多related 的话题研究。







​                                                                                                                                    







 









