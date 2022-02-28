# Mô hình triển khai
```sh
   ------------+---------------------------+---------------------------+---------------------------+
               |                           |                           |                           |
           eth0|10.0.0.30              eth0|10.0.0.50              eth0|10.0.0.51              eth0| 10.0.0.52
   +-----------+-----------+   +-----------+-----------+   +-----------+-----------+   +-----------+-----------+
   |    [ Control Node ]   |   |    [ Network Node ]   |   |    [ Compute Node ]   |   |    [ Storage Node ]   |
   |                       |   |                       |   |                       |   |                       |
   |  MariaDB    RabbitMQ  |   |        L2 Agent       |   |        Libvirt        |   |     Cinder-volume     |
   |  Memcached  Apache2   |   |        L3 Agent       |   |     Nova Compute      |   |                       |
   |  Keystone   Glance    |   |     Metadata Agent    |   |        L2 Agent       |   |                       |
   |  Nova API             |   |                       |   |                       |   |                       |
   |  Neutron Server       |   |                       |   |                       |   |                       |
   |  Metadata Agent       |   |                       |   |                       |   |                       | 
   +-----------------------+   +-----------------------+   +-----------------------+   +-----------------------+
```

# Cài đặt dịch vụ bổ trợ cho Openstack
## Trên node Controller
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
