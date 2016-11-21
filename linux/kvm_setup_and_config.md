# kvm安装与配置

本文全部参照文档：https://help.ubuntu.com/community/KVM 进行。

-------------------

## 查看系统是否适合安装kvm

> egrep -c '(vmx|svm)' /proc/cpuinfo

如果显示1或者更多，证明cpu支持硬件虚拟化（hardware virtualization）。
如果显示为0，证明cpu不支持虚拟化，应该是不能进行kvm安装了，不排除有其它方法。

安装尽量使用64位内核系统

------------------------
安装kvm
--------------
> sudo apt-get install qemu-kvm libvirt-bin ubuntu-vm-builder bridge-utils

然后把用户加入到kvm需要的api组中
>sudo adduser `id -un` libvirtd

---------------------
验证安装
--------------
使用命令
>\$ virsh list --all
 Id Name                 State
\----------------------------------
\$

安装正常会显示上图

如果显示错误信息，请参照文档https://help.ubuntu.com/community/KVM/Installation

--------------------
网络设置
---------------
安装完毕后需要设置网络。
通常虚拟机与宿主机有两种网络连接方式[nat](https://help.ubuntu.com/community/KVM/Networking#usermodenetworking)和[bridge](https://help.ubuntu.com/community/KVM/Networking#bridgednetworking)

nat模式对于局域网中的其它机子无法访问，bridge模式对于局域网中其它机子可见

这里只介绍bridge连接方式

###bridge
先安装一个设置bridge网络的软件
>sudo apt-get install bridge-utils

然后编辑/etc/network/interfaces
>auto lo
iface lo inet loopback
auto br0
iface br0 inet static
        address 192.168.0.13
        network 192.168.0.0
        netmask 255.255.255.0
        broadcast 192.168.0.255
        gateway 192.168.0.1
        dns-nameservers 223.5.5.5
        bridge_ports eno0
        bridge_stp off
        bridge_fd 0
        bridge_maxwait 0

然后重启network。
发现重启无法ping通网关，需把原来绑定在eno1上的ip删除，与bridge ip 冲突了。


----------------
创建虚拟机
----------------
本人尝试了官网介绍的两种方法安装，都安装失败（感觉是自己设置有问题，用iso安装都应该是需要图形界面帮助的，我挂一晚上貌似一点动静都没有，还有一种用官方mirror的方法，最后也报错失败，所以最后在自己主机上安装个virt-manager来连接server上的libvirt来启动一个图形界面安装）
使用图形界面安装，需要在本机（本机为ubuntu系统，windows系统方法未知）和宿主机上安装
>sudo apt-get install virt-manager ssh-askpass-gnome --no-install-recommends

安装virt-manager会把一个界面插件也安装下来，这样，没有图形界面的宿主机才能顺利把界面传输到本机。

在本机上启动virt-manager。
![这里写图片描述](http://img.blog.csdn.net/20161118111725584)
打开后建立一个新连接，指定到连接的server地址就好了
![这里写图片描述](http://img.blog.csdn.net/20161118111917290)
然后创建一个新的虚拟机
![这里写图片描述](http://img.blog.csdn.net/20161118112020829)
按照步骤下去，然后启动一个窗口，按照安装系统方式下去就好了。

（经尝试，下面2种方法失败，用iso方法安装，需要图形界面介入而失败）
创建虚拟机有3种方法：
1. virt-install,     python 脚本安装
2. ubuntu-vm-builder


####ubuntu-vm-builder
>sudo ubuntu-vm-builder kvm trusty

以上命令就可以建立一个虚拟机了，设置都是按照默认设置来的。
>ubuntu-vm-builder kvm hardy \
                  --domain newvm \
                  --dest newvm \
                  --arch i386 \
                  --hostname hostnameformyvm \
                  --mem 256 \
                  --user john \
                  --pass doe \
                  --ip 192.168.0.12 \
                  --mask 255.255.255.0 \
                  --net 192.168.0.0 \
                  --bcast 192.168.0.255 \
                  --gw 192.168.0.1 \
                  --dns 192.168.0.1 \
                  --mirror http://archive.localubuntumirror.net/ubuntu \
                  --components main,universe \
                  --addpkg acpid \ 
                  --addpkg vim \
                  --addpkg openssh-server \
                  --addpkg avahi-daemon \
                  --libvirt qemu:///system ;
             
 以上设置更为详细，注意要在后面加上--libvirt qemu:///system;这么创建的虚拟机就可以用libvirt管理，这样就会再/etc/libvirt/qemu中创建一个xml文件，如果后面要更改虚拟机的内存cpu等设置可以直接修改这个xml文件。

####virt-install
virt-install可以直接用iso文件创建虚拟机

>sudo virt-install --connect qemu:///system -c ubuntu-16.04.1-server-amd64.iso --name openstack-test --memory 2048 --disk /home/useease/test.disk

创建时间较长，需要等待

