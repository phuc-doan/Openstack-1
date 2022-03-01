# Mô hình triển khai
```sh
   ------------+---------------------------+---------------------------+---------------------------+------------
               |                           |                           |                           |
          ens38|10.0.0.30             ens38|10.0.0.50             ens38|10.0.0.51             ens38| 10.0.0.52
   +-----------+-----------+   +-----------+-----------+   +-----------+-----------+   +-----------+-----------+
   |    [ Control Node ]   |   |    [ Network Node ]   |   |    [ Compute Node ]   |   |    [ Storage Node ]   |
   |                       |   |                       |   |                       |   |                       |
   |  MariaDB    RabbitMQ  |   |        L2 Agent       |   |        Libvirt        |   |     Cinder-volume     |
   |  Memcached  Apache2   |   |        L3 Agent       |   |     Nova Compute      |   |                       |
   |  Keystone   Glance    |   |     Metadata Agent    |   |        L2 Agent       |   |                       |
   | Placement API Nova API|   |                       |   |                       |   |                       |
   |  Neutron Server       |   |                       |   |                       |   |                       |
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
