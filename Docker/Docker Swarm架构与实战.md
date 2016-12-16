## Docker Swarm架构与实战

作者：[EaconTang](https://github.com/EaconTang) | 文章出处：[博客链接](http://blog.tangyingkang.com/post/2016/11/15/docker-swarm-architecture-n-practice/)  

----

### Swarm架构
Swarm的架构比较简洁，一个服务发现模块，一个Master节点的Scheduler，以及集群的其他Slave节点。  
其中，Master节点上运行swarm manager，包含一个核心的Scheduler模块，客户端的命令到达Scheduler，首先被过滤器筛选，得出符合过滤条件要求的节点和容器，然后再根据设定的调度策略来选择最合适的节点进行分配。  
Slave节点包含一个docker daemon程序，运行的容器进程，以及服务注册。   

大致架构图如下：
![](http://qn.tangyingkang.com/image/blog/docker/swarm-architecture20.jpg)

Swarm还声称__“swap, plug, and play”__，采用了灵活可插拔的架构模式，所以你可以自由选择不同的调度系统、配置管理和服务发现工具，支持包括etcd/consul/zookeeper。当然，你也可以只用一个文件列出所有节点来简单管理(上面的部署示例即是)。  
Swarm也可以运行在Mesos这样的大规模管理框架之上，Mesos和Mesosphere DCOS有能力让Swarm和其它应用运行在同一个集群上，包括大数据应用，如Spark、Storm、Kafka以及Hadoop。这也提高了资源的使用，同时减少了消耗和复杂性。  
所以我认为这是Docker的一个明智选择，给了用户更多的选择余地。__这也是为什么Docker自带Swarm，与当年当年Windows自带IE的行为有着很大的不同。__  

最近的在项目中，刚好用了Swarm与Mesos结合的容器管理方案，顺便上一张整理的架构图：
![](http://qn.tangyingkang.com/image/blog/docker/swarm-mesos-1216.jpeg)


### Swarm过滤器
Swarm过滤器用于创建和运行容器时，选择符合条件的节点和容器。有以下两类、共5种过滤器：  

- 节点过滤器：
    + Constraint（约束）：是和节点相关的键值对，用于将容器指定运行在符合标签的节点，例如```-e constraint:operatingsystem==centos7```，指定操作系统为centos7的节点
    + Health（健康）：显然，让容器只在节点状态健康的机器上运行
- 容器过滤器：
    + Affnity（紧靠）：运行在指定容器所在节点，例如``` -e affinity:container==mysql```表示让容器运行在mysql容器所在节点
    + Dependency（依赖）：调度同一节点中相互依赖的容器，例如让彼此links的容器运行在同一节点
    + Port（端口）：挑选端口可用的节点

### Swarm调度策略
Swarm自带的调度策略有3种：Spread/Binpack/Random

- Spread（默认）：优先选择占用资源（CPU、内存等）最少的节点，均衡负载集群
- Binpack：与Spread相反，尽量选择同一节点，负载集中在集群的少数几台机器上
- Random：随机选择节点，一般指用于开发测试

### 实战部署
以下作为示例，搭建3个实验节点的linux，2个centos7和一个ubuntu16，地址分别是：

    as7-01-master   192.168.78.131  
    as7-02-slave    192.168.78.132  
    ubuntu16-slave  192.168.78.134  

__1) 所有机器上安装docker__  

__2) 所有机器的docker daemon监听2375端口__   
如果是直接拿docker二进制包部署启动的，可以这样启动dockerd：  
<pre>dockerd -H 0.0.0.0:2375 -H unix:///var/run/docker.sock</pre>
如果是软件包、apt-get/yum安装之类的，修改配置文件(Ubuntu 上是```/etc/default/docker``` ,其他版本的 Linux 上略有不同)，在文件末尾加上：  
<pre>DOCKER_OPTS="-H 0.0.0.0:2375 -H unix:///var/run/docker.sock"</pre>

__3) master机器上，拉取swarm镜像__  
<pre>docker pull swarm</pre>

__4) maste机器上，记录所有节点地址__  
将它们的地址和端口以如下形式保存到文件swarm_cluster:  

    192.168.78.131:2375  
    192.168.78.132:2375  
    192.168.78.134:2375  

__5) master机器上，运行swarm的manage指令，将swarm_cluster文件挂在为```/tmp/cluster```，并将swarm容器里的端口映射出来(例如：2375->2376)__  
<pre>
docker run -d --name swarm_master -p 2376:2375 -v $(pwd)/swarm_cluster:/tmp/cluster swarm manage file:///tmp/cluster
</pre>

__6) 检查swarm容器运行信息__  
发往Docker Daemon默认的2375端口, 输出：  

    [root@as7-01 docker]# docker ps
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
    702d5b4e335d        swarm               "/swarm manage file:/"   3 hours ago         Up 23 minutes       0.0.0.0:2376->2375/tcp   swarm_master

发往Docker Swarm容器的2376端口, 输出：  

    [root@as7-01 docker]# docker -H 192.168.78.131:2376 ps -a
    CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                           NAMES
    702d5b4e335d        swarm               "/swarm manage file:/"   3 hours ago         Up 24 minutes       192.168.78.131:2376->2375/tcp   as7-01-master/swarm_master

__7) 检查整个docker集群信息__  
发往swarm容器的端口2376，执行如下命令：
<pre>
docker -H 192.168.78.131:2376 info
</pre>
可以看到三个节点的docker运行状况， 具体输出如下：  

    [root@as7-01 docker]# docker -H 192.168.78.131:2376 info
    Containers: 1
     Running: 1
     Paused: 0
     Stopped: 0
    Images: 2
    Server Version: swarm/1.2.5
    Role: primary
    Strategy: spread
    Filters: health, port, containerslots, dependency, affinity, constraint
    Nodes: 3
     as7-01-master: 192.168.78.131:2375
      └ ID: YC2M:G72E:KAZW:KFD4:5ZRV:HH35:KJ5H:ABJX:4Y3U:QB43:E5PI:W6ZK
      └ Status: Healthy
      └ Containers: 1 (1 Running, 0 Paused, 0 Stopped)
      └ Reserved CPUs: 0 / 1
      └ Reserved Memory: 0 B / 1.003 GiB
      └ Labels: kernelversion=3.10.0-327.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), storagedriver=overlay
      └ UpdatedAt: 2016-12-15T14:55:12Z
      └ ServerVersion: 1.12.1-rc1
     as7-02-slave: 192.168.78.132:2375
      └ ID: EJJD:KTZ4:W4M4:ODAP:5YKG:2YJ4:ZXLF:P5IZ:IT5E:3MBZ:CAC7:RAQX
      └ Status: Healthy
      └ Containers: 0 (0 Running, 0 Paused, 0 Stopped)
      └ Reserved CPUs: 0 / 1
      └ Reserved Memory: 0 B / 1.003 GiB
      └ Labels: kernelversion=3.10.0-327.el7.x86_64, operatingsystem=CentOS Linux 7 (Core), storagedriver=overlay
      └ UpdatedAt: 2016-12-15T14:55:46Z
      └ ServerVersion: 1.12.1-rc1
     ubuntu16-slave: 192.168.78.134:2375
      └ ID: ORHV:ZBIE:VESL:7GF3:BZYE:6VE2:7F44:QYKC:O55U:KQHQ:5RY2:QLMU
      └ Status: Healthy
      └ Containers: 0 (0 Running, 0 Paused, 0 Stopped)
      └ Reserved CPUs: 0 / 1
      └ Reserved Memory: 0 B / 1024 MiB
      └ Labels: kernelversion=4.4.0-21-generic, operatingsystem=Ubuntu 16.04 LTS, storagedriver=aufs
      └ UpdatedAt: 2016-12-15T14:55:27Z
      └ ServerVersion: 1.12.1-rc1
    Plugins:
     Volume:
     Network:
    Swarm:
     NodeID:
     Is Manager: false
     Node Address:
    Security Options:
    Kernel Version: 3.10.0-327.el7.x86_64
    Operating System: linux
    Architecture: amd64
    CPUs: 3
    Total Memory: 3.005 GiB
    Name: 702d5b4e335d
    Docker Root Dir:
    Debug Mode (client): false
    Debug Mode (server): false
    WARNING: No kernel memory limit support
