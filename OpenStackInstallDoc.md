# OpenStack安装文档

## 环境架构

节点名称    | 节点IP   | hostname | 操作系统 | 内存  | 硬盘 |
----------|----------|----------|----------|----------|----------|
控制节点 | 192.168.0.231   | controller | CentOS 7.1 | 94G | 13T |
计算节点1 | 192.168.0.232 | compute1 | CentOS 7.1 | 8G | 240G |
计算节点2 | 192.168.0.233 | compute2 | CentOS 7.1 | 8G | 240G |
计算节点3 | 192.168.0.234 | compute3 | CentOS 7.1 | 8G | 240G |
计算节点4 | 192.168.0.235 | compute4 | CentOS 7.1 | 12G | 170G |
计算节点5 | 192.168.0.236 | compute5 | CentOS 7.1 | 4G | 240G |


## 控制节点安装

### 1. 设置计算机名及host信息

#### 修改计算机名：

```
hostnamectl --static set-hostname controller```
注销并重新登录后，提示符变成`root@controller`说明修改成功


#### 安装vim、net-tools等工具：

```
yum -y install vim net-tools
```

#### 设置host信息：

```
vim /etc/hosts
```
在最下面**增加**以下信息：

```
192.168.0.231 controller192.168.0.232 compute1192.168.0.233 compute2192.168.0.234 compute3192.168.0.235 compute4192.168.0.236 compute5
```


### 2. 安装NTP服务
#### 安装chrony

```
yum install chrony
```

#### 配置时钟源

```
vim /etc/chrony.conf
```

注释现有server开头的4行，并在下面增加以下内容：
```
server s1a.time.edu.cn iburst
server s1b.time.edu.cn iburst
server s1c.time.edu.cn iburst
server ntp.sjtu.edu.cn iburst
```

#### 启动chrony服务

```
systemctl enable chronyd.servicesystemctl start chronyd.service
```

#### 验证安装
```
chronyc sources
```

出现以下结果说明安装成功：

```
[root@controller ~]# chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 10.112.202.in-addr.arpa.1     2  10   377   733   +175us[ +221us] +/-   18ms
^? ntpa.nic.edu.cn               0  10     0   10y     +0ns[   +0ns] +/-    0ns
^? 202.112.7.150                 0  10     0   10y     +0ns[   +0ns] +/-    0ns
^- 202.120.2.100.dns.sjtu.ed     3  10     0  430m  -1478us[-1399us] +/-  272ms
```

### 3. 安装OpenStack基础包
```
yum -y install centos-release-openstack-ocatayum -y install https://rdoproject.org/repos/rdo-release.rpmyum upgradeyum -y install python-openstackclientyum -y install openstack-selinux
```

### 4. 安装必须应用包

```
yum -y install mariadb mariadb-server python2-PyMySQL
yum -y install rabbitmq-server
yum -y install memcached python-memcached
yum -y install openstack-keystone httpd mod_wsgi
yum -y install openstack-glance
yum -y install openstack-nova-api openstack-nova-conductor \
  openstack-nova-console openstack-nova-novncproxy \
  openstack-nova-scheduler openstack-nova-placement-api
yum -y install openstack-neutron openstack-neutron-ml2 \
  openstack-neutron-linuxbridge ebtables
yum -y install openstack-dashboard
```

### 5. 配置数据库

#### 新建配置文件

```
vim /etc/my.cnf.d/openstack.cnf
```

内容如下：

```
[mysqld]
bind-address = 192.168.0.231

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

其中`bind-address = 192.168.0.231`为控制节点的IP地址，要注意修改， `max_connections = 4096`为MySQL最大连接数，可适当修改避免连接数过多导致MySQL服务不可用。

#### 启动MySQL服务

```
systemctl enable mariadb.service
systemctl start mariadb.service
```

#### 配置MySQL

```
mysql_secure_installation
```
按照提示操作并设置MySQL root账户的密码。

#### 添加数据库及用户

```
mysql -u root -p
```
输入root密码并登录

```
CREATE DATABASE keystone;
CREATE DATABASE glance;
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;
CREATE DATABASE neutron;

GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY '9961c6b10cbac21bb8b7';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY '9961c6b10cbac21bb8b7';

GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'a8d6a4c68f29bdbf3579';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'a8d6a4c68f29bdbf3579';

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'c871cb0491d24642bc15';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'c871cb0491d24642bc15';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'c871cb0491d24642bc15';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'c871cb0491d24642bc15';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'c871cb0491d24642bc15';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'c871cb0491d24642bc15';


GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY '5578b526c4ac91ec97ad';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY '5578b526c4ac91ec97ad';
  
```

### 6. 配置消息队列

#### 启动RabbitMQ服务

```
systemctl enable rabbitmq-server.service
systemctl start rabbitmq-server.service
```

#### 添加RabbitMQ账户

添加账户：

```
rabbitmqctl add_user openstack 27b91c0a14c8c035ed31
```

设置账户权限：

```
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

### 7.配置Memcached

#### 编辑配置文件

```
vim /etc/sysconfig/memcached
```
内容如下：

```
PORT="11211"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS="-l 127.0.0.1,::1,controller"
```

#### 启动服务
```
systemctl enable memcached.service
systemctl start memcached.service
```

### 8. 配置Identity service

```
vim /etc/keystone/keystone.conf
```

内容如下：

```
[DEFAULT]
[assignment]
[auth]
[cache]
[catalog]
[cors]
[cors.subdomain]
[credential]
[database]
connection = mysql+pymysql://keystone:9961c6b10cbac21bb8b7@controller/keystone
[domain_config]
[endpoint_filter]
[endpoint_policy]
[eventlet_server]
[federation]
[fernet_tokens]
[healthcheck]
[identity]
[identity_mapping]
[kvs]
[ldap]
[matchmaker_redis]
[memcache]
[oauth1]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
[policy]
[profiler]
[resource]
[revoke]
[role]
[saml]
[security_compliance]
[shadow_users]
[signing]
[token]
provider = fernet
[tokenless_auth]
[trust]
```

#### 设置httpd服务

```
vim /etc/httpd/conf/httpd.conf
```
设置ServerName

```
ServerName controller
```

#### 同步数据库并启动服务

```
su -s /bin/sh -c "keystone-manage db_sync" keystone

keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

keystone-manage bootstrap --bootstrap-password fc8813dd591141bb2f0a \
  --bootstrap-admin-url http://controller:35357/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne

ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

systemctl enable httpd.service

systemctl start httpd.service
```

### 9. 创建OpenStack工程及用户

```
openstack project create --domain default \
  --description "Service Project" service

openstack project create --domain default \
  --description "Demo Project" demo

openstack user create --domain default \
  --password-prompt demo

openstack role create user

openstack role add --project demo --user demo user

openstack user create --domain default --password-prompt glance

openstack role add --project service --user glance admin

openstack service create --name glance \
  --description "OpenStack Image" image
  
openstack endpoint create --region RegionOne \
  image public http://controller:9292

openstack endpoint create --region RegionOne \
  image internal http://controller:9292

openstack endpoint create --region RegionOne \
  image admin http://controller:9292

openstack user create --domain default --password-prompt nova

openstack role add --project service --user nova admin

openstack service create --name nova \
  --description "OpenStack Compute" compute

openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1

openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1

openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1

openstack user create --domain default --password-prompt placement

openstack role add --project service --user placement admin

openstack service create --name placement --description "Placement API" placement

openstack endpoint create --region RegionOne placement public http://controller:8778

openstack endpoint create --region RegionOne placement internal http://controller:8778

openstack endpoint create --region RegionOne placement admin http://controller:8778

openstack user create --domain default --password-prompt neutron

openstack role add --project service --user neutron admin

openstack service create --name neutron \
  --description "OpenStack Networking" network

openstack endpoint create --region RegionOne \
  network public http://controller:9696

openstack endpoint create --region RegionOne \
  network internal http://controller:9696

openstack endpoint create --region RegionOne \
  network admin http://controller:9696


```

### 10. 配置glance

```
vim /etc/glance/glance-api.conf
```
内容如下：

```
[DEFAULT]
[cors]
[cors.subdomain]
[database]
connection = mysql+pymysql://glance:a8d6a4c68f29bdbf3579@controller/glance
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images
[image_format]
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = 1eab8126b25e9a4498e4
[matchmaker_redis]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]
```

```
vim /etc/glance/glance-registry.conf
```
内容如下：

```
[DEFAULT]
[database]
connection = mysql+pymysql://glance:a8d6a4c68f29bdbf3579@controller/glance
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = 1eab8126b25e9a4498e4
[matchmaker_redis]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
```

同步数据库并启动服务

```
su -s /bin/sh -c "glance-manage db_sync" glance
systemctl enable openstack-glance-api.service \
  openstack-glance-registry.service
systemctl start openstack-glance-api.service \
  openstack-glance-registry.service
```

### 11. 配置nova

```
vim /etc/nova/nova.conf
```

内容如下：

```
[DEFAULT]
my_ip=192.168.0.231
use_neutron=true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
enabled_apis=osapi_compute,metadata
transport_url = rabbit://openstack:27b91c0a14c8c035ed31@controller
[api]
auth_strategy=keystone
[api_database]
connection = mysql+pymysql://nova:c871cb0491d24642bc15@controller/nova_api
[barbican]
[cache]
[cells]
[cinder]
[cloudpipe]
[conductor]
[console]
[consoleauth]
[cors]
[cors.subdomain]
[crypto]
[database]
connection = mysql+pymysql://nova:c871cb0491d24642bc15@controller/nova
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
[guestfs]
[healthcheck]
[hyperv]
[image_file_url]
[ironic]
[key_manager]
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = d4059b96757f92e5cc3c
[libvirt]
[matchmaker_redis]
[metrics]
[mks]
[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = 1288a49d2f2f53dc6d3a
service_metadata_proxy = true
metadata_proxy_shared_secret = c5f261fb817b751d97ca
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path=/var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:35357/v3
username = placement
password = ba68271237605d679b5a
[quota]
[rdp]
[remote_debug]
[scheduler]
discover_hosts_in_cells_interval=300
[serial_console]
[service_user]
[spice]
[ssl]
[trusted_computing]
[upgrade_levels]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled=true
vncserver_listen=$my_ip
vncserver_proxyclient_address=$my_ip
[workarounds]
[wsgi]
[xenserver]
[xvp]
```

```
vim /etc/httpd/conf.d/00-nova-placement-api.conf
```

内容如下：

```
Listen 8778
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
<VirtualHost *:8778>
  WSGIProcessGroup nova-placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
  WSGIDaemonProcess nova-placement-api processes=3 threads=1 user=nova group=nova
  WSGIScriptAlias / /usr/bin/nova-placement-api
  <IfVersion >= 2.4>
    ErrorLogFormat "%M"
  </IfVersion>
  ErrorLog /var/log/nova/nova-placement-api.log
</VirtualHost>
Alias /nova-placement-api /usr/bin/nova-placement-api
<Location /nova-placement-api>
  SetHandler wsgi-script
  Options +ExecCGI
  WSGIProcessGroup nova-placement-api
  WSGIApplicationGroup %{GLOBAL}
  WSGIPassAuthorization On
</Location>
```

同步数据库并启动服务

```
systemctl restart httpd

su -s /bin/sh -c "nova-manage api_db sync" nova

su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

su -s /bin/sh -c "nova-manage db sync" nova

nova-manage cell_v2 list_cells

systemctl enable openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service
systemctl start openstack-nova-api.service \
  openstack-nova-consoleauth.service openstack-nova-scheduler.service \
  openstack-nova-conductor.service openstack-nova-novncproxy.service

```

### 12. 配置Neutron

```
vim /etc/neutron/neutron.conf
```

内容如下：

```
[DEFAULT]
auth_strategy = keystone
core_plugin = neutron.plugins.ml2.plugin.Ml2Plugin
service_plugins = odl-router_v2
allow_overlapping_ips = true
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
transport_url = rabbit://openstack:27b91c0a14c8c035ed31@controller
[agent]
[cors]
[cors.subdomain]
[database]
connection = mysql+pymysql://neutron:5578b526c4ac91ec97ad@controller/neutron
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 1288a49d2f2f53dc6d3a
[matchmaker_redis]
[nova]
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = d4059b96757f92e5cc3c
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[qos]
[quotas]
[ssl]
```

```
vim /etc/neutron/plugins/ml2/ml2_conf.ini
```

内容如下：

```
[DEFAULT]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = opendaylight_v2
extension_drivers = port_security
[ml2_type_flat]
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
enable_security_group = true
enable_ipset = true
[ml2_odl]
url = http://192.168.0.231:8080/controller/nb/v2/neutron
password = admin
username = admin
```

```
vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

内容如下：

```
[DEFAULT]
[agent]
[linux_bridge]
physical_interface_mappings = provider:ens9f0
[securitygroup]
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
enable_security_group = true
[vxlan]
enable_vxlan = true
local_ip = 192.168.0.231
l2_population = true
```

```
vim /etc/neutron/dhcp_agent.ini
```

内容如下：

```
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
force_metadata = true
[agent]
[ovs]
```

```
vim /etc/neutron/metadata_agent.ini
```

内容如下：

```
[DEFAULT]
nova_metadata_ip = controller
metadata_proxy_shared_secret = c5f261fb817b751d97ca
[agent]
[cache]
```

同步数据库并启动服务：

```
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini

su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

systemctl restart openstack-nova-api.service

systemctl enable neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service

systemctl start neutron-server.service \
  neutron-linuxbridge-agent.service neutron-dhcp-agent.service \
  neutron-metadata-agent.service
```

### 13. 配置Dashboard

```
vim /etc/openstack-dashboard/local_settings
```

内容如下：

```
import os
from django.utils.translation import ugettext_lazy as _
from openstack_dashboard.settings import HORIZON_CONFIG
DEBUG = False
WEBROOT = '/dashboard/'
ALLOWED_HOSTS = ['*']
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
LOCAL_PATH = '/tmp'
SECRET_KEY='06131b97f4c003929261'
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
OPENSTACK_HOST = "controller"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "_member_"
OPENSTACK_KEYSTONE_BACKEND = {
    'name': 'native',
    'can_edit_user': True,
    'can_edit_group': True,
    'can_edit_project': True,
    'can_edit_domain': True,
    'can_edit_role': True,
}
OPENSTACK_HYPERVISOR_FEATURES = {
    'can_set_mount_point': False,
    'can_set_password': False,
    'requires_keypair': False,
    'enable_quotas': True
}
OPENSTACK_CINDER_FEATURES = {
    'enable_backup': False,
}
OPENSTACK_NEUTRON_NETWORK = {
    'enable_router': True,
    'enable_quotas': True,
    'enable_ipv6': True,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': True,
    'enable_firewall': True,
    'enable_vpn': True,
    'enable_fip_topology_check': True,
    'supported_vnic_types': ['*'],
}
OPENSTACK_HEAT_STACK = {
    'enable_user_pass': True,
}
IMAGE_CUSTOM_PROPERTY_TITLES = {
    "architecture": _("Architecture"),
    "kernel_id": _("Kernel ID"),
    "ramdisk_id": _("Ramdisk ID"),
    "image_state": _("Euca2ools state"),
    "project_id": _("Project ID"),
    "image_type": _("Image Type"),
}
IMAGE_RESERVED_CUSTOM_PROPERTIES = []
API_RESULT_LIMIT = 1000
API_RESULT_PAGE_SIZE = 20
SWIFT_FILE_TRANSFER_CHUNK_SIZE = 512 * 1024
INSTANCE_LOG_LENGTH = 35
DROPDOWN_MAX_ITEMS = 30
TIME_ZONE = "Asia/Shanghai"
POLICY_FILES_PATH = '/etc/openstack-dashboard'
LOGGING = {
    'version': 1,
explicitly.
    'disable_existing_loggers': False,
    'formatters': {
        'operation': {
            'format': '%(asctime)s %(message)s'
        },
    },
    'handlers': {
        'null': {
            'level': 'DEBUG',
            'class': 'logging.NullHandler',
        },
        'console': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
        },
        'operation': {
            'level': 'INFO',
            'class': 'logging.StreamHandler',
            'formatter': 'operation',
        },
    },
    'loggers': {
        'django.db.backends': {
            'handlers': ['null'],
            'propagate': False,
        },
        'requests': {
            'handlers': ['null'],
            'propagate': False,
        },
        'horizon': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'horizon.operation_log': {
            'handlers': ['operation'],
            'level': 'INFO',
            'propagate': False,
        },
        'openstack_dashboard': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'novaclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'cinderclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'keystoneclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'glanceclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'neutronclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'heatclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'swiftclient': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'openstack_auth': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'nose.plugins.manager': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'django': {
            'handlers': ['console'],
            'level': 'DEBUG',
            'propagate': False,
        },
        'iso8601': {
            'handlers': ['null'],
            'propagate': False,
        },
        'scss': {
            'handlers': ['null'],
            'propagate': False,
        },
    },
}
SECURITY_GROUP_RULES = {
    'all_tcp': {
        'name': _('All TCP'),
        'ip_protocol': 'tcp',
        'from_port': '1',
        'to_port': '65535',
    },
    'all_udp': {
        'name': _('All UDP'),
        'ip_protocol': 'udp',
        'from_port': '1',
        'to_port': '65535',
    },
    'all_icmp': {
        'name': _('All ICMP'),
        'ip_protocol': 'icmp',
        'from_port': '-1',
        'to_port': '-1',
    },
    'ssh': {
        'name': 'SSH',
        'ip_protocol': 'tcp',
        'from_port': '22',
        'to_port': '22',
    },
    'smtp': {
        'name': 'SMTP',
        'ip_protocol': 'tcp',
        'from_port': '25',
        'to_port': '25',
    },
    'dns': {
        'name': 'DNS',
        'ip_protocol': 'tcp',
        'from_port': '53',
        'to_port': '53',
    },
    'http': {
        'name': 'HTTP',
        'ip_protocol': 'tcp',
        'from_port': '80',
        'to_port': '80',
    },
    'pop3': {
        'name': 'POP3',
        'ip_protocol': 'tcp',
        'from_port': '110',
        'to_port': '110',
    },
    'imap': {
        'name': 'IMAP',
        'ip_protocol': 'tcp',
        'from_port': '143',
        'to_port': '143',
    },
    'ldap': {
        'name': 'LDAP',
        'ip_protocol': 'tcp',
        'from_port': '389',
        'to_port': '389',
    },
    'https': {
        'name': 'HTTPS',
        'ip_protocol': 'tcp',
        'from_port': '443',
        'to_port': '443',
    },
    'smtps': {
        'name': 'SMTPS',
        'ip_protocol': 'tcp',
        'from_port': '465',
        'to_port': '465',
    },
    'imaps': {
        'name': 'IMAPS',
        'ip_protocol': 'tcp',
        'from_port': '993',
        'to_port': '993',
    },
    'pop3s': {
        'name': 'POP3S',
        'ip_protocol': 'tcp',
        'from_port': '995',
        'to_port': '995',
    },
    'ms_sql': {
        'name': 'MS SQL',
        'ip_protocol': 'tcp',
        'from_port': '1433',
        'to_port': '1433',
    },
    'mysql': {
        'name': 'MYSQL',
        'ip_protocol': 'tcp',
        'from_port': '3306',
        'to_port': '3306',
    },
    'rdp': {
        'name': 'RDP',
        'ip_protocol': 'tcp',
        'from_port': '3389',
        'to_port': '3389',
    },
}
REST_API_REQUIRED_SETTINGS = ['OPENSTACK_HYPERVISOR_FEATURES',
                              'LAUNCH_INSTANCE_DEFAULTS',
                              'OPENSTACK_IMAGE_FORMATS',
                              'OPENSTACK_KEYSTONE_DEFAULT_DOMAIN']
ALLOWED_PRIVATE_SUBNET_CIDR = {'ipv4': [], 'ipv6': []}
```

启动服务

```
systemctl restart httpd.service memcached.service
```
## 计算节点安装


### 1. 设置计算机名及host信息

#### 修改计算机名：

```
hostnamectl --static set-hostname compute1```
注销并重新登录后，提示符变成`root@ compute1 `说明修改成功


#### 安装vim、net-tools等工具：

```
yum -y install vim net-tools
```

#### 设置host信息：

```
vim /etc/hosts
```
在最下面**增加**以下信息：

```
192.168.0.231 controller192.168.0.232 compute1192.168.0.233 compute2192.168.0.234 compute3192.168.0.235 compute4192.168.0.236 compute5
```


### 2. 安装NTP服务
#### 安装chrony

```
yum install chrony
```

#### 配置时钟源

```
vim /etc/chrony.conf
```

注释现有server开头的4行，并在下面增加以下内容：
```
server controller iburst
```

#### 启动chrony服务

```
systemctl enable chronyd.servicesystemctl start chronyd.service
```

#### 验证安装
```
chronyc sources
```

出现以下结果说明安装成功：

```
[root@controller ~]# chronyc sources
210 Number of sources = 4
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* 10.112.202.in-addr.arpa.1     2  10   377   733   +175us[ +221us] +/-   18ms
^? ntpa.nic.edu.cn               0  10     0   10y     +0ns[   +0ns] +/-    0ns
^? 202.112.7.150                 0  10     0   10y     +0ns[   +0ns] +/-    0ns
^- 202.120.2.100.dns.sjtu.ed     3  10     0  430m  -1478us[-1399us] +/-  272ms
```

### 3. 安装OpenStack基础包
```
yum -y install centos-release-openstack-ocatayum -y install https://rdoproject.org/repos/rdo-release.rpmyum upgradeyum -y install python-openstackclientyum -y install openstack-selinux
```

### 4. 安装Nova

```
yum -y install openstack-nova-compute
```

```
vim /etc/nova/nova.conf
```

内容如下：

```
[DEFAULT]my_ip=192.168.0.234auth_strategy=keystonefirewall_driver = nova.virt.firewall.NoopFirewallDriveruse_neutron=truetransport_url=rabbit://openstack:27b91c0a14c8c035ed31@controller[api_database][barbican][cache][cells][cinder][conductor][cors][cors.subdomain][database][ephemeral_storage_encryption][glance]api_servers=http://controller:9292[guestfs][hyperv][image_file_url][ironic][keymgr][keystone_authtoken]auth_uri=http://controller:5000auth_url = http://controller:35357memcached_servers=controller:11211auth_type=passwordproject_domain_name = defaultuser_domain_name = defaultproject_name = serviceusername = novapassword = d4059b96757f92e5cc3c[libvirt][matchmaker_redis][metrics][neutron]url = http://controller:9696auth_url = http://controller:35357auth_type = passwordproject_domain_name = defaultuser_domain_name = defaultregion_name = RegionOneproject_name = serviceusername = neutronpassword = 1288a49d2f2f53dc6d3a[osapi_v21][oslo_concurrency]lock_path=/var/lib/nova/tmp[oslo_messaging_amqp][oslo_messaging_notifications][oslo_messaging_rabbit][oslo_middleware][oslo_policy][rdp][serial_console][spice][ssl][trusted_computing][upgrade_levels][vmware][vnc]enabled=truevncserver_listen=0.0.0.0vncserver_proxyclient_address=$my_ipnovncproxy_base_url=http://controller:6080/vnc_auto.html[workarounds][xenserver][placement]auth_uri = http://controller:5000auth_url = http://controller:35357memcached_servers = controller:11211auth_type = passwordproject_domain_name = defaultuser_domain_name = defaultproject_name = serviceusername = placementpassword = ba68271237605d679b5aos_region_name = RegionOne[placement_database]connection=mysql+pymysql://nova:c871cb0491d24642bc15@controller/nova_api
```

启动服务

```
systemctl enable libvirtd.service openstack-nova-compute.servicesystemctl start libvirtd.service openstack-nova-compute.service
```

### 5. 安装Neutron

```
yum -y install openstack-neutron-linuxbridge ebtables ipset
```

```
vim /etc/neutron/neutron.conf
```

内容如下：

```
[DEFAULT]auth_strategy = keystonerpc_backend = rabbit[agent][cors][cors.subdomain][database][keystone_authtoken]auth_uri = http://controller:5000auth_url = http://controller:35357memcached_servers = controller:11211auth_type = passwordproject_domain_name = defaultuser_domain_name = defaultproject_name = serviceusername = neutronpassword = 1288a49d2f2f53dc6d3a[matchmaker_redis][nova][oslo_concurrency]lock_path = /var/lib/neutron/tmp[oslo_messaging_amqp][oslo_messaging_kafka][oslo_messaging_notifications][oslo_messaging_rabbit]rabbit_host = controllerrabbit_userid = openstackrabbit_password = 27b91c0a14c8c035ed31[oslo_messaging_zmq][oslo_middleware][oslo_policy][qos][quotas][ssl]
```

```
vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

内容如下：

```
[DEFAULT][agent][linux_bridge]physical_interface_mappings = provider:enp5s0f1[securitygroup]firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriverenable_security_group = true[vxlan]enable_vxlan = truelocal_ip = 192.168.0.234l2_population = true
```

启动服务

```
systemctl restart openstack-nova-compute.servicesystemctl enable neutron-linuxbridge-agent.servicesystemctl restart neutron-linuxbridge-agent.service
```

## 控制节点OpenDaylight安装

### 安装OpenDaylight

```
tar xvfz distribution-karaf-0.5.1-Boron-SR1.tar.gz
cd distribution-karaf-0.5.1-Boron-SR1
./bin/start
```

安装features

```
./bin/client # Connect to OpenDaylight with the client
opendaylight-user@root> feature:install odl-netvirt-openstack odl-dlux-core odl-mdsal-apidocs
```

安装OpenvSwitch

```
yum -y install openvswitch
```

清空配置：

```
systemctl stop openvswitch
rm -rf /var/log/openvswitch/*
rm -rf /etc/openvswitch/conf.db
systemctl start openvswitch
```

设置OpenvSwitch：

```
ovs-vsctl set-manager tcp:192.168.0.231:6640
ovs-vsctl set Open_vSwitch . other_config:local_ip=192.168.0.231
ovs-vsctl set Open_vSwitch . other_config:provider_mappings=provider:enp11s0f0

ovs-vsctl show
ovs-vsctl get Open_vSwitch . other_config
```

重置neutron数据库

```
mysql -e "DROP DATABASE IF EXISTS neutron;"
mysql -e "CREATE DATABASE neutron CHARACTER SET utf8;"
/usr/local/bin/neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head
```

### 编译安装networking-odl

```
git clone https://github.com/openstack/networking-odl.git
cd networking-odl
python setup.py
```

注意前面nova和neutron的配置文件已经包含有OpenDaylight的配置信息，此处无需再次配置。

## 计算节点OpenvSwitch配置

安装OpenvSwitch

```
yum -y install openvswitch
```

清空配置：

```
systemctl stop openvswitch
rm -rf /var/log/openvswitch/*
rm -rf /etc/openvswitch/conf.db
systemctl start openvswitch
```

设置OpenvSwitch：

```
ovs-vsctl set-manager tcp:192.168.0.231:6640
ovs-vsctl set Open_vSwitch . other_config:local_ip=192.168.0.232
ovs-vsctl set Open_vSwitch . other_config:provider_mappings=provider:enp11s0f0

ovs-vsctl show
ovs-vsctl get Open_vSwitch . other_config
```

同步主机设置：

```
neutron-odl-ovs-hostconfig --ovs_hostconfigs='{"ODL L2": \
{"allowed_network_types":["flat","vlan","vxlan"],"bridge_mappings": \
{"provider":"ens9f0"},"supported_vnic_types": \
[{"vnic_type":"normal","vif_type":"ovs","vif_details":{}}]}, \
"ODL L3":{"allowed_network_types":["flat","vlan","vxlan"]}}'
```

关闭防火墙：

```
systemctl stop firewalld.service
systemctl disable firewalld.service
```