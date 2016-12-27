作者：[WeiYu53111](https://github.com/WeiYu53111)


## 目录

1、Cloudera Manager是什么  
2、Cloudera Manager 体系结构  
3、安装方式1.使用官方脚本快速搭建CDH集群  
4、安装方式2.手动部署docker版的CDH集群  
5、安装方式3 只用docker版Cloudera Manager  
6、安装方式4.原生的CDH集群部署

## 1、Cloudera Manager是什么
Cloudera Manager是Cloudera公司推出的用WEB来管理集群的一套系统。用这套系统就可以通过点点点的方式来安装hadoop、Spark等集群，而不需要手动ssh到每个服务器上进行繁琐的解压安装运行程序了。


## 2、Cloudera Manager 体系结构

这里简单了解一下Cloudera Manager的体系结构，后面部署有问题的时候，就不会太迷茫。

[官方文档 --> 搜索 Architecture](http://www.cloudera.com/documentation/enterprise/5-7-x/topics/cm_intro_primer.html)

![image](http://www.cloudera.com/documentation/enterprise/5-7-x/images/cm_arch.png)

如上图所示，Cloudera Manager有5个组件构成
- agent 运行在各个节点的进程，用于监控节点、安装程序、启动和关闭进程（HDFS、Spark等）
- Server 提供了web界面，通过server把指令分发到agent中执行，负责把数据写到DataBase里面
- DataBase 存储Server的元数据
- Cloudera Repository Spark、Hadoop这些jar包存储在这里，当需要安装到某个节点的时候从这里取
- Management Service 个人理解是整合监控数据给Server，以及告警等功能


## 3、 安装方式1.使用官方脚本快速搭建CDH集群 
官方有提供Docker镜像以及脚本来快速搭建Cloudera Manager.更详细的介绍可以看看下面的网址。
[传送门](http://blog.cloudera.com/blog/2016/08/multi-node-clusters-with-cloudera-quickstart-for-docker/)

官方提供的脚本，会自动去下载镜像，并根据你的参数进行部署。脚本参数支持配置节点数，相当方便。但是该脚本只能把docker容器集群部署到一个宿主机里面，需要部署到多个宿主机需要修改脚本。因为我只是粗略地浏览过脚本，有些地方还不大理解，所以没有使用脚本进行部署，直接用方式2来进行部署。




## 4、安装方式2.手动部署docker版的CDH集群

粗略看了脚本以及看了一些官方文档后，尝试安装后的笔记。

### 4.1下载镜像

镜像使用的是官方，也就是第3节中脚本使用的镜像。primary-node是启动Server容器的镜像，secondary-node是用来启动从节点的镜像
```
docker pull cloudera/clusterdock:cdh580_cm581_primary-node
docker pull cloudera/clusterdock:cdh580_cm581_secondary-node
```

### 4.2 启动Server容器

```
docker run -h node-1.cluster -it \
--privileged \
--name cloudera-server \
-p 7777:7180 -p 9999:80 \
-v /tmp/clusterdock \
-v /etc/hosts:/etc/hosts \
-v /etc/localtime:/etc/localtime \
-v /var/run/docker.sock:/var/run/docker.sock \
cloudera/clusterdock:cdh580_cm581_primary-node \
/bin/bash
```


这里挂载/etc/hosts、/etc/localtime是为了可以直接SSH到容器里面，以及同步集群时间。
然后-h node-1.cluster是因为镜像中配置文件默认Server是node-1.cluster（镜像是别人提供的，按照别人的套路来最省事）
-p 7777:7180 -p 9999:80，7777是Cloudera Manager Web界面访问端口，9999是Hue的界面端口

#### 4.2.1 起Server服务
进入到容器里面，按照顺序起相对应的进程  
1. service cloudera-scm-server-db start
2. service cloudera-scm-server start
3. service cloudera-scm-agent start  
4. service sshd start  

对应的输出日志在/var/log/下面，如cloudera-scm-server 在/var/log/cloudera-scm-server下
Server的启动相对要久一些，虽然界面上命令执行完了，但是，后台程序还在启动，因此WEB界面需要等待1两分钟后才可以访问。




### 4.3 启动agent容器
#### 4.3.1 启动容器  

```
docker run -h node-2.cluster -it \
--privileged \
--name cloudera-node2 \
-v /etc/hosts:/etc/hosts \
-v /etc/localtime:/etc/localtime \
-v /var/run/docker.sock:/var/run/docker.sock \
cloudera/clusterdock:cdh580_cm581_secondary-node \
/bin/bash
```


#### 4.3.2 修改/etc/hosts
ip -a查看地址，增加xxxx node-2.cluster到hosts文件中
#### 4.3.3 清除一些旧的数据  

```
rm -f /var/lib/cloudera-scm-agent/uuid  
```
uuid是agent用于注册到Cloudera中的唯一标示符，不删除这个，多个容器都会映射成一个agent)  

```
rm -rf /dfs*/dn/current/*
```
datanode的标识，作用与uuid差不多


#### 4.3.4 启动agent

```
service cloudera-scm-agent start
```

#### 4.3.5 sshd服务

```
service sshd start 
```
 

#### 4.3.6 
重复上述步骤，部署多个节点，例如部署节点3

```
docker run -h node-3.cluster -it \
--privileged \
--name cloudera-node3 \
-v /etc/hosts:/etc/hosts \
-v /etc/localtime:/etc/localtime \
-v /var/run/docker.sock:/var/run/docker.sock \
cloudera/clusterdock:cdh580_cm581_secondary-node \
/bin/bash
```



## 5 安装方式3 只用docker版Cloudera Manager
- 在官方网站上有提示，不建议使用docker搭建CDH集群。
- 个人觉得像Hadoop、Spark、Hbase这些相互依赖较多，复杂的集群，搞成每个框架的进程docker化分离开，并且使容器数据共享。要实现这个目标，需要对这些框架的进程的交互输出都非常熟悉。  
- 因此想到，把宿主机以及其他机子当做方式2中的容器，也就搭建了原生的CDH集群了。CDH集群的部署苦难的地方在于安装复杂的Cloudera Manager，使用镜像版的Cloudera Manager,就不用去安装复杂Cloudera Manager，直接部署即可。
- 鉴于别人的镜像版本不对以及需要的parcel已经删除掉了，需要重新下载需要的parcel，因此这次放弃了这种安装方式，使用第4种安装方式



## 6 安装方式4 原生的CDH集群部署
官方提供了3种安装方式，安装的时候，醉心于解决bug，没有做笔记，笔记后面补上。。。












