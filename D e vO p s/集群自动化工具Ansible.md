## 集群自动化工具Ansible


作者：[EaconTang](https://github.com/EaconTang) | 文章出处：[博客链接](http://blog.tangyingkang.com/post/2016/12/20/use-ansible/)  

----


![](http://qn.tangyingkang.com/image/blog/ansible/ansible-logo.png)

### 安装
    $ pip install ansible

### 配置
Ansible配置文件的读取顺序为：

    1) ANSIBLE_CONFIG (环境变量)
    2) ansible.cfg (当前目录)
    3) ~/.ansible.cfg (用户目录)
    4) /etc/ansible/ansible.cfg

将通信主机的地址（hostname或者ip）写入文件：

    $ echo "192.168.78.131" >> /etc/ansible/hosts

测试地址可以使用ssh账号密码连接，也可以用公钥认证。  
如果使用SSH公钥授权，为了避免在建立SSH连接时重复输入密码，可以执行:

    $ ssh-agent bash
    $ ssh-add ~/.ssh/id_rsa

如果有个主机重新安装并在“known_hosts”中有了不同的key，这会提示一个错误信息直到被纠正为止.在使用Ansible时，你可能不想遇到这样的情况:如果有个主机没有在“known_hosts”中被初始化将会导致在交互使用Ansible或定时执行Ansible时对key信息的确认提示。  
添加配置项host_key_checking可以解决这个问题：  

    $ vi ~/.ansible.cfg

    [defaults]
    host_key_checking = False

### 运行任务
Ansible提供两种方式去完成任务，一是 ad-hoc 命令，一是写 Ansible playbook。前者可以解决一些简单的任务， 后者解决较复杂的任务。  

### ad-hoc临时任务
如果我们敲入一些命令去比较快的完成一些事情，而不需要将这些执行的命令特别保存下来，这样的命令就叫做ad-hoc命令。  

测试ping通远程节点：

    $ ansible all -m ping    
    192.168.78.131 | SUCCESS => {
    "changed": false,
    "ping": "pong"
    }

查看远程节点的motd欢迎信息:

    $ ansible 192.168.78.131 -m command -a "cat /etc/motd"
    192.168.78.131 | SUCCESS | rc=0 >>
    Hello

#### 关于ansbile工具的shell、command、script、raw模块的区别和使用场景
__command__：执行远程命令

    $ ansible 192.168.78.131 -m command -a 'uname -n'

__script__：在远程主机执行主控端的shell/python脚本（支持相对路径）

    $ ansible 192.168.78.131 -m script -a './test.sh'

__shell__：执行远程主机的shell/python脚本

    $ ansible 192.168.78.131 -m shell -a 'bash /root/test.sh'

__raw__：类似于command模块、支持管道传递

    $ ansible 192.168.78.131 -m raw -a "cat /etc/hosts |grep 127.0.0.1"


### Ansible的强大之处----playbooks
playbooks是配置管理系统与多机器部署系统的基础，以yaml配置文件的形式保存。  
在 playbooks 中可以编排有序的执行过程，甚至于做到在多组机器间，来回有序的执行特别指定的步骤，并且可以同步或异步的发起任务。在我看来，它的功能性和复杂度相当于一个脚本语言了。  

举个最简单的栗子，编写如下文件：  

    - hosts: 192.168.78.131
      remote_user: root
      tasks:
        - name: test connection
          ping : 

保存为playbook.yml，执行：

    $ ansible-playbook playbook.yml

    PLAY [192.168.78.131] ******************************************************************

    TASK [setup] *******************************************************************
    ok: [192.168.78.131]

    TASK [test connection] *********************************************************
    ok: [192.168.78.131]

    PLAY RECAP *********************************************************************
    192.168.78.131                     : ok=2    changed=0    unreachable=0    failed=0



#### 异步操作
为了异步启动一个任务,可以指定其最大超时时间以及轮询其状态的频率.如果你没有为 poll 指定值,那么默认的轮询频率是10秒钟:  
例如，编写如下playbook任务，保存为文件asyn.yml：

    - hosts: all
      remote_user: root      
      tasks:
        - name: 执行sleep15秒的耗时任务，每隔5秒轮询，最大等待时间45秒
          command: /bin/sleep 15
          async: 45
          poll: 5

执行任务：

    $ ansible-playbook asyn.yml

    PLAY [all] *********************************************************************

    TASK [setup] *******************************************************************
    ok: [as7-01]
    ok: [as7-04]
    ok: [as7-03]

    TASK [执行sleep15秒的耗时任务，每隔5秒轮询，最大等待时间45秒] ****************************************
    changed: [as7-01]
    changed: [as7-03]
    changed: [as7-04]

    PLAY RECAP *********************************************************************
    as7-01                     : ok=2    changed=1    unreachable=0    failed=0
    as7-03                     : ok=2    changed=1    unreachable=0    failed=0
    as7-04                     : ok=2    changed=1    unreachable=0    failed=0








