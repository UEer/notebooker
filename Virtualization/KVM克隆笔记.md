# 如何在linux中进行镜像虚拟机静态迁移（KVM克隆笔记）
---------------
*author*:[shikanon](https://github.com/shikanon)

##  操作
####    安装virsh、qemu
>   sudo apt-get install qemu-kvm libvirt

####    拷贝image文件
先查看下要img的格式：
`qemu-img info devstack-controller-clone.img `
>image: devstack-controller.img
file format: raw
virtual size: 120G (128849125376 bytes)
disk size: 120G
[root@ue211 images]# qemu-img info devstack-controller.img 
image: devstack-controller.img
file format: raw
virtual size: 120G (128849018880 bytes)
disk size: 120G


说明这是一个raw格式的image，
image文件通常是raw，相对较大，不适合传输，所以先把raw转为qcow2格式:
`qemu-img convert -c -f raw -O qcow2 devstack-controller.img devstack-controller-clone2.img`


看看转换后的格式：
`qemu-img info devstack-controller-clon2e.img `
>image: devstack-controller-clone2.img
file format: qcow2
virtual size: 120G (128849018880 bytes)
disk size: 3.5G
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false

用scp传到目标地址上：
`scp /var/lib/libvirt/images/devstack-controller-clone2.img root@192.168.0.12:/var/lib/libvirt/images/devstack-controller-clone2.img`

####    修改和拷贝配置文件
导出配置文件：
`virsh dumpxml devstack-controller > /home/devstack-controller.xml`

或者直接找到路径拷贝也可以，centos下路径在：``

修改配置文件，主要需要修改的地方有name、uuid、mac:
`<name>devstack-controller</name>`
`<uuid>4aba494c-6f6a-f992-2ec2-23795f6c4680</uuid>`
`<mac address='52:54:00:49:03:d2'/>`
name表示虚拟机的名字 
uuid表示id号，可以用`uuid`命令生成
mac表示网关mac地址

如果是迁移到其他系统，其他机器上，还需要修改emulator、source file:
`<emulator>/usr/bin/kvm-spice</emulator>`
`<source file='/var/lib/libvirt/images/devstack-controller-clone2.img'/>`
emulator表示kvm路径
source file 表示image路径

用scp传到目标地址上：
`scp /home/devstack-controller.xml root@192.168.0.12:/home/devstack-controller.xml`

将配置文件和image都传到目标机后，将qcow2转换为raw：
`qemu-img convert -f qcow2 -O raw devstack-controller-clone2.img devstack-controller.img`

####    启动虚拟机
用virsh启动新的虚拟机
`virsh define devstack-controller.xml`

## 命令解说
`qemu-img convert [-c] [-e] [-f format] filename [-O output_format] output_filename`
qemu-img convert主要用来转换镜像格式，`-c`表示压缩，只有qcow和qcow2才有压缩，`-f`表示输入的格式,`-O`表示输出的格式

