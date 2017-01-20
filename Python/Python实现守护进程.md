## Python实现守护进程

作者：[EaconTang](https://github.com/EaconTang) | 文章出处：[博客链接](http://blog.tangyingkang.com/post/2016/10/20/python-daemon/)  

----

### Daemon场景
考虑如下场景：你编写了一个python服务程序，并且在命令行下启动，而你的命令行会话又被终端所控制，python服务成了终端程序的一个子进程。因此如果你关闭了终端，这个命令行程序也会随之关闭。  
要使你的python服务不受终端影响而常驻系统，就需要将它变成守护进程。  
守护进程就是Daemon程序，是一种在系统后台执行的程序，它独立于控制终端并且执行一些周期任务或触发事件，通常被命名为"d"字母结尾，如常见的httpd、syslogd、systemd和dockerd等。  

### 代码实现
python可以很简洁地实现守护进程，下面给出代码和相应注释。这份代码稳定运行在我本地电脑的一个守护进程([自制闹钟](https://github.com/EaconTang/python-tools/blob/master/clock/alarm.py))里，暂时没出过问题。

    # coding=utf8
    import os
    import sys
    import atexit


    def daemonize(pid_file=None):
        """
        创建守护进程
        :param pid_file: 保存进程id的文件
        :return:
        """
        # 从父进程fork一个子进程出来
        pid = os.fork()
        # 子进程的pid一定为0，父进程大于0
        if pid:
            # 退出父进程，sys.exit()方法比os._exit()方法会多执行一些刷新缓冲工作
            sys.exit(0)

        # 子进程默认继承父进程的工作目录，最好是变更到根目录，否则回影响文件系统的卸载
        os.chdir('/')
        # 子进程默认继承父进程的umask（文件权限掩码），重设为0（完全控制），以免影响程序读写文件
        os.umask(0)
        # 让子进程成为新的会话组长和进程组长
        os.setsid()

        # 注意了，这里是第2次fork，也就是子进程的子进程，我们把它叫为孙子进程
        _pid = os.fork()
        if _pid:
            # 退出子进程
            sys.exit(0)

        # 此时，孙子进程已经是守护进程了，接下来重定向标准输入、输出、错误的描述符(是重定向而不是关闭, 这样可以避免程序在 print 的时候出错)

        # 刷新缓冲区先，小心使得万年船
        sys.stdout.flush()
        sys.stderr.flush()

        # dup2函数原子化地关闭和复制文件描述符，重定向到/dev/nul，即丢弃所有输入输出
        with open('/dev/null') as read_null, open('/dev/null', 'w') as write_null:
            os.dup2(read_null.fileno(), sys.stdin.fileno())
            os.dup2(write_null.fileno(), sys.stdout.fileno())
            os.dup2(write_null.fileno(), sys.stderr.fileno())

        # 写入pid文件
        if pid_file:
            with open(pid_file, 'w+') as f:
                f.write(str(os.getpid()))
            # 注册退出函数，进程异常退出时移除pid文件
            atexit.register(os.remove, pid_file)

__概括一下守护进程的编写步骤：__

1. fork出子进程，退出父进程
2. 子进程变更工作目录(chdir)、文件权限掩码(umask)、进程组和会话组(setsid)
3. 子进程fork孙子进程，退出子进程
4. 孙子进程刷新缓冲，重定向标准输入／输出／错误（一般到/dev/null，意即丢弃）
5. (可选)pid写入文件


### 理解几个要点

#### 为什么要fork两次
第一次fork，是为了脱离终端控制的魔爪。父进程之所以退出，是因为终端敲击键盘、或者关闭时给它发送了信号；而fork出来的子进程，在父进程自杀后成为孤儿进程，进而被操作系统的init进程接管，因此脱离终端控制。  
所以其实，第二次fork并不是必须的（很多开源项目里的代码就没有fork两次）。只不过出于谨慎考虑，防止进程再次打开一个控制终端。因为子进程现在是会话组长了（对话期的首次进程），有能力打开控制终端，再fork一次，孙子进程就不能打开控制终端了。  

#### 文件描述符
Linux是“一切皆文件”，文件描述符是内核为已打开的文件所创建的索引，通常是非负整数。进程通过文件描述符执行IO操作。  
每个进程有自己的文件描述符表，因此相同的描述符可能指向同一个文件，也可能指向不同文件；来自不同进程的不同的描述符，当然也有可能指向同一个文件。  
默认情况下，0代表标准输入，1代表标准输出，2代表标准错误。  

#### umask权限掩码
我们知道，在Linux中，任何一个文件都有读（read）、写（write）和执行（execute）的三种使用权限。其中，读的权限用数字4代表，写权限是2，执行权限是1。命令```ls -l```可以查看文件权限，r/w/x分别表示具有读/写/执行权限。  
任何文件，也都有用户（User）,用户组（Group）,其他组（Others）三种身份权限。一般用3个数字表示文件权限，例如754：  

- 7，是User权限，即文件拥有者权限
- 5，是Group权限，拥有者所在用户组的组员所具有的权限
- 4，是Others权限，即其他组用户的权限啦

而umask是为了控制默认权限，防止新建文件或文件夹具有全权。  
系统一般默认为022（使用命令umask查看），表示默认创建文件的权限是644，文件夹是755。你应该可以看出它们的规律，就是文件权限和umask的相加结果为666（笑），文件夹权限和umask的相加结果为777。

#### 进程组
每个进程都属于一个进程组（PG,Process Group），进程组可以包含多个进程。  
进程组有一个进程组长（Leader），进程组长的ID（PID, Process ID）就作为整个进程组的ID（PGID,Process Groupd ID）。

#### 会话组
登陆终端时，就会创造一个会话，多个进程组可以包含在一个会话中。而创建会话的进程，就是会话组长。  
已经是会话组长的进程，不可以再调用setsid()方法创建会话。因此，上面代码中，子进程可以调用setsid()，而父进程不能，因为它本身就是会话组长。  
另外，sh（Bourne Shell）不支持会话机制，因为会话机制需要shell支持工作控制（Job Control）。

#### 守护进程与后台进程
通过&符号，可以把命令放到后台执行。它与守护进程是不同的：

1. 守护进程与终端无关，是被init进程收养的孤儿进程；而后台进程的父进程是终端，仍然可以在终端打印
2. 守护进程在关闭终端时依然坚挺；而后台进程会随用户退出而停止，除非加上nohup
3. 守护进程改变了会话、进程组、工作目录和文件描述符，后台进程直接继承父进程（shell）的

换句话说：守护进程就是默默地奋斗打拼的有为青年，而后台进程是默默继承老爸资产的富二代。


