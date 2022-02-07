# 1. Keystone

## Token

- Kiểm tra token đã được cấp

```
$openstack token issue
```

## Manage

- Khởi tạo domain

```
$openstack domain create domain01
```

- Khởi tạo project

```
$openstack project create --domain domain01 project01
```

- Khởi tạo user

```
$openstack user create --domain domain01 --password-prompt user01
```

- Khởi tạo role

```
$openstack role create role01
```

- Gán role vào user và project

```
$openstack role add --project project01 --user user01 role01
```

# 2. Glance

- Upload image

```
$openstack image create --disk-format qcow2 --container-format bare \
 --file cirros.qcow2 --public cirros-image
```

# 3. Nova Compute

## flavor

- Khởi tạo flavor

```
$openstack flavor create --id 10 --ram 1 --disk 5 --vcpus 2 --public flavor01
```

## Keypair

- Khởi tạo keypair (SSH)

```
$openstack keypair create key01
```

- Import keypair

```
$openstack keypair create --public-key ./ssh/id_rsa.pub key02
```

## Instance

- Khởi tạo vm

```
$openstack server create --flavor cirros256 --image cirros --nic net-id= [id] \
--security-group default --key-name key01 vm01
```

- start,stop,delete,reboot vm

- xem url console

```
$openstack console url show vm01
```

# 4. Neutron

- Khởi tạo router

```
$openstack router create router01
```

## Internal network

- Khởi tạo internal network (Mạng nội bộ)

```
$openstack netwwork create --provide-network-type vxlan private01
```

- Khởi tạo subnet

```
$openstack subnet create private-subnet --network private --subnet-range 10.0.0.0/24 --gateway 10.0.0.1 \
--dns-nameserver 10.0.0.10
```

- Add subnet cho router

```
$openstack router add router01 private-subnet
```

## External network

- Khởi tạo external network ( Mạng bên ngoài)

```
$openstack network create --provider-physical-network provider --provider-network-type flat --external public
```

- Khởi tạo subnet


```
$openstack subnet create public-subnet --network public --subnet-range 10.10.10.0/24 \
allocation-pool start=10.10.10.200,end=10.10.10.254 --gateway 10.10.10.1
```

- Đặt subnet cho router

```
$openstack router set router01 --external-gateway public
```

## floating ip, port

- Khởi tạo floating ip

```
$openstack floating ip create public
```
```
Output: floating_ip_address: 10.10.10.209
```

- Attach floating ip vào vm

```
$openstack server add floating ip vm01 10.10.10.209
```

- Khởi tạo port

```
$openstack port create --network public --fixed-ip subnet=public-subnet,ip-address=10.10.10.214 \
security-group default port01
```

- Attach port vào vm

```
$openstack server add port vm01 port01
```

- Detach port khỏi vm

```
$openstack server remove port vm01 port01
```


## Security group

- Khởi tạo security group

```
$openstack security group create secgroup01
```

- rule

```
$openstack security group rule create --help
```

- Add security group vào port

```
$openstack port set --security-group secgroup01 port01
```



# 5. Cinder

- Khởi tạo volume no-source

```
$openstack volume create --size 1 --description "volume no-soure" volume01
```

- Khởi tạo volume source image

```
$openstack volume create --image cirros --size 1 volume02
```

- Khởi tạo volume source volume

```
$openstack volume create --source [id volume] --size 1 volume03
```

- Attach volume vào vm

```
$openstack server add volume vm01 volume01
```

- Detach volume khỏi vm

```
$openstack server remove volume vm01 volume01
```

- Volume snapshot

```
$openstack volume snapshot create --volume volume01 snapshot01
```


