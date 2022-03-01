# Mô hình triển khai
```sh
   ------------+---------------------------+---------------------------+---------------------------+------------
               |                           |                           |                           |
          ens38|10.0.0.30             ens38|10.0.0.50             ens38|10.0.0.51             ens38| 10.0.0.52
   +-----------+-----------+   +-----------+-----------+   +-----------+-----------+   +-----------+-----------+
   |    [ Control Node ]   |   |    [ Network Node ]   |   |    [ Compute Node ]   |   |    [ Storage Node ]   |
   |                       |   |                       |   |                       |   |          lvm2         |
   |  MariaDB    RabbitMQ  |   |        L2 Agent       |   |        Libvirt        |   |     Cinder-volume     |
   |  Memcached  Apache2   |   |        L3 Agent       |   |     Nova Compute      |   |                       |
   |  Keystone   Glance    |   |     Metadata Agent    |   |        L2 Agent       |   |                       |
   | Placement API Nova API|   |       DHCP Agent      |   |  Linux Bridge Agent   |   |                       |
   |  Neutron Server       |   |  Linux Bridge Agent   |   |                       |   |                       |
   |  Metadata Agent       |   |                       |   |                       |   |                       | 
   +-----------------------+   +-----------------------+   +----------+------------+   +-----------------------+
                                           |
                                      ens33|

```

# Cài đặt dịch vụ bổ trợ cho Openstack
### Trên node Controller
#### 1. Cấu hình file host
```sh
root@controller:~# vim /etc/hosts
10.0.0.30       controller
```
#### 2. Cài đặt Chrony (Đồng bộ hoá thời gian)

```sh
root@controller:~# apt install chrony -y
```

- Cấu hình chrony

```sh
root@controller:~# vim /etc/chrony/chrony.conf
pool vn.pool.ntp.org iburst   //Server NTP
allow 10.0.0.0/24        //Cho phép các node kết nối với chrony deamon
```

- Khởi động lại dịch vụ
```sh
root@controller:~# service chrony restart
```

#### 3. Add repo Openstack

```sh
root@controller:~# add-apt-repository cloud-archive:victoria
```

#### 4. Cài đặt MariaDB

- Các dịch vụ của Openstack đều sử dụng SQL database để lưu trữ thông tin

```sh
root@controller:~# apt install mariadb-server python3-pymysql
```

- Cấu hình MariaDB

```sh
root@controller:~#vim /etc/mysql/mariadb.conf.d/99-openstack.cnf
[mysqld]
bind-address = 10.0.0.30 //Các node có thể truy cập vào DB qua địa chỉ này

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

- Khởi động lại dịch vụ

```sh
root@controller:~# service mysql restart
```

#### 5. Cài đặt RabbitMQ

- Sử dụng để điều phối hoạt động và thông tin trạng thái của các dịch vụ Openstack

 ```sh
 root@controller:~# apt install rabbitmq-server
 ```
 
 - Thêm User openstack và cho phép user openstack configuration, write và read
```sh
root@controller:~# rabbitmqctl add_user openstack RABBIT_PASS
root@controller:~# rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### 6. Cài đặt Memcache

- Cơ chế xác thực dịch vụ cho các dịch vụ sử dụng Memcached để lưu cache tokens
```sh
root@controller:~# apt install memcached python3-memcache
```

- Cấu hình memcache

```sh
root@controller:~# vim /etc/memcached.conf
-l 10.0.0.30  ////Các node có thể truy cập qua địa chỉ này
 ```

- Khởi động lại dịch vụ

```sh
root@controller:~# service memcached restart
```
#### 8. Cài đặt dịch vụ openstack-client để sử dụng client openstack
```sh
root@controller:~# apt install python3-openstackclient
```

### Trên tất cả các node còn lại
#### 1. Cấu hình file host
```sh
root@compute:~# vim /etc/hosts
10.0.0.30       controller
```
#### 2. Cài đặt Chrony (Đồng bộ hoá thời gian)

```sh
root@compute:~# apt install chrony -y
```

- Cấu hình chrony

```sh
root@controller:~# vim /etc/chrony/chrony.conf
pool controller iburst   //Server NTP
```

- Khởi động lại dịch vụ
```sh
root@controller:~# service chrony restart
```

- Kiểm tra hoạt động
```sh
root@compute:~# chronyc sources
210 Number of sources = 1
MS Name/IP address         Stratum Poll Reach LastRx Last sample
===============================================================================
^* controller                    3   6   177    14   +324us[+3022us] +/-   27ms
```


#### 3. Add repo Openstack

```sh
root@controller:~# add-apt-repository cloud-archive:victoria
```

#### 4. Cài đặt dịch vụ openstack-client để sử dụng client openstack
```sh
root@compute:~# apt install python3-openstackclient
```
# Cài đặt dịch vụ Openstack

### Cài đặt dịch vụ identity (Keystone)
#### Trên node controller
- Khởi tạo database cho keystone
```sh
root@controller:~# mysql
MariaDB [(none)]> CREATE DATABASE keystone;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost'  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%'  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> exit
```

- Cài đặt dịch vụ keystone

```sh
root@controller:~# apt install keystone
```

- Chỉnh sửa file cấu hình dịch vụ keystone

```sh
root@controller:~#  vim /etc/keystone/keystone.conf

[database] //cấu hình quyền truy cập database
connection = mysql+pymysql://keystone:lean15998@controller/keystone
[token] //cấu hình nhà cung cấp token là fernet
provider = fernet
```

- Populate vào CSDL dịch vụ keystone

```sh
root@controller:~# su -s /bin/sh -c "keystone-manage db_sync" keystone
```

- 	Khởi tạo kho khoá fernet
```sh
root@controller:~# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
root@controller:~# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```


- Bootstrap dịch vụ

```sh
root@controller:~# keystone-manage bootstrap --bootstrap-password lean15998 \
>   --bootstrap-admin-url http://controller:5000/v3/ \
>   --bootstrap-internal-url http://controller:5000/v3/ \
>   --bootstrap-public-url http://controller:5000/v3/ \
>   --bootstrap-region-id RegionOne
```

- Cấu hình Apache

```sh
root@controller:~# vim /etc/apache2/apache2.conf
ServerName controller
```

- Khởi động lại dịch vụ

```sh
root@controller:~# service apache2 restart
```

- 	Tạo script chứa thông tin đăng nhập vào dịch vụ identity

```sh
-	Tạo script chứa thông tin đăng nhập vào dịch vụ identity
root@controller:~# vim admin-openrc

export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=lean15998
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

-	Khởi tạo project service

```sh
root@controller:~# openstack project create --domain default \
>   --description "Service Project" service
```
- Xem thông tin project

```sh
root@controller:~# openstack project show service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 8dd57fe0ce7e49dc9652d40381c7f1f2 |
| is_domain   | False                            |
| name        | service                          |
| options     | {}                               |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

### Cài đặt dịch vụ image (Glance)
#### Trên node controller
- Khởi tạo database cho glance
```sh
root@controller:~# mysql
MariaDB [(none)]> CREATE DATABASE glance;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost'  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%'    IDENTIFIED BY 'lean15998';
MariaDB [(none)]> exit
```

- 	Tạo thông tin xác thực dịch vụ cho glance

```sh
root@controller:~# openstack user create --domain default --password-prompt glance
root@controller:~# openstack role add --project service --user glance admin
root@controller:~# openstack service create --name glance \
>   --description "OpenStack Image" image
root@controller:~# openstack endpoint create --region RegionOne \
>   image public http://controller:9292
root@controller:~# openstack endpoint create --region RegionOne \
>   image internal http://controller:9292
root@controller:~# openstack endpoint create --region RegionOne \
>   image admin http://controller:9292
```

- 	Cài đặt dịch vụ glance
```sh
root@controller:~# apt install glance
```

- Chỉnh sửa file cấu hình dịch vụ glance

```sh
root@controller:~# vim /etc/glance/glance-api.conf

[database]  //truy cập CSDL
connection = mysql+pymysql://glance:lean15998@controller/glance

[keystone_authtoken] //thông tin truy cập keystone
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = lean15998

[paste_deploy]
flavor = keystone

[glance_store] //vị trí lưu image
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

```

- Populate vào CSDL dịch vụ glance

```sh
root@controller:~# su -s /bin/sh -c "glance-manage db_sync" glance
```

- Khởi động lại dịch vụ

```sh
root@controller:~# service glance-api restart
```

-	Tải image và upload lên dịch vụ glance
```
root@controller:~#wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
root@controller:~# glance image-create --name "cirros" \
>   --file cirros-0.4.0-x86_64-disk.img \
>   --disk-format qcow2 --container-format bare \
>   --visibility=public
root@controller:~# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 33e9b1ef-5738-4aa2-9ba5-d4008742b543 | cirros | active |
+--------------------------------------+--------+--------+
```


### Cài đặt dịch vụ Placement
#### Trên node controller
- Khởi tạo database cho placement
```sh
root@controller:~# mysql
MariaDB [(none)]> CREATE DATABASE placement;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost'  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%'  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> exit
```

- 	Tạo thông tin xác thực dịch vụ cho placement

```sh
root@controller:~# openstack user create --domain default --password-prompt placement
root@controller:~# openstack role add --project service --user placement admin
root@controller:~# openstack service create --name placement placement
root@controller:~# openstack endpoint create --region RegionOne \
>   placement public http://controller:8778
root@controller:~# openstack endpoint create --region RegionOne \
>   placement internal http://controller:8778
root@controller:~# openstack endpoint create --region RegionOne \
>   placement admin http://controller:8778
```

- 	Cài đặt dịch vụ placement
```sh
root@controller:~# apt install placement-api
```

- Chỉnh sửa file cấu hình dịch vụ placement

```sh
root@controller:~# vim  /etc/placement/placement.conf

[placement_database] //truy cập CSDL
connection = mysql+pymysql://placement:lean15998@controller/placement

[api]
auth_strategy = keystone

[keystone_authtoken] //thông tin truy cập keystone
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = lean15998
```


- Populate vào CSDL placement

```sh
root@controller:~# su -s /bin/sh -c "placement-manage db sync" placement
```

- Khởi động lại dịch vụ

```sh
root@controller:~# service apache2 restart
```

### Cài đặt dịch vụ Compute (Nova)

#### Trên node Controller

- Khởi tạo database cho nova
```sh
root@controller:~# mysql
MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> exit
```

- 	Tạo thông tin xác thực dịch vụ cho nova

```sh
root@controller:~# openstack user create --domain default --password-prompt nova
root@controller:~# openstack role add --project service --user nova admin
root@controller:~# openstack service create --name nova \
  --description "OpenStack Compute" compute
root@controller:~# openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1
root@controller:~# openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1
root@controller:~# openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1
```

- 	Cài đặt dịch vụ nova
```sh
root@controller:~# apt install nova-api nova-conductor nova-novncproxy nova-scheduler
```

- Chỉnh sửa file cấu hình dịch vụ nova

```sh
root@controller:~# vim /etc/nova/nova.conf


[DEFAULT]
transport_url = rabbit://openstack:lean15998@controller:5672/
my_ip = 10.0.0.30
[api_database]
connection = mysql+pymysql://nova:lean15998@controller/nova_api

[database]
connection = mysql+pymysql://nova:lean15998@controller/nova

[api]
auth_strategy = keystone

[keystone_authtoken] //thông tin truy cập keystone
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = lean15998

[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]  //configure the location of the Image service API
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]  // truy cập placement
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = lean15998
```


- Populate vào CSDL nova-api, nova và đăng ký database cell0 và cell1

```sh
root@controller:~# su -s /bin/sh -c "nova-manage api_db sync" nova
root@controller:~# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
root@controller:~# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
root@controller:~# su -s /bin/sh -c "nova-manage db sync" nova
```

- Khởi động lại dịch vụ

```sh
root@controller:~# service nova-api restart
root@controller:~# service nova-scheduler restart
root@controller:~# service nova-conductor restart
root@controller:~# service nova-novncproxy restart
```


#### Trên node compute

- 	Cài đặt dịch vụ nova-compute
```sh
root@compute:~# apt install nova-compute
```

- Chỉnh sửa file cấu hình dịch vụ nova

```sh
root@compute:~# vim /etc/nova/nova.conf


[DEFAULT]
transport_url = rabbit://openstack:lean15998@controller:5672/
my_ip = 10.0.0.51

[api]
auth_strategy = keystone

[keystone_authtoken] //thông tin truy cập keystone
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = lean15998

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://10.0.0.30:6080/vnc_auto.html

[glance]  //configure the location of the Image service API
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]  // truy cập placement
# ...
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = lean15998
```

- Cấu hình nova-compute

```sh
root@compute:~# vim /etc/nova/nova-compute.conf

[libvirt]
virt_type = qemu
```


- Khởi động lại dịch vụ

```sh
root@compute:~# service nova-compute restart
```

#### Trên node Controller (Thêm node compute vào cell database)

```sh
root@controller:~# openstack compute service list --service nova-compute
+----+--------------+------------+------+---------+-------+----------------------------+
| ID | Binary       | Host       | Zone | Status  | State | Updated At                 |
+----+--------------+------------+------+---------+-------+----------------------------+
|  6 | nova-compute | compute    | nova | enabled | up    | 2022-03-01T02:30:23.000000 |
+----+--------------+------------+------+---------+-------+----------------------------+
root@controller:~# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```


### Cài đặt dịch vụ Network (Neutron)

#### Trên node Controller

- Khởi tạo database cho neutron
```sh
root@controller:~# mysql
MariaDB [(none)] CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> exit
```

- 	Tạo thông tin xác thực dịch vụ cho neutron

```sh
root@controller:~# openstack user create --domain default --password-prompt neutron
root@controller:~# openstack role add --project service --user neutron admin
root@controller:~# openstack service create --name neutron \
  --description "OpenStack Networking" network
root@controller:~# openstack endpoint create --region RegionOne \
  network public http://controller:9696
root@controller:~# openstack endpoint create --region RegionOne \
  network internal http://controller:9696
root@controller:~# openstack endpoint create --region RegionOne \
  network admin http://controller:9696
```

- 	Cài đặt dịch vụ neutron
```sh
root@controller:~# apt -y install neutron-server neutron-metadata-agent neutron-plugin-ml2 python3-neutronclient
```

- Chỉnh sửa file cấu hình dịch vụ netron

```sh
root@controller:~# vim /etc/neutron/neutron.conf

[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:lean15998@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = lean15998

[database]
connection = mysql+pymysql://neutron:lean15998@controller/neutron

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = lean15998

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

```sh

root@controller:~# vim /etc/neutron/metadata_agent.ini

[DEFAULT]
nova_metadata_host = 10.0.0.30
metadata_proxy_shared_secret = lean15998

```
```sh
root@controller:~# vim /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 1:1000
```
```sh
root@controller:~# vi /etc/nova/nova.conf

[DEFAULT]
use_neutron = True

[neutron]
auth_url = http://10.0.0.30:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = lean15998
service_metadata_proxy = True
metadata_proxy_shared_secret = lean15998
```


- Populate vào CSDL neutron

```sh
root@controller:~# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

- Khởi động lại dịch vụ

```sh
root@controller:~# systemctl restart neutron-server neutron-metadata-agent nova-api
root@controller:~# systemctl enable neutron-server neutron-metadata-agent
```

#### Trên node network

- 	Cài đặt dịch vụ neutron
```sh
root@network:~# apt -y install neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent python3-neutronclient
```

- Chỉnh sửa file cấu hình dịch vụ netron

```sh
root@network:~# vim /etc/neutron/neutron.conf

[DEFAULT]
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
state_path = /var/lib/neutron
allow_overlapping_ips = True
# RabbitMQ connection info
transport_url = rabbit://openstack:lean15998@10.0.0.30

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = lean15998

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

```sh
root@network:~# vi /etc/neutron/l3_agent.ini

[DEFAULT]
interface_driver = linuxbridge

```
```sh
root@network:~# vim /etc/neutron/dhcp_agent.ini

[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

```sh
root@network:~# vi /etc/neutron/metadata_agent.ini

[DEFAULT]
nova_metadata_host = 10.0.0.30
metadata_proxy_shared_secret = lean15998
```

```sh
root@network:~# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 1:1000
```

```sh
root@network:~# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[agent]
prevent_arp_spoofing = True

[linux_bridge]
physical_interface_mappings = provider:ens33

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

[vxlan]
enable_vxlan = true
local_ip = 10.0.0.50
```

- Khởi động lại dịch vụ

```sh
root@network:~# for service in l3-agent dhcp-agent metadata-agent linuxbridge-agent; do
systemctl restart neutron-$service
systemctl enable neutron-$service
done
```


#### Trên node compute

- 	Cài đặt dịch vụ neutron
```sh
root@compute:~# apt -y install neutron-common neutron-plugin-ml2 neutron-linuxbridge-agent
```

- Chỉnh sửa file cấu hình dịch vụ netron

```sh
root@compute:~# vi /etc/neutron/neutron.conf

[DEFAULT]
core_plugin = ml2
service_plugins = router
auth_strategy = keystone
state_path = /var/lib/neutron
allow_overlapping_ips = True
# RabbitMQ connection info
transport_url = rabbit://openstack:lean15998@controller

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = lean15998

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

```sh
root@compute:~# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 1:1000
```

```sh
root@compute:~# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[agent]
prevent_arp_spoofing = True

[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

[vxlan]
enable_vxlan = true
local_ip = 10.0.0.51
```

```sh
root@compute:~# vi /etc/nova/nova.conf

[DEFAULT]
use_neutron = True

[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = lean15998
service_metadata_proxy = true
metadata_proxy_shared_secret = lean15998
```

- Khởi động lại dịch vụ

```sh
root@compute:~# systemctl restart nova-compute neutron-linuxbridge-agent
root@compute:~# systemctl enable neutron-linuxbridge-agent
```

#### Trên node controller

- Danh sách neutron agent

```sh
root@controller:~# openstack network agent list
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 1d3e83ec-88d7-4c36-a284-aa320fd989dc | DHCP agent         | network    | nova              | :-)   | UP    | neutron-dhcp-agent        |
| 57d55b5b-cedf-4797-bff7-981186c55cd0 | Linux bridge agent | compute    | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 6f4100fb-2d8f-47c3-9722-6a2b1db2fa52 | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| 7278ca22-157e-4c8e-8b1a-fa411bf4ae29 | L3 agent           | network    | nova              | :-)   | UP    | neutron-l3-agent          |
| b464d34d-2bd2-4c45-8abe-0c74b241a368 | Linux bridge agent | network    | None              | :-)   | UP    | neutron-linuxbridge-agent |
| ba959d89-8134-446b-9d80-d8154b5db756 | Metadata agent     | network    | None              | :-)   | UP    | neutron-metadata-agent    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
```

- Tạo internal network và external network

```sh
root@controller:~# openstack router create router01
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2022-03-01T03:02:49Z                 |
| description             |                                      |
| distributed             | False                                |
| external_gateway_info   | null                                 |
| flavor_id               | None                                 |
| ha                      | False                                |
| id                      | 45bae2ed-bae4-4937-9f85-88409fa2f43c |
| name                    | router01                             |
| project_id              | f99cdd3d3d1344ba858ff4beb80fe624     |
| revision_number         | 1                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| updated_at              | 2022-03-01T03:02:49Z                 |
+-------------------------+--------------------------------------+
root@controller:~# openstack network create private --provider-network-type vxlan
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2022-03-01T03:02:55Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 48febb12-db06-4ec2-bdc5-df28277688b0 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | private                              |
| port_security_enabled     | True                                 |
| project_id                | f99cdd3d3d1344ba858ff4beb80fe624     |
| provider:network_type     | vxlan                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 1                                    |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2022-03-01T03:02:55Z                 |
+---------------------------+--------------------------------------+
root@controller:~# openstack subnet create private-subnet --network private \
> --subnet-range 192.168.100.0/24 --gateway 192.168.100.1
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 192.168.100.2-192.168.100.254        |
| cidr                 | 192.168.100.0/24                     |
| created_at           | 2022-03-01T03:03:13Z                 |
| description          |                                      |
| dns_nameservers      |                                      |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | True                                 |
| gateway_ip           | 192.168.100.1                        |
| host_routes          |                                      |
| id                   | fc77e167-5507-43ea-9cca-deb65b7ad937 |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | private-subnet                       |
| network_id           | 48febb12-db06-4ec2-bdc5-df28277688b0 |
| prefix_length        | None                                 |
| project_id           | f99cdd3d3d1344ba858ff4beb80fe624     |
| revision_number      | 0                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2022-03-01T03:03:13Z                 |
+----------------------+--------------------------------------+
root@controller:~# openstack router add subnet router01 private-subnet
root@controller:~# openstack network create \
> --provider-physical-network provider \
> --provider-network-type flat --external public
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2022-03-01T03:03:37Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | dc2a492f-5068-4d55-bbe6-5340e1faf830 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | public                               |
| port_security_enabled     | True                                 |
| project_id                | f99cdd3d3d1344ba858ff4beb80fe624     |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 1                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2022-03-01T03:03:37Z                 |
+---------------------------+--------------------------------------+
root@controller:~# openstack subnet create public-subnet \
> --network public --subnet-range 10.0.0.0/24 \
> --allocation-pool start=10.0.0.200,end=10.0.0.254
+----------------------+--------------------------------------+
| Field                | Value                                |
+----------------------+--------------------------------------+
| allocation_pools     | 10.0.0.200-10.0.0.254                |
| cidr                 | 10.0.0.0/24                          |
| created_at           | 2022-03-01T03:03:49Z                 |
| description          |                                      |
| dns_nameservers      |                                      |
| dns_publish_fixed_ip | None                                 |
| enable_dhcp          | True                                 |
| gateway_ip           | 10.0.0.1                             |
| host_routes          |                                      |
| id                   | d98cafaf-e187-405f-9d48-5604ab971eb9 |
| ip_version           | 4                                    |
| ipv6_address_mode    | None                                 |
| ipv6_ra_mode         | None                                 |
| name                 | public-subnet                        |
| network_id           | dc2a492f-5068-4d55-bbe6-5340e1faf830 |
| prefix_length        | None                                 |
| project_id           | f99cdd3d3d1344ba858ff4beb80fe624     |
| revision_number      | 0                                    |
| segment_id           | None                                 |
| service_types        |                                      |
| subnetpool_id        | None                                 |
| tags                 |                                      |
| updated_at           | 2022-03-01T03:03:49Z                 |
+----------------------+--------------------------------------+
root@controller:~# openstack router set router01 --external-gateway public
```

### Cài đặt dịch vụ Storage (Cinder)


#### Trên node Controller

- Khởi tạo database cho cinder
```sh
root@controller:~# mysql
MariaDB [(none)]> CREATE DATABASE cinder;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \
  IDENTIFIED BY 'lean15998';
MariaDB [(none)]> exit
```

- 	Tạo thông tin xác thực dịch vụ cho cinder

```sh
root@controller:~# openstack user create --domain default --password-prompt cinder
root@controller:~# openstack role add --project service --user cinder admin
root@controller:~# openstack service create --name cinderv2 \
  --description "OpenStack Block Storage" volumev2
root@controller:~# openstack service create --name cinderv3 \
  --description "OpenStack Block Storage" volumev3
root@controller:~# openstack endpoint create --region RegionOne \
  volumev2 public http://controller:8776/v2/%\(project_id\)s
root@controller:~# openstack endpoint create --region RegionOne \
  volumev2 internal http://controller:8776/v2/%\(project_id\)s
root@controller:~# openstack endpoint create --region RegionOne \
  volumev2 admin http://controller:8776/v2/%\(project_id\)s
root@controller:~# openstack endpoint create --region RegionOne \
  volumev3 public http://controller:8776/v3/%\(project_id\)s
root@controller:~# openstack endpoint create --region RegionOne \
  volumev3 internal http://controller:8776/v3/%\(project_id\)s
root@controller:~# openstack endpoint create --region RegionOne \
  volumev3 admin http://controller:8776/v3/%\(project_id\)s
```

- 	Cài đặt dịch vụ cinder
```sh
root@controller:~# apt install cinder-api cinder-scheduler
```

- Chỉnh sửa file cấu hình dịch vụ cinder

```sh
root@controller:~# vim /etc/cinder/cinder.conf

[DEFAULT]
enabled_backends = lvm
transport_url = rabbit://openstack:lean15998@controller
my_ip = 10.0.0.30

[database]
connection = mysql+pymysql://cinder:lean15998@controller/cinder

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = lean15998

[oslo_concurrency]
lock_path = /var/lib/cinder/tmp
```


- Populate vào CSDL cinder

```sh
root@controller:~# su -s /bin/sh -c "cinder-manage db sync" cinder
```

- Cấu hình nova được sử dụng Block storage

```sh
root@controller:~# vim /etc/nova/nova.conf

[cinder]
os_region_name = RegionOne
```

- Khởi động lại dịch vụ

```sh
root@controller:~# service nova-api restart
root@controller:~#  service cinder-scheduler restart
root@controller:~#  service apache2 restart
```


#### Trên node Storage

- Cài đặt lvm2

```sh
root@storage:~# apt install lvm2 thin-provisioning-tools
```

- Khởi tạo Physical volume và volume group

```sh
root@storage:~# pvcreate /dev/sdb
root@storage:~# vgcreate cinder-volumes /dev/sdb
```

- Cấu hình lvm

```sh
root@storage:~# vim /etc/lvm/lvm.conf 

devices {
  filter = [ "a/sdb/", "r/.*/"]
```

- Cài đặt dịch vụ cinder-volume

```sh
root@storage:~# apt install cinder-volume
```

- Cấu hình dịch vụ cinder


```sh
root@storage:~# vim /etc/cinder/cinder.conf

[DEFAULT]
rootwrap_config = /etc/cinder/rootwrap.conf
api_paste_confg = /etc/cinder/api-paste.ini
iscsi_helper = tgtadm
volume_name_template = volume-%s
volume_group = cinder-volumes
verbose = True
auth_strategy = keystone
state_path = /var/lib/cinder
lock_path = /var/lock/cinder
volumes_dir = /var/lib/cinder/volumes
enabled_backends = lvm
transport_url = rabbit://openstack:lean15998@controller
my_ip = 10.0.0.53
glance_api_servers = http://controller:9292

[database]
# ...
connection = mysql+pymysql://cinder:lean15998@controller/cinder

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = cinder
password = lean15998

[lvm]
# ...
volume_driver = cinder.volume.drivers.lvm.LVMVolumeDriver
volume_group = cinder-volumes
target_protocol = iscsi
target_helper = tgtadm

[oslo_concurrency]
# ...
lock_path = /var/lib/cinder/tmp
```

- Khởi động lại dịch vụ

```sh
root@storage:~# service tgt restart
root@storage:~# service cinder-volume restart
```


#### Trên node controller

- Liệt kê các dịch vụ volume

```sh
root@controller:~# openstack volume service list
+------------------+-------------+------+---------+-------+----------------------------+
| Binary           | Host        | Zone | Status  | State | Updated At                 |
+------------------+-------------+------+---------+-------+----------------------------+
| cinder-scheduler | controller  | nova | enabled | up    | 2022-03-01T04:08:44.000000 |
| cinder-volume    | storage@lvm | nova | enabled | up    | 2022-03-01T04:08:39.000000 |
+------------------+-------------+------+---------+-------+----------------------------+
```

### Cài đặt Dashboard (Horizon)
#### Trên node controller

- Cài đặt openstack-dashboard

 ```sh
 root@controller:~# apt -y install openstack-dashboard
 ```
- Cấu hình Openstack dashboard

```sh
root@controller:~# vi /etc/openstack-dashboard/local_settings.py

# line 99: change Memcache server
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '10.0.0.30:11211',
    },
}
SESSION_ENGINE = "django.contrib.sessions.backends.cache"

# line 126: set Openstack Host
# line 127: comment out and add a line to specify URL of Keystone Host
OPENSTACK_HOST = "10.0.0.30"
#OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST
OPENSTACK_KEYSTONE_URL = "http://10.0.0.30:5000/v3"
```

- Khởi động lại dịch vụ apache

```sh
root@controller:~# systemctl restart apache2
```
#### Một số hình ảnh của dashboard

<img src="">
<img src="">
<img src="">
