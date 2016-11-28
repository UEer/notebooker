###### OpenStack通过各种补充服务提供基础设施即服务 Infrastructure-as-a-Service (IaaS)。

openstack 主要服务

服务 | 名称
---|---
dash | horizon
compute | nova
networking | neutron
object storge | swift
block storge | cinder
identity service | keystone
image service | glance
telemetry | ceilometer
orchestration | heat

## 示例架构
*示例架构要求至少两个节点。块存储和对象存储节点可选。*
- 网络代理在控制节点
- 私有网络的叠加(隧道)流量通过管理网络而不是专用网络。
![image](http://docs.openstack.org/liberty/zh_CN/install-guide-rdo/_images/hwreqs.png)

#### 控制节点
运行身份认证服务(keystone),镜像服务(glance), 管理部分网络(neutron),计算(nova)服务，==不同的网络代理==和dashborad(horizon)。
额外的 sql数据库(maridb), 消息队列(rabbitmq), 时间同步服务(chrony)
至少两块网卡
#### 计算节点
运行操作实例的 `hypervisor`计算部分，默认情况下使用`KVM <kernel-based VM (KVM)>`作为hypervisor。同样运行网络代理服务(neutron)，通过`security groups <security group>`为实例提供防火墙服务。
可以部署多少个计算节点，至少两个网卡

#### 块设备存储
向实例提供磁盘
计算节点和这个节点间的服务流量使用==管理网络==。生产环境中应该实施单独的==存储网络==以增强性能和安全

#### 对象存储
==对象存储服务用来存储账号，容器和对象==

### 网络
1. 提供者网络`bridge`
提供者网络选项以最简单的方式部署OpenStack网络服务，可能包括二层服务(桥/交换机)服务、VLAN网络分段。本质上，它建立虚拟网络到物理网络的桥，依靠物理网络基础设施提供三层服务(路由)。
桥接
2. 自服务网络
它使用 :`NAT`路由虚拟网络到路由物理网络

### 时间同步服务
    yum install chrony
编辑/etc/chrony.conf
> server controller iburst

    systemctl enable chronyd.service
    systemctl start chronyd.service
验证`chronyc sources`

### 网络配置
控制节点 eth0:192.168.0.161 eth1：不用配置


计算节点 eth1:192.168.0.163 eht1: 不用配置

### 所有节点安装 openstack 包
    yum install centos-release-openstack-liberty
    yum install https://rdoproject.org/repos/openstack-liberty/rdo-release-liberty.rpm
    yum upgrade
    yum install python-openstackclient
    yum install openstack-selinux
    
### SQL 数据库(控制节点)
    yum install mariadb mariadb-server MySQL-python

编辑 /etc/mysql/conf.d/mariadb_openstack.cnf

```
sed -i "/\[mysqld\]$/a character-set-server = utf8" /etc/my.cnf
sed -i "/\[mysqld\]$/a init-connect = 'SET NAMES utf8'" /etc/my.cnf
sed -i "/\[mysqld\]$/a collation-server = utf8_general_ci" /etc/my.cnf
sed -i "/\[mysqld\]$/a innodb_file_per_table" /etc/my.cnf
sed -i "/\[mysqld\]$/a default-storage-engine = innodb" /etc/my.cnf
sed -i "/\[mysqld\]$/a bind-address = 192.168.0.161" /etc/my.cnf
```

    systemctl enable mariadb.service
    systemctl start mariadb.service
    mysql_secure_installation

### NoSQL数据库(控制节点) Telemetry 使用
    yum install mongodb-server mongodb
编辑 /etc/mongod.conf
> bind_ip = 192.168.0.161

> smallfiles = true

    systemctl enable mongod.service
    systemctl start mongod.service
    
### 消息队列 协调操作和各服务的状态信息
    yum install rabbitmq-server
    systemctl enable rabbitmq-server.service
    systemctl start rabbitmq-server.service
    rabbitmqctl add_user openstack RABBIT_PASS
    rabbitmqctl set_permissions openstack ".*" ".*" ".*"
    
**安装openstack 配置工具**

    yum install -y python-openstackclient openstack-utils
    
### 安装身份认证服务(keystone)
配置部署Apache HTTP服务处理查询

Memcached存储tokens

```
mysql -u root -p
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
  IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
  IDENTIFIED BY '123456';
```
    yum install openstack-keystone httpd mod_wsgi memcached python-memcached
    systemctl enable memcached.service
    systemctl start memcached.service

    生成`ADMIN_TOKEN`
    openssl rand -hex 10

```
openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token ADMIN_TOKEN
openstack-config --set /etc/keystone/keystone.conf database connection mysql://keystone:pass@controller/keystone
openstack-config --set /etc/keystone/keystone.conf memcache servers localhost:11211
openstack-config --set /etc/keystone/keystone.conf token provider uuid
openstack-config --set /etc/keystone/keystone.conf token driver memcache
openstack-config --set /etc/keystone/keystone.conf revoke driver sql
```
     su -s /bin/sh -c "keystone-manage db_sync" keystone
#### 安装apache
- 编辑 `/etc/httpd/conf/httpd.conf`
    sed -i "s/#ServerName www.example.com:80/ServerName controller/" /etc/httpd/conf/httpd.conf

- 创建apache启动的配置文件

```
cat > /etc/httpd/conf.d/wsgi-keystone.conf << OFF
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>
    ErrorLog /var/log/httpd/keystone-error.log
    CustomLog /var/log/httpd/keystone-access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>
OFF
```

#### 启动服务

```
systemctl enable httpd.service
systemctl start httpd.service
```

### 创建服务实体(identity)和API端点
keystone 提供服务的 目录和位置
每个添加到openstack 的服务 需要一个server实体和api endpoints

    export OS_TOKEN=ADMIN_TOKEN
    export OS_URL=http://controller:35357/v3
    export OS_IDENTITY_API_VERSION=3
    

创建实体(openstack service create)和端点(openstack endpoint create)
```
openstack service create \
  --name keystone --description "OpenStack Identity" identity
openstack endpoint create --region RegionOne \
  identity public http://controller:5000/v2.0
openstack endpoint create --region RegionOne \
  identity internal http://controller:5000/v2.0
openstack endpoint create --region RegionOne \
  identity admin http://controller:35357/v2.0
```
> 每个添加到OpenStack环境中的服务要求一个或多个服务实体和三个认证服务中的API 端点变种(public, internal, admin 为三种网络，本安装使用同一个网络)。

#### 创建项目、用户和角色
身份认证服务为每个OpenStack服务提供认证服务

认证服务使用 project， user， role 组合
1. 在你的环境中，为进行管理操作，创建管理的项目(admin, service)、用户(admin)和 角色(admin)：
admin
```
openstack project create --domain default \
  --description "Admin Project" admin
openstack user create --domain default \
  --password-prompt admin
openstack role create admin
openstack role add --project admin --user admin admin
```
单独service 项目
```
 openstack project create --domain default \
  --description "Service Project" service
```

demmo
```
openstack project create --domain default \
  --description "Demo Project" demo
openstack user create --domain default \
  --password-prompt demo
openstack role create user
openstack role add --project demo --user demo user
```

#### 验证操作
1. 因为安全性的原因，关闭临时认证令牌机制
编辑``/usr/share/keystone/keystone-dist-paste.ini``文件
从``[pipeline:public_api]``,``[pipeline:admin_api]``和``[pipeline:api_v3]``部分删除``admin_token_auth`` 。
2. 重置``OS_TOKEN``和``OS_URL`` 环境变量：
    unset OS_TOKEN OS_URL
3. 使用 admin 用户，请求认证令牌：

```
openstack --os-auth-url http://controller:35357/v3 \
  --os-project-domain-id default --os-user-domain-id default \
  --os-project-name admin --os-username admin --os-auth-type password \
  token issue
```
#### 创建 OpenStack 客户端环境脚本
admin-openrc.sh
```
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=admin
export OS_TENANT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=123456
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
```
demon-openrc.sh
```
export OS_PROJECT_DOMAIN_ID=default
export OS_USER_DOMAIN_ID=default
export OS_PROJECT_NAME=demo
export OS_TENANT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=123456
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```
## 添加镜像服务
OpenStack 的镜像服务 (glance) 允许用户发现、注册和恢复虚拟机镜像。
1. 创建数据库
```
mysql -u root -p
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  ID
```
2. 创建服务实体service image

```
openstack service create --name glance \
  --description "OpenStack Image service" image
```

3. 创建api endppints

```
openstack user create glance --domain default --password pass
openstack role add --project service --user glance admin
```
```
openstack endpoint create --region RegionOne \
  image public http://controller:9292
openstack endpoint create --region RegionOne \
  image internal http://controller:9292
openstack endpoint create --region RegionOne \
  image admin http://controller:9292
```
4 安装

```
yum install openstack-glance python-glance python-glanceclient
```
修改 /etc/glance/glance-api.conf
```
openstack-config --set /etc/glance/glance-api.conf database  connection mysql://glance:pass@controller/glance
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken  auth_uri http://controller:5000
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken  auth_url http://controller:35357
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken  auth_plugin  password
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken  project_domain_id  default
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken  user_domain_id default
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken  project_name service
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken  username glance
openstack-config --set /etc/glance/glance-api.conf keystone_authtoken  password  pass
openstack-config --set /etc/glance/glance-api.conf paste_deploy flavor keystone
openstack-config --set /etc/glance/glance-api.conf glance_store default_store file
openstack-config --set /etc/glance/glance-api.conf glance_store filesystem_store_datadir /var/lib/glance/images/
openstack-config --set /etc/glance/glance-api.conf DEFAULT notification_driver noop
openstack-config --set /etc/glance/glance-api.conf DEFAULT verbose True
```
修改 /etc/glance/glance-registry.conf

```
openstack-config --set /etc/glance/glance-registry.conf database connection mysql://glance:pass@controller/glance
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken  auth_uri http://controller:5000
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken  auth_url http://controller:35357
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken  auth_plugin  password
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken  project_domain_id  default
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken  user_domain_id default
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken  project_name service
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken  username glance
openstack-config --set /etc/glance/glance-registry.conf keystone_authtoken  password  pass
openstack-config --set /etc/glance/glance-registry.conf paste_deploy flavor keystone
openstack-config --set /etc/glance/glance-registry.conf DEFAULT notification_driver noop
openstack-config --set /etc/glance/glance-registry.conf DEFAULT verbose True
```


    su -s /bin/sh -c "glance-manage db_sync" glance
    systemctl enable openstack-glance-api.service openstack-glance-registry.service
    systemctl start openstack-glance-api.service openstack-glance-registry.service
    
5. 验证

```
echo "export OS_IMAGE_API_VERSION=2" \
  | tee -a admin-openrc.sh demo-openrc.sh
source admin-openrc.sh
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
glance image-create --name "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --visibility public --progress
glance image-list
```
## 安装计算服务nova
1. 创建数据库
mysql -u root -p
```
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'   IDENTIFIED BY 'pass';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%'   IDENTIFIED BY 'pass';
```
2. 添加用户 服务和Endpoint
project service, user ova

```
openstack user create nova --domain default --password pass
openstack role add --project service --user nova admin
openstack service create --name nova   --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne  compute public http://controller:8774/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne  compute internal http://controller:8774/v2/%\(tenant_id\)s
openstack endpoint create --region RegionOne  compute admin http://controller:8774/v2/%\(tenant_id\)s
```
3. 组件安装

```
yum install openstack-nova-api openstack-nova-cert \
  openstack-nova-conductor openstack-nova-console \
  openstack-nova-novncproxy openstack-nova-scheduler \
  python-novaclient
```

4. 配置

```
openstack-config --set /etc/nova/nova.conf database connection mysql://nova:pass@controller/nova
openstack-config --set /etc/nova/nova.conf DEFAULT rpc_backend rabbit
openstack-config --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_host controller
openstack-config --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_userid openstack
openstack-config --set /etc/nova/nova.conf oslo_messaging_rabbit rabbit_password pass
openstack-config --set /etc/nova/nova.conf DEFAULT auth_strategy keystone
openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_uri http://controller:5000
openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_url http://controller:35357
openstack-config --set /etc/nova/nova.conf keystone_authtoken auth_plugin password
openstack-config --set /etc/nova/nova.conf keystone_authtoken project_domain_id default
openstack-config --set /etc/nova/nova.conf keystone_authtoken user_domain_id default
openstack-config --set /etc/nova/nova.conf keystone_authtoken project_name service
openstack-config --set /etc/nova/nova.conf keystone_authtoken username nova
openstack-config --set /etc/nova/nova.conf keystone_authtoken password pass
openstack-config --set /etc/nova/nova.conf DEFAULT my_ip 192.168.10.102
openstack-config --set /etc/nova/nova.conf DEFAULT network_api_class nova.network.neutronv2.api.API
openstack-config --set /etc/nova/nova.conf DEFAULT security_group_api neutron
openstack-config --set /etc/nova/nova.conf DEFAULT linuxnet_interface_driver nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver
openstack-config --set /etc/nova/nova.conf DEFAULT firewall_driver nova.virt.firewall.NoopFirewallDriver
openstack-config --set /etc/nova/nova.conf vnc vncserver_listen 192.168.10.102
openstack-config --set /etc/nova/nova.conf vnc vncserver_proxyclient_address 192.168.10.102
openstack-config --set /etc/nova/nova.conf glance host controller
openstack-config --set /etc/nova/nova.conf oslo_concurrency lock_path /var/lib/nova/tmp
openstack-config --set /etc/nova/nova.conf DEFAULT enabled_apis osapi_compute,metadata
openstack-config --set /etc/nova/nova.conf DEFAULT verbose True 
```
5. 初始化数据库
    su -s /bin/sh -c "nova-manage db sync" nova
6. 启动服务

```
systemctl enable openstack-nova-api.service \
openstack-nova-cert.service openstack-nova-consoleauth.service \
openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service

systemctl start openstack-nova-api.service \
openstack-nova-cert.service openstack-nova-consoleauth.service \
openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service
```

#### 计算节点安装 nova
1. 安装

```
 yum install openstack-nova-compute sysfsutils
```
2. 配置 /etc/nova/nova.conf

```
[DEFAULT]
...
rpc_backend = rabbit

[oslo_messaging_rabbit]
...
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
[DEFAULT]
...
auth_strategy = keystone

[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = nova
password = NOVA_PASS
[DEFAULT]
...
my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
[DEFAULT]
...
network_api_class = nova.network.neutronv2.api.API
security_group_api = neutron
linuxnet_interface_driver = nova.network.linux_net.NeutronLinuxBridgeInterfaceDriver
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[vnc]
...
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
[glance]
...
host = controller
[oslo_concurrency]
...
lock_path = /var/lib/nova/tmp
[DEFAULT]
...
verbose = True
```
3. 计算节点是否支持加速
    $ egrep -c '(vmx|svm)' /proc/cpuinfo

```
[libvirt]
...
virt_type = qemu
```
4. 启动服务

```
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```

5. 验证
    nova service-list
    nova endpoints
    nova image-list

## 添加网络服务 neutron
1. 创建数据库neutron
2. 创建用户
3. 用户添加到项目service， 角色admin
4. 创建service network和api endpoints
5. 配置元数据代理`/etc/neutron/metadata_agent.ini`
6. 配置nova使用网络` /etc/nova/nova.conf`   
7. ` ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini`
8. su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
9. systemctl restart openstack-nova-api.service
10. 

```
systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
```
对网络选项2，同样也启用并启动layer-3服务：

```
systemctl enable neutron-l3-agent.service
systemctl start neutron-l3-agent.service
```
#### 提供者网络
1. 编辑 `/etc/neutron/neutron.conf`

```
openstack-config --set /etc/neutron/neutron.conf database connection mysql://neutron:pass@controller/neutron
openstack-config --set /etc/neutron/neutron.conf DEFAULT core_plugin ml2
openstack-config --set /etc/neutron/neutron.conf DEFAULT service_plugins
openstack-config --set /etc/neutron/neutron.conf DEFAULT rpc_backend rabbit
openstack-config --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_host controller
openstack-config --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_userid openstack
openstack-config --set /etc/neutron/neutron.conf oslo_messaging_rabbit rabbit_password pass
openstack-config --set /etc/neutron/neutron.conf DEFAULT auth_strategy keystone
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_uri http://controller:5000
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_url http://controller:35357
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken auth_plugin password
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken project_domain_id default
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken user_domain_id default
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken project_name service
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken username neutron
openstack-config --set /etc/neutron/neutron.conf keystone_authtoken password pass
openstack-config --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_status_changes True
openstack-config --set /etc/neutron/neutron.conf DEFAULT notify_nova_on_port_data_changes True
openstack-config --set /etc/neutron/neutron.conf DEFAULT nova_url http://controller:8774/v2
openstack-config --set /etc/neutron/neutron.conf nova auth_url http://controller:35357
openstack-config --set /etc/neutron/neutron.conf nova auth_plugin password
openstack-config --set /etc/neutron/neutron.conf nova project_domain_id default
openstack-config --set /etc/neutron/neutron.conf nova user_domain_id default
openstack-config --set /etc/neutron/neutron.conf nova region_name RegionOne
openstack-config --set /etc/neutron/neutron.conf nova project_name service
openstack-config --set /etc/neutron/neutron.conf nova username nova
openstack-config --set /etc/neutron/neutron.conf nova password pass
openstack-config --set /etc/neutron/neutron.conf oslo_concurrency lock_path /var/lib/neutron/tmp
openstack-config --set /etc/neutron/neutron.conf DEFAULT verbose True 
```
2. 配置 Modular Layer 2 (ML2) 插件

*ML2插件使用Linux桥接机制为实例创建layer-2 （桥接/交换）虚拟网络基础设施。*

```
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 type_drivers flat
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers linuxbridge
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers port_security
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2_type_flat flat_networks public
openstack-config --set /etc/neutron/plugins/ml2/ml2_conf.ini securitygroup enable_ipset  True
```

3. linux bridge

```
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini linux_bridge physical_interface_mappings public:enp8s0
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini vxlan enable_vxlan  False
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini agent prevent_arp_spoofing True
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup enable_security_group True
openstack-config --set /etc/neutron/plugins/ml2/linuxbridge_agent.ini securitygroup firewall_driver neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

4. dhcp agent

```
openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.BridgeInterfaceDriver
openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT dhcp_driver neutron.agent.linux.dhcp.Dnsmasq
openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT enable_isolated_metadata True
openstack-config --set /etc/neutron/dhcp_agent.ini DEFAULT verbose True
```

5. metadata agent

```
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_uri http://controller:5000
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_url http://controller:35357  
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_region RegionOne  
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT auth_plugin password  
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT project_domain_id  default
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT user_domain_id default
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT project_name  service 
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT username  neutron
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT password  pass
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT nova_metadata_ip  controller 
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT metadata_proxy_shared_secret neutron 
openstack-config --set /etc/neutron/metadata_agent.ini DEFAULT verbose  True
```

#### 网络选项2：自服务网络

### 计算节点安装 neutron
1. 安装
yum install openstack-neutron openstack-neutron-linuxbridge ebtables ipset
2. 配置 `/etc/neutron/neutron.conf`
3. 配置 Nova使用 Neutron
4. 重启服务
    systemctl restart openstack-nova-compute.service
5. 启动Linux桥接代理并配置它开机自启动
    systemctl enable neutron-linuxbridge-agent.service
    systemctl start neutron-linuxbridge-agent.service
6. 提供者网络 编辑 `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`
7. 自服务网络 编辑 `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`


## 控制节点安装cinder
1. 创建数据库cinder
2. 创建用户cinder,添加到project(service), role(admin)
3. 创建服务实体 cinder, cinder2
4. 创建api endpoints
5. 安装cinder
6. 配置cinder `/etc/cinder/cinder.conf`
7. 配置nova 使用 cinder
8. 验证`neutron ext-list` `neutron agent-list`

### 配置一个block storge
1. 创建pv
2. 创建vg
3. 配置 `/etc/cinder/cinder.conf`
4. 
```
systemctl enable openstack-cinder-volume.service target.service
systemctl start openstack-cinder-volume.service target.service
```
5. 验证 `cinder service-list`


## 控制节点安装 swift
1. 创建数据库 swift
2. 创建用户 switf, 添加到project， role
3. 创建服务实体swift,
4. 创建api endpoints
5. 安装软件包

```
yum install openstack-swift-proxy python-swiftclient \
  python-keystoneclient python-keystonemiddleware \
  memcached
```
6. 下载配置文件
    curl -o /etc/swift/proxy-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/proxy-server.conf-sample?h=stable/liberty
7. 编辑 /etc/swift/proxy-server.conf

```
openstack-config --set /etc/swift/proxy-server.conf DEFAULT  bind_port 8080
openstack-config --set /etc/swift/proxy-server.conf DEFAULT  user swift
openstack-config --set /etc/swift/proxy-server.conf DEFAULT swift_dir /etc/swift
openstack-config --set /etc/swift/proxy-server.conf pipeline:main pipeline " catch_errors gatekeeper healthcheck proxy-logging cache
container_sync bulk ratelimit authtoken keystoneauth container-quotas
account-quotas slo dlo versioned_writes proxy-logging proxy-server"
openstack-config --set /etc/swift/proxy-server.conf app:proxy-server use egg:swift#proxy
openstack-config --set /etc/swift/proxy-server.conf app:proxy-server account_autocreate true
openstack-config --set /etc/swift/proxy-server.conf filter:keystoneauth use = egg:swift#keystoneauth
openstack-config --set /etc/swift/proxy-server.conf filter:keystoneauth  operator_roles "admin,user"
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken paste.filter_factory keystonemiddleware.auth_token:filter_factory
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken auth_uri http://controller:5000
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken auth_url http://controller:35357
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken auth_plugin password
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken project_domain_id default
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken user_domain_id default
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken project_name service
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken username swift
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken password SWIFT_PASS
openstack-config --set /etc/swift/proxy-server.conf filter:authtoken delay_auth_decision true
openstack-config --set /etc/swift/proxy-server.conf filter:cache use egg:swift#memcache
openstack-config --set /etc/swift/proxy-server.conf filter:cache memcache_servers 127.0.0.1:11211

```

#### 安装配置存储节点
    systemctl start rsyncd.service
安装包
    yum install openstack-swift-account openstack-swift-container \
  openstack-swift-object

```
curl -o /etc/swift/account-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/account-server.conf-sample?h=stable/liberty
curl -o /etc/swift/container-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/container-server.conf-sample?h=stable/liberty
curl -o /etc/swift/object-server.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/object-server.conf-sample?h=stable/liberty
```

```
#  /etc/swift/account-server.conf 
openstack-config --set /etc/swift/account-server.conf DEFAULT bind_ip 0.0.0.0
openstack-config --set /etc/swift/account-server.conf DEFAULT bind_port 6002
openstack-config --set /etc/swift/account-server.conf DEFAULT user swift
openstack-config --set /etc/swift/account-server.conf DEFAULT swift_dir /etc/swift
openstack-config --set /etc/swift/account-server.conf DEFAULT devices /srv/node
openstack-config --set /etc/swift/account-server.conf DEFAULT mount_check true
openstack-config --set /etc/swift/account-server.conf pipeline:main pipeline healthcheck recon account-server
openstack-config --set /etc/swift/account-server.conf filter:recon use egg:swift#recon
openstack-config --set /etc/swift/account-server.conf filter:recon recon_cache_path /var/cache/swift
# /etc/swift/container-server.conf
openstack-config --set /etc/swift/container-server.conf DEFAULT bind_ip 0.0.0.0
openstack-config --set /etc/swift/container-server.conf DEFAULT bind_port 6001
openstack-config --set /etc/swift/container-server.conf DEFAULT user swift
openstack-config --set /etc/swift/container-server.conf DEFAULT swift_dir /etc/swift
openstack-config --set /etc/swift/container-server.conf DEFAULT devices /srv/node
openstack-config --set /etc/swift/container-server.conf DEFAULT mount_check true
openstack-config --set /etc/swift/container-server.conf pipeline:main pipeline healthcheck recon container-server
openstack-config --set /etc/swift/container-server.conf filter:recon use egg:swift#recon
openstack-config --set /etc/swift/container-server.conf filter:recon recon_cache_path /var/cache/swift
# /etc/swift/container-server.conf


openstack-config --set /etc/swift/object-server.conf DEFAULT bind_ip  0.0.0.0
openstack-config --set /etc/swift/object-server.conf DEFAULT bind_port 6000
openstack-config --set /etc/swift/object-server.conf DEFAULT user swift
openstack-config --set /etc/swift/object-server.conf DEFAULT swift_dir /etc/swift
openstack-config --set /etc/swift/object-server.conf DEFAULT devices /srv/node
openstack-config --set /etc/swift/object-server.conf DEFAULT mount_check true
openstack-config --set /etc/swift/object-server.conf pipeline:main pipeline = healthcheck recon object-server
openstack-config --set /etc/swift/object-server.conf filter:recon use = egg:swift#recon
openstack-config --set /etc/swift/object-server.conf filter:recon recon_cache_path = /var/cache/swift
openstack-config --set /etc/swift/object-server.conf filter:recon recon_lock_path = /var/lock
```
    chown -R swift:swift /srv/node
    mkdir -p /var/cache/swift
    chown -R root:swift /var/cache/swift

## 创建和分发初始化rings
cd /etc/swift
###### 创建account.builder
    swift-ring-builder account.builder create 10 3 1

```
swift-ring-builder account.builder add \
  --region 1 --zone 1 --ip 192.168.0.172 --port 6002 --device vdb --weight 100
Device d0r1z1-10.0.0.51:6002R10.0.0.51:6002/sdb_"" with 100.0 weight got id 0
# swift-ring-builder account.builder add \
  --region 1 --zone 2 --ip 192.168.0.172 --port 6002 --device vdc --weight 100
Device d1r1z2-10.0.0.51:6002R10.0.0.51:6002/sdc_"" with 100.0 weight got id 1
# swift-ring-builder account.builder add \
  --region 1 --zone 3 --ip 192.168.0.183 --port 6002 --device vdb --weight 100
Device d2r1z3-10.0.0.52:6002R10.0.0.52:6002/sdb_"" with 100.0 weight got id 2
# swift-ring-builder account.builder add \
  --region 1 --zone 4 --ip 192.168.0.183 --port 6002 --device vdc --weight 100
Device d3r1z4-10.0.0.52:6002R10.0.0.52:6002/sdc_"" with 100.0 weight got id 3
```
    swift-ring-builder account.builder
    swift-ring-builder account.builder rebalance
    
###### 创建容器ring container.builer
```
swift-ring-builder container.builder create 10 3 1
swift-ring-builder container.builder add \
  --region 1 --zone 1 --ip 192.168.0.172 --port 6002 --device vdb --weight 100
Device d0r1z1-10.0.0.51:6002R10.0.0.51:6002/sdb_"" with 100.0 weight got id 0
# swift-ring-builder container.builder add \
  --region 1 --zone 2 --ip 192.168.0.172 --port 6002 --device vdc --weight 100
Device d1r1z2-10.0.0.51:6002R10.0.0.51:6002/sdc_"" with 100.0 weight got id 1
# swift-ring-builder container.builder add \
  --region 1 --zone 3 --ip 192.168.0.183 --port 6002 --device vdb --weight 100
Device d2r1z3-10.0.0.52:6002R10.0.0.52:6002/sdb_"" with 100.0 weight got id 2
# swift-ring-builder container.builder add \
  --region 1 --zone 4 --ip 192.168.0.183 --port 6002 --device vdc --weight 100
Device d3r1z4-10.0.0.52:6002R10.0.0.52:6002/sdc_"" with 100.0 weight got id 3
```
    swift-ring-builder container.builder
    swift-ring-builder container.builder rebalance
###### 创建对象ring object.builder
```
swift-ring-builder object.builder create 10 3 1
swift-ring-builder object.builder add \
  --region 1 --zone 1 --ip 192.168.0.172 --port 6002 --device vdb --weight 100
Device d0r1z1-10.0.0.51:6002R10.0.0.51:6002/sdb_"" with 100.0 weight got id 0
# swift-ring-builder object.builder add \
  --region 1 --zone 2 --ip 192.168.0.172 --port 6002 --device vdc --weight 100
Device d1r1z2-10.0.0.51:6002R10.0.0.51:6002/sdc_"" with 100.0 weight got id 1
# swift-ring-builder object.builder add \
  --region 1 --zone 3 --ip 192.168.0.183 --port 6002 --device vdb --weight 100
Device d2r1z3-10.0.0.52:6002R10.0.0.52:6002/sdb_"" with 100.0 weight got id 2
# swift-ring-builder object.builder add \
  --region 1 --zone 4 --ip 192.168.0.183 --port 6002 --device vdc --weight 100
Device d3r1z4-10.0.0.52:6002R10.0.0.52:6002/sdc_"" with 100.0 weight got id 3
```
    swift-ring-builder object.builder
    swift-ring-builder object.builder rebalance

复制``account.ring.gz``，container.ring.gz``和``object.ring.gz 文件到每个存储节点和其他运行了代理服务的额外节点的 /etc/swift 目录。


###### 获取配置文件 curl -o /etc/swift/swift.conf https://git.openstack.org/cgit/openstack/swift/plain/etc/swift.conf-sample?h=stable/liberty

```
openstack-config --set /etc/swift/swift.conf swift-hash swift_hash_path_suffix  useease
openstack-config --set /etc/swift/swift.conf swift-hash swift_hash_path_prefix  useease1
openstack-config --set /etc/swift/swift.conf swift-hash storage-policy:0 name Policy-0
openstack-config --set /etc/swift/swift.conf swift-hash storage-policy:0 default yes
```
    chown -R root:swift /etc/swift

```
systemctl enable openstack-swift-proxy.service memcached.service
systemctl start openstack-swift-proxy.service memcached.service
```


```
systemctl enable openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service
systemctl enable openstack-swift-container.service \
  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
  openstack-swift-container-updater.service
systemctl enable openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service

  
systemctl start openstack-swift-container.service \
  openstack-swift-container-auditor.service openstack-swift-container-replicator.service \
  openstack-swift-container-updater.service
systemctl start openstack-swift-account.service openstack-swift-account-auditor.service \
  openstack-swift-account-reaper.service openstack-swift-account-replicator.service  
systemctl start openstack-swift-object.service openstack-swift-object-auditor.service \
  openstack-swift-object-replicator.service openstack-swift-object-updater.service
```

###### 验证操作

```
echo "export OS_AUTH_VERSION=3" | tee -a admin-openrc.sh demo-openrc.sh
source demo-openrc.sh
swift stat
swift upload container1 FILE
swift list
swift download container1 FILE
```

















> swift liberasurecode-1.2.0-2.el7.x86_64.rpm  