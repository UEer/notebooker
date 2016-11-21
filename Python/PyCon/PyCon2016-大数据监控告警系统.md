## [PyCon2016演讲]大数据监控告警系统的实现  


<hr/>

[作者Github](https://github.com/EaconTang)
&nbsp;&nbsp;&nbsp;&nbsp;
[原文出处](http://blog.tangyingkang.com/post/2016/09/26/monitor-alert-system-in-big-data/)  

<hr/>

![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016.jpg)
以下内容整理自[PyCon2016](http://cn.pycon.org/2016/)深圳场的[演讲稿](http://ocgxshkaw.bkt.clouddn.com/10%E3%80%8A%E5%A4%A7%E6%95%B0%E6%8D%AE%E7%9B%91%E6%8E%A7%E5%91%8A%E8%AD%A6%E7%B3%BB%E7%BB%9F%E3%80%8B%E6%B1%A4%E8%8B%B1%E5%BA%B7.pdf)－－大数据监控告警系统。  

### 开场 
本次演讲将会一步步地，向大家展示我们这个系统架构。  
由于时间有限，我不会深入讲解技术细节（事实上我一开始做好、发给Sting的ppt有多达40页现在精简到20多页）。  
我希望达到的效果是－－  

- 对于有相关项目经验的开发人员，可以起到一个参考的作用
- 对于没有监控项目经验的人员，也可以让你对如何实现监控平台有一个快速的认知

### 背景介绍
监控系统对于大数据平台的重要性不言而喻。  
那要实现这样一种系统，我们需要解决哪些问题？  

 - 首先我们要知道如何采集监控数据，监控数据主要有三种
 	- 系统本身的运行状态，例如CPU、内存、磁盘、网络的使用情况 
 	- 各种应用的运行状况，例如数据库、容器等
 	- 处理网络上发送过来的数据
 - 有了数据，我们需要采用合适的存储方案来保存海量的监控数据
 - 然后需要把这些数据在web界面进行展示，把监控指标的变化情况可视化
 - 另外，如果监控系统只能看而不能及时发出告警（以邮件／微信等通知方式），价值也大打折扣
 - 最后，对于这样的大型架构，我们同样需要考虑高可用／高并发／可伸缩性

### 架构设计之路

接下来，我们来初步设计这个架构的实现方案。  
根据对现有监控产品的调研，以及我们列出的所需解决的问题，可以发现监控系统的__一般套路__：__采集－存储－展示－告警__，也就是图上这四个模块：  
![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-005.jpeg)  
为了实现解耦、异步和对采集器的控制，我们在采集和存储之间增加一个任务队列；考虑到可能需要进行接口封装、统一开放对外接口，我们增加一个API服务模块。  

#### 存储－OpenTSDB
我们先来看看存储方面的OpenTSDB。  
由于监控数据（例如CPU、内存等）跟时间点密切相关，我们确定了采用时间序列来存储监控数据。OpenTSDB是一个基于HBase、分布式、高可用、可伸缩的时间序列数据库，支持每秒百万级别的写入请求，并可以通过增加节点来灵活扩展处理能力。  
我们可以把它当作一个HBase的应用，利用它丰富的API和聚合函数来查询监控数据。  

它存储的数据格式包涵以下四个元素：

- Metric：指标名，比如过去1分钟的系统负载，可以表示为proc.loadavg.1m
- Timestamp：Unix时间戳，可以精确到毫秒级别
- Value：指标的数值，整型
- Tag：标签，指标的过滤条件，作用相当于SQL语句中的Where条件查询；每个指标可以有多个标签

每一条数据由以上4种数值组成，如（telnet端口发送的数据格式）：

- [metric] [timestamp] [value] [tags]

一个示例：
> proc.loadavg.1m 1234567890 0.42 host=web42 pool=static

![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-007.jpeg)  
这是它的应用场景，中间绿色的就是OpenTSDB（简称TSD），上面每个Server的c就是采集器（collector），采集器把数据发送到TSD，TSD再异步写入到HBase集群，web UI则可以通过TSD的HTTP API接口来查询数据和进行展示。

![]()
因此，在我们这个系统架构里，存储模块就是OpenTSDB模块。  

#### 采集－Collector
我们的采集器基于开源的TCollector。  
TCollector是一个python编写的、OpenTSDB的采集器客户端。它提供了一个采集器的框架，让你只需要编写简单的采集脚本，其他诸如网络连接、性能优化等工作由它处理。  
![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-010.jpeg)  
上面是它的工作原理：编写的采集器脚本，从Linux的/proc目录下获取系统相关信息，或者收集其他自定义的指标，输出到标准输出，然后有一个核心的采集器管理器统一处理输出数据，最后发送到TSD。  
这个核心管理器，其内部实现也不复杂（源代码大概1000多行），就是起两个主循环线程：读取线程ReaderThread和发送线程SenderThread。ReaderThread从采集器运行实例（也就是脚本输出）yield一行数据，将数据异步推入ReaderQueue，然后SenderThread从ReaderQueue里拿到数据，存到SenderQueue，最后发送到TSD。其中还有一些优化工作，ReaderThread负责做一些数据去重，减少一段时间内相同数据的发送次数；SenderThread负责网络连接的管理，比如与TSD的心跳检测、黑白名单等。  
我们在TCollector的基础上进行开发，包括：

- 重构，提高代码可读性、解耦模块和配置
- 增加实现代理和用户管理
- 增加与任务队列Celery的集成
- 性能优化

因此，这个系统架构的采集器模块，也有了实现。

#### 队列－Celery
Celery是一个快速、灵活、高可用、分布式的异步任务调度队列。  
集成到我们这个系统里，其实就是把采集器当成生产者，采集器生产的数据发送到Broker；Broker是消息中间件，我们选用了Redis；Worker就是消费者，消费者行为就是从Redis中获取数据，并最终写入到TSD里面。  
整个流程比起采集器直接发送到TSD会更长，但得益于Redis和Celery的高效，依然保持极佳的性能，且可以通过结合Celery－Flower这种管理界面，对采集行为进行控制。  
![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-013.jpeg)  
因此，任务队列由Celery实现。

#### API－Tornado
Tornado是一个高性能的Web服务框架，很适合构建支持高并发的API服务，而且Tornado可以和Celery整合在一起。这个Tornado API服务，我们在系统中主要用它来：

- API的封装，对TSD、Bosun（告警模块）的API进行二次开发
- 可以作为对外接口，接收处理网络数据
![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-016.jpeg)  
因此，系统架构中API服务的实现也敲定了。

#### 展示－Metrilyx
Metrilyx是基于OpenTSDB的开源可视化界面：

- 它是基于django开发的，可以很好地利用django生态的工具
- 数据展示的面板简单易用
- 对数据指标更好的查询操作
- 更丰富的指标名搜索工具
- 分布式、高可用

![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-019.jpeg)  
这是它的数据面板，左边是指标名搜索栏，右边每个小面板展示的是监控指标的图表。

#### 告警－Bosun
最后，告警这个模块，我们采用了StackOverflow的Bosun。
Bosun是一个基于OpenTSDB开源的告警系统：

- GO语言和AngularJS开发，性能好且易于部署
- 通过灵活强大的表达式来定义告警规则
- 提供HTTP调用的告警方式

![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-022.jpeg)  

### 架构全景
至此，我们基本看完了整个系统架构的技术面貌。
![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-023.jpeg)  
我们把上面的架构图再稍微完善一下...
![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-025.jpeg)  
这就是我们系统的整个架构全景。  
可以看到，在OpenTSDB节点上，我们增加了一个HAProxy，用于进行负载均衡。  
在采集器部分，还增加了一个Proxy代理。因为在大数据场景下，完全有可能是跨地区的大规模采集，这时候我们需要在不同的地域增加一个代理，用于中转处理和统一发送数据。  

整个架构可以概括为：__采集－队列－存储－展示－告警，以及协助提供模块间通讯的API服务__。

![](http://qn.tangyingkang.com/image/blog/pycon/pycon2016-026.jpeg)  
现在来看，在这个系统架构里涉及到的技术选型，Python几乎占据了半壁江山，包括采集器TCollector、Celery队列、django展示界面和Tornado。  
所以，正如本次大会的主题所说的，我们看到Python正在大数据领域发挥着重要的作用，也希望更多的Pythoner一起来分享自己的成果，贡献自己的力量。  




 	
 	