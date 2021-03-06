# FLINK

**基本认知：** 

一个流式计算框架，代入Hadoop生态圈最多替代MapReduce。计算过程都在内存中，所以速度相对MR很快，中间除了shuffle基本不落盘。出现落盘要么OOM了，要么就是在检查点的时候保存一下结果状态信息。因为IO少所以速度飞快。然后就是Flink脱颖而出的最核心理念——流式计算，把无限流切成很多可计算的小块单独计算然后聚合结果。在实时响应计算方面，流式处理在无限数据流(或者说实时数据流)方面的优势是批处理无法抹平的，得益于流式的时间状态、分布式快照等等而来的容错恢复等机制更是甩了批处理几条街，所以flink做实时`牛啤`还真不是白吹的。

## 一. 重要概念

### 1.时间状态

按照先后顺序，大致可以划分为：

#### 1.1 Event Time(事件时间)：事件发生(数据产生)的时间

用的比较多。

数据产生的时候会记录下当(时)的时间，也就是事件时间。

#### 1.2 Ingestion Time(**提取时间**)：事件(数据)进入Flink窗口的时间

就没见过用这个的。

数据进入窗口的时候，会记录一个时间，就是提取时间，一般比处理时间要晚。

#### 1.3 Processing Time(处理时间)：Flink处理事件(数据)的时间

进行处理的时候的系统当前时间。当系统运行在处理时间语义下的时候，将不会管事件时间等其他因素，会在这个事件到达flink系统的时间记录下来，类似给了一个唯一的有顺序的ID，之后的操作都按照这个时间来。

计算时用能避免很多问题，但是问题就是无法保证数据的准确性(肯定没法保证啊，断网断电老鼠咬断网线集群突然宕机都会导致数据传输不按照顺序来)。

### 2.窗口机制

前：在实际生产中，数据是源源不断产生的，而且用户行为不可控，间接导致实时的场景下数据流的数据量和时间跨度也是不可控的。Flink提供了窗口的概念，将无限不可控的流划分出边界，使得计算可以进行。

**？？？**然而在这一块有一个一度困扰我很长时间的问题：窗口究竟是根据什么时间划分的？

有三种窗口的概念，而窗口的划分又有多种形式，最常见的是时间和计数。这里用时间做窗口划分举例

#### 2.1 滚动窗口（roll window）

在无限的事件流上定义一段时间为一个窗口，在此期间的所有数据在该窗口内，该窗口到临界是会创建一个新的窗口，所以最终窗口不会重叠，呈滚动效果。

比如，定义窗口size=10 s，则0-10秒内的所有数据属于一个窗口，而11-20秒的数据属于另一个窗口......这样就将一个没有边界的无限流切分成了许多小窗口，而每个窗口的值都是可以直接计算的。比如现在业务场景下需要计算一分钟内的访问量，此时对前六个窗口的数据做count处理就可以计算出访问量了。

滚动窗口的API只需要传一个参数，所以使用起来也很简单，但是在使用事件事件语义时会产生一些问题，这个后面再说。

#### 2.2 滑动窗口(slide window)

在滚动窗口的概念上，将新窗口的触发条件从达到临界值改成达到一个可自定义的步长，达到该步长即创建一个新的窗口，这样的话会导致窗口之间有重叠的部分，虽然相对滚动窗口要稍微麻烦了一点，但是却能进行更多更精细的操作。

比如场景一，窗口长度还是10，步长为5，每5秒产生一个新窗口，那么前20秒就产生了三个完整的窗口，**第四个窗口还没算完所以就没加上了**。

![1621391764465](HelloFlink.assets/1621391764465.png)

此时，业务上需要计算每20秒前后五秒的数据的数据该怎么操作？好了现在window123的结果都计算出来了,接下来：

result=w1+w3-w2

好了问题解决，当然我这只是瞎编的例子，说明用的，便于理解。

场景二，窗口大小为5，步长为10

![1621393313009](HelloFlink.assets/1621393313009.png)

此时，编一个比较离谱的业务：从第五秒开始每隔5秒会传一批持续5秒的脏数据，这时候用滚动窗口的话会接收到脏数据，造成资源的浪费，影响性能。此时滑动窗口就可以完美地解决这个问题。其实虽然这个例子很离谱，但是不同业务场景下的需求真的千奇百怪，不能以常理踱之，只能不断充实提高自己以面对各种各样的问题和挑战。

#### 2.3 会话窗口（session window）

这是一个比较特殊的窗口，它是以传入数据时的空闲间隔为依据划分窗口的，原理和session类似，但是千万不要直接理解为和session直接绑定(-_- !)。

![1621394926205](HelloFlink.assets/1621394926205.png)

如图，设置空闲间隔(正式名称叫session gap)为5秒，第一个窗口接收了10秒的数据，然后过了5秒没来新数据，就会由触发器执行生成一个新的窗口，然后第二个窗口因为用户操作比较密集，持续了30秒，然后又因为用户10秒没有操作而又开启了第三个窗口。当然这30秒内有可能存在操作空闲时期，但是每次都停下没到五秒钟就又开始了，所以没有触发新开窗口的操作，直到第三个窗口出来之前空闲了10秒，但是注意其实第三个窗口是在第十秒才开始的，也就是说空闲到达五秒的时候，触发触发器关闭上一个窗口(可以进行计算了)，然后等到第十秒来新的数据了，又开了一个新的窗口 (这里我又特地去了解了一下，其实每个数据进来的时候都会创建一个窗口，而如果这个窗口和上个窗口的间隔小于session gap也就是设置好的空闲间隔时间的时候，就会进行合并窗口操作，如果大于空闲间隔时间，就保留新开的窗口，关闭原先的窗口，然后后面来的数据放到新窗口中) 。

总之会话窗口的好处很明显，相对前两种，如果长时间没数据过来的时候会大大节省资源，但是同时，在并发量比较高的场景下，如果持续(甚至都不用大量)来数据可能会导致窗口过大的问题，一个窗口存的数据太大的话算起来就比较有压力。

但是无论用哪一种窗口都得根据业务情况来。

### 3.水印(Watermarks)

在滚动和滑动窗口中，如果数据按照事件时间进行计算的话，就不得不考虑数据丢失或者数据延时的问题，实际上在数据传输过程中很容易发生意外情况(断网断电老鼠咬断网线集群突然宕机)导致数据传输乱序。而如果窗口使用的是处理时间语义自然没有问题，但是如果使用的是事件时间语义的话，这样会导致窗口接收到的数据不全。而watermarks机制就是给这些窗口加上一个触发器和一个"盖子"，比如当窗口达到一定水位(watermark)的时候，触发器就会关闭这个窗口(盖上盖子)。这里，我用尽毕生所学将这个概念总结成一句通俗易懂的话：**“最多等你十分钟，再不来俺就走了”**是的，俺们农村人就是这么朴实无华，十分钟就是水位，我走了就是触发器()，十分钟一到不管人再不来就立马开溜。要注意的是当盖子没有盖上的时候数据依然会进入窗口，直到触发器关闭窗口为止，之前我把这一块弄混淆了，所以更正一下。首先，当事件时间语义下的窗口在工作时，比如第一个窗口需要1-60分钟之间的数据才能开始计算，但是现在1-59都来了，60却迟到了，这时候，当61进来的时候，咱们的watermarks机制会将61暂时标记为：61-10=51，这样的话没到60触发器就不会触发，一旦60他来了，触发器触发直接关掉这个窗口。但是如果等到了71都来了，60还没来，超过水位，触发器触发，那就不等它了直接关掉窗口计算。最完美的水印就是恰好知道数据最迟什么时候过来，但是实际上很难实现。

### 4.触发器

![1621408454779](HelloFlink.assets/1621408454779.png)

简单来说就是用来做执行操作的启动器，换算成代码中的概念就类似触发算子的概念。

因为Flink作为一个计算引擎，最终是要对数据进行计算的，而计算的规则或者说方法就要通过窗口函数调用各种算子来执行，而程序只有碰到触发算子才会真正开始执行。就比如在上文达到水位以后，如果触发器坏了，那么不管水涨的有多高窗口也不会关闭。就好像在水位处放了一个通着电的开关，一旦水涨到这里了，通上电了，啪一下，很快啊，盖子就盖上了。

废话不多说了，想要弄懂触发器就要到代码里面去看，今天就先不看了。

### 5.checkpoint(检查点)

其实我更喜欢另一种叫法：**分布式全域一致快照** 

喜欢打游戏的人都知道存档功能，随手存档是个好习惯。不过很多游戏都会有自动存档过程，这个过程就类似checkpoint功能。比如在一段长——长的过场动画后，如果没有自动存档的话，一旦在结束的时候卡死，下次进来就要再看一般，非常地反人类，但是一旦有自动存档下一次就可以继续体验内容，而不用再等一次过场动画了(除非真的又大又好看)，一般人是等不了第二次的，如果等第二次就会很暴躁，但是程序不会，它只会默默地等待，让坐在电脑前的程序员非常暴躁。说实话下午啥事也没干一直在写这个我也是有点烦了，一直码字还得查文档验证一些说法有点枯燥，不过过程中收获也有很多，就换种说法娱乐下自己。

怎么解决这个问题呢？回归正题，比如现在有一个比较大的计算任务，咱们分成10000个窗口分别执行计算然后merge结果，但是有点大，内存装不下了，这时候就可以每100个窗口的计算存一次盘，然后再把存下的100个小文件中的结果合并一下就可以了。

再比如，现在好好地，每算到100个窗口任务就挂一次，然后重启机制触发再启动重新算，最后等一个小时发现进度还在10%，明显有问题，现在发现这个节点很有规律地每十分钟挂一次，然后又神奇地好了。怎么解决呢？好了，现在从1算到100的时候要用九分钟，很好，我们在这时候存一次盘，然后等第二批计算的时候挂掉了，但是不碍事，我们这次从101开始算......最终我们的任务是可以完成的，顺便给运维找点事做，多好。

checkpoint用到的地方很多，但是概念都大差不差。

### 6. state(状态)

Flink会开辟一块内存空间用来记录状态、快照、offset等信息，以支持窗口、时间、水印、检查点等正常运行。

### 7. offset(偏移量)

单独拿出来说是因为这个概念在数据安全方面几乎是现有的解决方案中的最优解，不管是kafka还是flink一旦涉及到数据安全必然会提到偏移量这个概念。那么什么是偏移量呢？无论在何种形式的数据传输中，数据进出必然会有记录，而两次记录的差值就被称为偏移量。为什么我会觉得offset如此重要？比如，现在我有一个任务，系统会记录下当前的offset，一旦任务挂掉了需要重新计算，现在我可以拿到系统的这个offset，然后从这个位置开始计算，这样就能避免数据丢失的问题；又比如，现在有一个很大的任务，我checkpoint了一下，顺手存了个offset，然后任务又挂掉了重算，这时候，我可以根据当前记录的offset直接进行后面的计算，因为前面的计算结果已经落盘了。



## 二. 集群架构（纯手打文字版）

如果是需要搭建一整套的流式处理系统的话，首先就得配合业务流程，但是如果抛出业务只谈数据流的话，其实架构都差不多：

1.最原始的数据源，第一种是业务数据库里面的数据，比如mysql，Oracle等等，公司APP或者业务操作产生的数据大部分都在这里了；第二种是前端埋点产生的日志文件，通过数据收集服务器(Nginx)存储

(生成数据)

2.flume一般用来拉取日志文件，可以从Nginx上通过source 抓取日志文件，通过channel传输然后sink到消息中间件kafka中；mysql的数据也可以用canal抓取到kafka中，用于计算。

(存)

3.flink从kafka拿到数据实时计算，将结果存到mysql等数据库中，并由后端人员编写接口，提供给前端人员

(算)

4.前端做可视化，在页面展示或者做成报表。

(做成老板能看懂的样子)

## 三.为什么要用 Flink

### 大数据起源：

事情还要从盘古开天辟地说起，话说当时传统数仓已经可以做到很多事了，但是当数据量大到一定程度的时候，传统的关系型数据库已经不适合用来处理超大规模的历史数据。后来有大佬(道格卡丁)开发了大数据框架用来处理大数据，也就是Hadoop框架。最早是用Hadoop中的MapReduce来进行数仓的计算处理的，核心思想是分布式的分而治之，将大量的数据计算分成很多小的任务分配到集群的节点上进行计算，最后将计算结果汇总得到最终结果。然后据此又根据业务场景衍生出了两种数仓的概念：离线数仓和实时数仓。字面意思，离线数仓一般数据固定，整个拿去慢慢算，特性就是有延迟性，一般用`Spark`做；而实时数仓的数据是实时传送过来的，有些时候业务甚至要求毫秒级响应，一般会考虑用`Flink`做。

### 对比：

Spark是批处理计算框架，SparkStreaming是用批来模拟流达到批流一体的效果。

Flink是基于流式处理的计算框架，DataStream就是主要的原生接口，开启流式计算，而DataSet是用流来模拟批，达到流批一体的效果。(后面把DataSet并到DataStream里面了)

**批处理**：同时开启大量任务同时处理计算，讲究一个量大。

**流处理**：将数据源切分成时间连续的小块计算，讲究一个持续。

这里比较一下就能看出来，如果数据源已经确定，就干脆一次性建多个任务给他算，算完了结果一整合就完了。如果数据源不确定就用流，过一会切一下，算一次，然后等下一段过来再切一下，算一次。

**批流一体**：Spark用批来模拟流是怎么操作的？首先，把流当做一条直线，它在时间、空间上都是连续的，但是如果我批建的足够快也能达到一样的效果。比如，原本我批是一次性开10个计算，现在我不这么做了，我隔一秒开一个计算，一直往下开，假设每1秒我关掉一个批，那么在时间上上个批和下个批是相连的，在空间上我们把它一拍，它就连续了，，，开个玩笑，数据传输是乱序的一般没有空间的说法，Spark对这种情况没什么好的解决方案(也就是说Flink有)。好，现在从结果上看，每秒会有一个管道接收数据，然后关掉拿去算，下一秒又开了一个新的管道，然后拿去算......从结果上看已经和流没有任何区别了。

```txt
我自己理解为：流是一条横线，而批流是一条斜线。
```

**流批一体**：那Flink又是怎么用流来模拟批的？简单暴力，就是当流中的窗口开的足够小的时候，其实可以视为同时有多个计算同时进行。假设，每个任务要算10秒，而每一秒开一个新窗口计算，这样的话在一个10秒的区间内会有10-11个任务在算，如果忽略这1秒的差距话，可以视为计算在同时进行。

```txt
我自己理解为：批是一条竖线，而流批是一条斜线。
```

一体的意思就是都有，别忘了。

差异说完了现在说说为什么要用Flink。第一章的概念这时候就用上了。

简单来说，Flink基于watermark和trigger提供了乱序数据处理的能力，同时加上state和window提供了强大的容错能力，再加上state和checkpoint提供了无敌的数据恢复能力。

**场景一**：使用EventWindow的时候，如果现在1-9号的文件之中缺少了8号，那么到9号到达时候这个窗口并不会关闭，而是再等待一定的时间，时间到了再通过触发器关闭窗口，这个期间如果8号到了，自然皆大欢喜，我们的保障起作用了。但是如果还没有到，那没办法，我们已经等了一段时间了，再等下去就不划算了，于是由触发器执行关闭窗口。

**场景二**：窗口计算到一半的时候任务挂掉了。首先state中会保存窗口的相关信息，如果此时任务异常关闭，offset是不会更新的，而下次计算会从这个窗口之前的offset开始计算，也就是说重新算一遍，这就是容错和恢复机制。当然这一套没有那么简单，checkpoint也会提供定时快照以保证故障以后任务重新运行能有个依据。

**场景三**：现在整个任务已经算了一天了，马上就要出结果了，但是突然断电了。要是没有保障机制的话恐怕这一天白算。但是之前聪明的工程师已经设置了，每隔10分钟计算一下结果，并用checkpoint保存一下。那么之前计算的所有结果其实都已经存盘了，就算集群宕掉了，我最多只需要多计算10分钟的任务。

然后Spark没有这玩意儿，Flink有。现在告诉我为什么要用Flink？

对了，Flink还有一个比较大的优势：算得快这个也是重要原因。

## 四. Flink的计算流程

