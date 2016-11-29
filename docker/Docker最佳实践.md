## Docker最佳实践

![](http://qn.tangyingkang.com/image/blog/docker/best-practise.jpeg)

根据最近使用docker的经验，总结了一些Docker/Dockerfile最佳实践。  


#### 一个容器运行单一进程
docker容器服务与微服务概念天然融合，理想的docker项目中，最好是一个容器中只运行一个进程、一类进程或一种服务。这使得系统容易横向扩展，也有利于复用容器。  
如果你有几个进程是必须丢进同一个容器运行的，那应该是你的程序设计过于集中式，没有很好的解耦和服务化。  


#### 清除不需要的程序和安装包
容器里用不上的程序，删掉；安装好程序后剩余的安装包，删掉。这可以尽量减小镜像和容器的体积，降低构建时间和传输成本。


#### 打上易读的镜像仓库名和标签
好的仓库名和标签有助于管理和发布你的镜像，这在你镜像越来越多的时候显得越加重要。  


#### 选择精简可靠的基础镜像
官方镜像debian、centos、ubuntu等都是不错的选择，一般推荐最精简的debian，体积最小。


#### 使用Dockerfile管理镜像构建过程
这是最佳实践的基本要求，没有Dockerfile，你的镜像就是一个黑盒子，接管镜像的维护人员根本不知道这个黑盒有多坑，坑了也不知道怎么维修。  
而有了Dockerfile，至少可以看到镜像构建的每一个步骤，同时还可以加快构建速度，并很好地利用docker的缓存特性。


#### 利用ONBUILD指令构建自己的基础镜像
ONBUILD指令的内容并不会对该Dockerfile构建出来的镜像有什么实质作用，但会在继承该镜像的子镜像中执行生效。  
这意味着你可以将通用软件包构建在基础镜像中，但"延迟生效"到子镜像，因此节省了基础镜像的体积和构建时间。


#### 充分利用缓存特性
通过Dockerfile构建镜像时，每一条指令是一层镜像，因此尽量保持构建指令的顺序，有助于利用缓存加速构建镜像；  


#### 减少镜像层的数目
尽量缩减指令数以达到减小镜像层数的目的，例如把多个linux命令通过```&&```连接在一个RUN指令中；
像WORKDIR、CMD、ENV这些命令应该在底部，然而一个```RUN apt-get -y update```更新应该在上面，因为它需要更长时间来运行，也可以与你所有的镜像共享


#### 使用DockerIgnore
类似于git的```.gitignore```文件，用于Dockerfile构建的上下文中忽略一些文件，使得容器更加迅速有效的加载和启动。


#### 不要在Dockerfile映射公有端口
在Dockerfile中你有能力映射私有和公有端口，但是你永远不要通过Dockerfile映射公有端口。  
如果镜像的使用者关心容器公有映射了哪个公有端口，他们可以在运行镜像时通过-p参数设置。


#### 考虑使用非root用户
如果应用服务不需要root权限也可以运行，那么就新建一个用户吧，通过类似如下命令来创建用户和用户组：
    
    RUN groupadd -r postgres && useradd -r -g postgres postgres.

#### 区分并利用好CMD与ENTRYPOINT
CMD命令的意义：
> The main purpose of a CMD is to provide defaults for an executing container.  
CMD的作用在于执行容器时提供默认的命令操作

ENTRYPOINT的定义：
> An ENTRYPOINT allows you to configure a container that will run as an executable.  
它可以让你的容器功能表现得像一个可执行程序一样

两者有类似的功能，都可以在启动容器时执行命令操作；但CMD的命令可以在容器启动时被自定义的命令覆盖，而ENTRYPOINT则不会，自定义的命令会作为参数传递给ENTRYPOINT。    
因此，实际上我们推荐把ENTRYPOINT和CMD结合使用，CMD可以作为ENTRYPOINT的默认参数。  

另外，CMD和ENTRYPOINT都支持如下几种写法：

    CMD ["executable","param1","param2"]    # 第1种写法（推荐）
    CMD executable param1 param2            # 第2种写法
要注意，第2种写法在执行时，是以”/bin/sh -c”的方法执行的命令，即如果你的CMD是这样：  
    
    CMD echo "hello world" 
容器启动时会这样执行：
    
    /bin/sh -c echo "hello world"


