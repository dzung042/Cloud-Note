## Cấu hình chung cho controller và compute
### Thiết lập Hostname
```sh
hostnamectl set-hostname compute2
```
### Thiết lập IP
```sh
echo "Setup IP  eth0"
nmcli c modify eth0 ipv4.addresses 10.10.10.13/24
nmcli c modify eth0 ipv4.gateway 10.10.10.1
nmcli c modify eth0 ipv4.dns 10.10.10.1
nmcli c modify eth0 ipv4.method manual
nmcli con mod eth0 connection.autoconnect yes

echo "Setup IP eth1"
nmcli c modify eth1 ipv4.addresses 10.10.11.13/24
nmcli c modify eth1 ipv4.method manual
nmcli con mod eth1 connection.autoconnect yes

echo "Setup IP eth2"
nmcli c modify eth2 ipv4.addresses 10.10.12.13/24
nmcli c modify eth2 ipv4.method manual
nmcli con mod eth2 connection.autoconnect yes

echo "Setup IP eth3"
nmcli c modify eth3 ipv4.addresses 10.10.13.13/24
nmcli c modify eth3 ipv4.method manual
nmcli con mod eth3 connection.autoconnect yes

echo "Disable Firewall & SElinux"
sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
```
### Khai báo repo và cài đặt các package cho OpenStack Stein
```sh
echo '[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.3/centos7-amd64
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1' >> /etc/yum.repos.d/MariaDB.repo

yum install -y epel-release
yum update -y
yum install -y centos-release-openstack-queens \
   open-vm-tools python2-PyMySQL vim telnet wget curl 
yum install -y python-openstackclient openstack-selinux 
yum upgrade -y
```
### Setup timezone về Ho_Chi_Minh
```sh
rm -f /etc/localtime
ln -s /usr/share/zoneinfo/Asia/Ho_Chi_Minh /etc/localtime
```
### Reboot Server
```sh
init 6
```
------------------
## Cấu hình NTPD
### Cấu hình trên node controler
```sh
yum install -y chrony
mv /etc/chrony.{conf,conf.bk}
cat << EOF >> /etc/chrony.conf
server 0.vn.pool.ntp.org iburst
allow 10.10.10.0/24
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF
systemctl start chronyd
systemctl enable chronyd
```
#### kiem tra dong bộ
```sh
chronyc sources -V
timedatectl
```

### Cấu hình trên node compute
```sh
yum install -y chrony
mv /etc/chrony.{conf,conf.bk}
cat << EOF >> /etc/chrony.conf
server 10.10.10.11 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF
## Cài đặt MySQL (Cấu hình trên node controler)
yum install mariadb mariadb-server -y 
cat << EOF >> /etc/my.cnf.d/openstack.cnf 
[mysqld]
bind-address = 10.10.10.11
        
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
EOF

systemctl enable mariadb.service
systemctl start mariadb.service
```
### Cài đặt passwd cho MySQL
```sh
mysql_secure_installation <<EOF

y
123456abc
123456abc
y
y
y
y
EOF
```
## Cài đặt RabbitMQ (Chỉ cấu hình trên Node Controller)
```sh
yum install rabbitmq-server -y
```
### Cấu hình cho rabbitmq-server
```sh
rabbitmq-plugins enable rabbitmq_management
systemctl restart rabbitmq-server
curl -O http://localhost:15672/cli/rabbitmqadmin
chmod a+x rabbitmqadmin
mv rabbitmqadmin /usr/sbin/
rabbitmqadmin list users

systemctl start rabbitmq-server
systemctl enable rabbitmq-server
systemctl status rabbitmq-server
```
### Tạo user và gán quyền
```sh
rabbitmqctl add_user openstack 123456abc
rabbitmqctl set_permissions openstack ".*" ".*" ".*"
systemctl enable rabbitmq-server.service
rabbitmqctl set_user_tags openstack administrator
```
### kiểm tra user:
```sh
rabbitmqadmin list users
```
### Đăng nhập vào Dashboard quản trị của Rabbit-mq
```sh
http://10.10.10.11:15672
user: openstack
password: 123456abc
```
## Cài đặt Memcached (Chỉ cấu hình trên Node Controller)
```sh
yum install memcached python-memcached -y 
sed -i "s/-l 127.0.0.1,::1/-l 10.10.10.11/g" /etc/sysconfig/memcached
systemctl enable memcached.service
systemctl start memcached.service
```
# Cài đặt OpenStack
## Cài đặt Keystone (Service Identity) (Chỉ cấu hình trên Node Controller)
```sh
mysql -uroot -p

CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost'  IDENTIFIED BY '123456abc';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY '123456abc';
FLUSH PRIVILEGES;
exit;
```
### cài đặt package
```sh
yum install -y openstack-keystone httpd mod_wsgi
```
### cấu hình
```sh
mv /etc/keystone/keystone.{conf,conf.bk}
cat << EOF >> /etc/keystone/keystone.conf
[DEFAULT]
[application_credential]
[assignment]
[auth]
[cache]
memcache_servers = 10.10.10.11:11211
[catalog]
[cors]
[credential]
[database]
connection = mysql+pymysql://keystone:123456abc@10.10.10.11/keystone
[domain_config]
[endpoint_filter]
[endpoint_policy]
[eventlet_server]
[federation]
[fernet_tokens]
[healthcheck]
[identity]
[identity_mapping]
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
[unified_limit]
EOF
```
### phân quyền 
```sh
chown root:keystone /etc/keystone/keystone.conf
```
### đồng bộ database
```sh
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
### Thiết lập Fernet key
```sh
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```
### Thiết lập boostrap cho Keystone
```sh
keystone-manage bootstrap --bootstrap-password 123456abc \
--bootstrap-admin-url http://10.10.10.11:5000/v3/ \
--bootstrap-internal-url http://10.10.10.11:5000/v3/ \
--bootstrap-public-url http://10.10.10.11:5000/v3/ \
--bootstrap-region-id RegionOne
```
###  cấu hình apache
```sh
sed -i 's|#ServerName www.example.com:80|ServerName 10.10.10.11|g' /etc/httpd/conf/httpd.conf 
ln -s /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/
ls /etc/httpd/conf.d/
systemctl enable httpd.service
systemctl restart httpd.service
systemctl status httpd.service
```
### Tạo file biến môi trường openrc-admin cho tài khoản quản trị
```sh
cat << EOF >> admin-openrc
export export OS_REGION_NAME=RegionOne
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=123456abc
export OS_AUTH_URL=http://10.10.10.11:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='[\u@\h \W(admin-openrc-r1)]\$ '
EOF
```
### Tạo file biến môi trường openrc-demo cho tài khoản demo
```sh
cat << EOF >> demo-openrc
export export OS_REGION_NAME=RegionOne
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=123456abc
export OS_AUTH_URL=http://10.10.10.11:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
export PS1='[\u@\h \W(demo-openrc-r1)]\$ '
EOF
```
### Tạo domain, projects, users, và roles
```sh
source admin-openrc 
openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" demo
openstack user create --domain default --password 123456abc demo
openstack role create user
openstack role add --project demo --user demo user
```
#### Unset các biến môi trường
```sh
unset OS_AUTH_URL OS_PASSWORD
```
### Kiểm tra xác thực trên Project admin
```sh
openstack --os-auth-url http://10.10.10.11:5000/v3 --os-project-domain-name Default \
--os-user-domain-name Default --os-project-name admin --os-username admin token issue
```
### kiểm tra xác thực trên Project demo.
```sh
openstack --os-auth-url http://10.10.10.11:5000/v3 --os-project-domain-name default \
--os-user-domain-name default --os-project-name demo --os-username demo token issue
```
### Sau khi kiểm tra xác thực xong source lại biến môi trường
```sh
source admin-openrc 
```
### Nếu trong quá trình thao tác, xác thực token có vấn đề, get lại token mới
```sh
openstack token issue
```
## Cài đặt Glance (Service Images) (Chỉ cấu hình trên Node Controller)
### Create DB, user và các endpoint cho Glance
```sh
mysql -uroot -p

CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost'  IDENTIFIED BY '123456abc';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY '123456abc';
FLUSH PRIVILEGES;
exit;
```
#### sử dụng biến môi trường
```sh
source admin-openrc
```
### Tạo glance user
```sh
openstack user create --domain default --password 123456abc glance
```
#### Thêm roles admin cho user glance trên project service
```sh
openstack role add --project service --user glance admin
```
#### kiểm tra lại user glance
```sh
openstack role list --user glance --project service
```
### khởi tạo dịch vụ glance
```sh
openstack service create --name glance --description "OpenStack Image" image
```
#### Tạo các enpoint cho glane
```sh
openstack endpoint create --region RegionOne image public http://10.10.10.11:9292
openstack endpoint create --region RegionOne image internal http://10.10.10.11:9292
openstack endpoint create --region RegionOne image admin http://10.10.10.11:9292
```
### Cài đặt cấu hình Glance
```sh
yum install -y openstack-glance
mv /etc/glance/glance-api.{conf,conf.bk}
cat << EOF >> /etc/glance/glance-api.conf
[DEFAULT]
bind_host = 10.10.10.11
registry_host = 10.10.10.11
[cinder]
[cors]
[database]
connection = mysql+pymysql://glance:123456abc@10.10.10.11/glance
[file]
[glance.store.http.store]
[glance.store.rbd.store]
[glance.store.sheepdog.store]
[glance.store.swift.store]
[glance.store.vmware_datastore.store]
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
[image_format]
[keystone_authtoken]
www_authenticate_uri = http://10.10.10.11:5000
auth_url = http://10.10.10.11:5000
memcached_servers = 10.10.10.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = glance
password = 123456abc
region_name = RegionOne
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]
EOF
```
#### phần quyền 
```sh
chown root:glance /etc/glance/glance-api.conf
```
#### Cấu hình glance-registry
```sh
mv /etc/glance/glance-registry.{conf,conf.bk}
cat << EOF >> /etc/glance/glance-registry.conf
[DEFAULT]
bind_host = 10.10.10.11
[database]
connection = mysql+pymysql://glance:123456abc@10.10.10.11/glance
[keystone_authtoken]
www_authenticate_uri = http://10.10.10.11:5000
auth_url = http://10.10.10.11:5000
memcached_servers = 10.10.10.11
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = 123456abc
region_name = RegionOne
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
EOF
chown root:glance /etc/glance/glance-registry.conf
su -s /bin/sh -c "glance-manage db_sync" glance
```
### Enable và restart Glance
```sh
systemctl enable openstack-glance-api.service openstack-glance-registry.service
systemctl start openstack-glance-api.service openstack-glance-registry.service
```
### test tạo image:
```sh
wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img
openstack image create "cirros" --file cirros-0.3.5-x86_64-disk.img \
--disk-format qcow2 --container-format bare --public
openstack image list
```
# Cài đặt Nova (Service Compute)
## Cấu hình trên Node Controller
```sh
mysql -u root -p

CREATE DATABASE nova;
CREATE DATABASE nova_api;
CREATE DATABASE nova_cell0;
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost'  IDENTIFIED BY '123456abc';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY '123456abc';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost'  IDENTIFIED BY '123456abc';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY '123456abc';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost'  IDENTIFIED BY '123456abc';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY '123456abc';
FLUSH PRIVILEGES;
exit;
```
#### sử dụng biến môi trường
```sh
source admin-openrc
```
#### Tạo user nova
```sh
openstack user create --domain default --password 123456abc nova
```
#### Thêm role admin cho user placement trên project service
```sh
openstack role add --project service --user nova admin
```
#### Tạo dịch vụ nova
```sh
openstack service create --name nova --description "OpenStack Compute" compute
```
#### Tạo các endpoint cho dịch vụ compute
```sh
openstack endpoint create --region RegionOne compute public http://10.10.10.11:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://10.10.10.11:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://10.10.10.11:8774/v2.1
```
#### Tạo user placement
```sh
openstack user create --domain default --password 123456abc placement
```
#### Thêm role admin cho user placement trên project service
```sh
openstack role add --project service --user placement admin
```
#### Tạo dịch vụ placement
```sh
openstack service create --name placement --description "Placement API" placement
```
#### Tạo endpoint cho placement
```sh
openstack endpoint create --region RegionOne placement public http://10.10.10.11:8778
openstack endpoint create --region RegionOne placement internal http://10.10.10.11:8778
openstack endpoint create --region RegionOne placement admin http://10.10.10.11:8778
```
### Cài đặt cấu hình Nova
```sh
yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-console \
openstack-nova-novncproxy openstack-nova-scheduler openstack-nova-placement-api
```
### Cấu hình nova
```sh
mv /etc/nova/nova.{conf,conf.bk}

cat << EOF >> /etc/nova/nova.conf
[DEFAULT]
# define own IP
my_ip = 10.10.10.11
enabled_apis = osapi_compute,metadata
use_neutron = True
osapi_compute_listen=10.10.10.11
metadata_host=10.10.10.11
metadata_listen=10.10.10.11
metadata_listen_port=8775    
firewall_driver = nova.virt.firewall.NoopFirewallDriver
allow_resize_to_same_host=True
#notify_on_state_change = vm_and_task_state
#state_path = /var/lib/nova
#log_dir = /var/log/nova
# RabbitMQ connection info
transport_url = rabbit://openstack:123456abc@10.10.10.11:5672
[api]
auth_strategy = keystone
# MariaDB connection info
[api_database]
connection = mysql+pymysql://nova:123456abc@10.10.10.11/nova_api
[barbican]
[cache]
backend = oslo_cache.memcache_pool
enabled = true
memcache_servers = 10.10.10.11:11211
[cells]
[cinder]
os_region_name = RegionOne
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
connection = mysql+pymysql://nova:123456abc@10.10.10.11/nova
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
# Glance connection info
[glance]
api_servers = http://10.10.10.11:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
www_authenticate_uri = http://10.10.10.11:5000
auth_url = http://10.10.10.11:5000/v3
memcached_servers = 10.10.10.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = 123456abc
region_name = RegionOne
[libvirt]
[metrics]
[mks]
[neutron]
region_name = RegionOne
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
rabbit_ha_queues = true
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues= true
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
auth_url = http://10.10.10.11:5000/v3
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = placement
password = 123456abc
region_name = RegionOne
[placement_database]
connection = mysql+pymysql://nova:123456abc@10.10.10.11/nova_placement
[powervm]
[privsep]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
discover_hosts_in_cells_interval = 300
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
# add follows (enable VNC)
[vnc]
enabled = True
server_listen = 10.10.10.11
server_proxyclient_address = 10.10.10.11
novncproxy_base_url = http://10.10.10.11:6080/vnc_auto.html 
[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
EOF

chown root:nova /etc/nova/nova.conf
cp /etc/httpd/conf.d/00-nova-placement-api.{conf,conf.bk}
cat << 'EOF' >> /etc/httpd/conf.d/00-nova-placement-api.conf


<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
EOF
```
#### Cấu hình bind cho nova placement api trên httpd
```sh
sed -i -e 's/VirtualHost \*/VirtualHost 10.10.10.11/g' /etc/httpd/conf.d/00-nova-placement-api.conf
sed -i -e 's/Listen 8778/Listen 10.10.10.11:8778/g' /etc/httpd/conf.d/00-nova-placement-api.conf
```
#### 
```sh
systemctl restart httpd 
```
#### Import DB nova
```sh
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova 
su -s /bin/sh -c "nova-manage db sync" nova
```
#### Check nova cell
```sh
nova-manage cell_v2 list_cells
```
#### Enable và start service nova
```sh
systemctl enable openstack-nova-api.service openstack-nova-consoleauth.service \
openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service
systemctl start openstack-nova-api.service openstack-nova-consoleauth.service \
openstack-nova-scheduler.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service
```
#### Kiểm tra cài đặt lại dịch vụ
```sh
openstack compute service list
```
## Cấu hình trên Node Compute1
```sh
yum install -y openstack-nova-compute libvirt-client
```
### Backup cấu hình nova
```sh
mv /etc/nova/nova.{conf,conf.bk}
```
### Cấu hình Nova
```sh
cat << EOF >> /etc/nova/nova.conf 
[DEFAULT]
enabled_apis = osapi_compute,metadata
transport_url = rabbit://openstack:123456abc@10.10.10.11:5672
my_ip = 10.10.10.12
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
[barbican]
[cache]
[cells]
[cinder]
os_region_name = RegionOne
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[crypto]
[database]
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://10.10.10.11:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://10.10.10.11:5000/v3
memcached_servers = 10.10.10.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = 123456abc
region_name = RegionOne
[libvirt]
# egrep -c '(vmx|svm)' /proc/cpuinfo = 0
virt_type = qemu
#virt_type = kvm
#cpu_mode = host-passthrough
#hw_disk_discard = unmap
[matchmaker_redis]
[metrics]
[mks]
[neutron]
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
#driver = messagingv2
[oslo_messaging_rabbit]
rabbit_ha_queues = true
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues= true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://10.10.10.11:5000/v3
username = placement
password = 123456abc
region_name = RegionOne
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = 10.10.10.12
novncproxy_base_url = http://10.10.10.11:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
EOF
```
### Phân quyền lại file config
```sh
chown root:nova /etc/nova/nova.conf
```
### Enable và start service
```sh
systemctl enable libvirtd.service openstack-nova-compute.service
systemctl start libvirtd.service openstack-nova-compute.service
```
## Kiểm tra trên Controller
```sh
source admin-openrc
openstack compute service list
```
# Cài đặt Neutron (Service Network)
## Cấu hình trên Node Controller
```sh
mysql -uroot -p

CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY '123456abc';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY '123456abc';
FLUSH PRIVILEGES;
exit;
```
### Tạo user 
```sh
openstack user create --domain default --password 123456abc neutron
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network
```
### Tao endpoint
```sh
openstack endpoint create --region RegionOne network public http://10.10.10.11:9696
openstack endpoint create --region RegionOne network internal http://10.10.10.11:9696
openstack endpoint create --region RegionOne network admin http://10.10.10.11:9696
```
### Cài đặt cấu hình neutron
```sh
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y
```
#### Cấu hình
```sh
mv /etc/neutron/neutron.{conf,conf.bk}

cat << EOF >> /etc/neutron/neutron.conf
[DEFAULT]
bind_host = 10.10.10.11
core_plugin = ml2
service_plugins = router
transport_url = rabbit://openstack:123456abc@10.10.10.11
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
allow_overlapping_ips = True
dhcp_agents_per_network = 2
[agent]
[cors]
[database]
connection = mysql+pymysql://neutron:123456abc@10.10.10.11/neutron
[keystone_authtoken]
www_authenticate_uri = http://10.10.10.11:5000
auth_url = http://10.10.10.11:5000
memcached_servers = 10.10.10.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 123456abc
region_name = RegionOne
[matchmaker_redis]
[nova]
auth_url = http://10.10.10.11:5000
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = 123456abc
region_name = RegionOne
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
#driver = messagingv2
[oslo_messaging_rabbit]
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues = true
rabbit_ha_queues = true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[quotas]
[ssl]
EOF
```
#### phân quyền
```sh
chown root:neutron /etc/neutron/neutron.conf
```
#### Cấu hình ml2_config
```sh
mv /etc/neutron/plugins/ml2/ml2_conf.{ini,ini.bk}

cat << EOF >> /etc/neutron/plugins/ml2/ml2_conf.ini
[DEFAULT]
[l2pop]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
# mechanism_drivers = linuxbridge,l2population
mechanism_drivers = linuxbridge
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
# network_vlan_ranges = provider
[ml2_type_vxlan]
vni_ranges = 1:1000
[securitygroup]
enable_ipset = true
EOF
```
#### phân quyền
```sh
chown root:neutron /etc/neutron/plugins/ml2/ml2_conf.ini
```
#### Cấu hình linuxbridge_agent
```sh
mv /etc/neutron/plugins/ml2/linuxbridge_agent.{ini,init.bk}

cat << EOF >> /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[DEFAULT]
[agent]
[linux_bridge]
physical_interface_mappings = provider:eth1
[network_log]
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
[vxlan]
# enable_vxlan = true
## network dataVM
local_ip = 10.10.12.11
# l2_population = true
EOF
```
#### phân quyền
```sh
chown root:neutron /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```
#### Cấu hình trên file l3_agent
```sh
mv /etc/neutron/l3_agent.{ini,ini.bk}

cat << EOF >> /etc/neutron/l3_agent.ini
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
[agent]
[ovs]
EOF

chown root:neutron /etc/neutron/l3_agent.ini
```
#### Bổ xung cấu hình nova service trên controller sử dụng networking service
##### Chỉnh sửa bổ sung cấu hình [neutron] trong file /etc/nova/nova.conf
```sh
[neutron]
endpoint_override = http://10.10.10.11:9696
auth_url = http://10.10.10.11:5000
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 123456abc
service_metadata_proxy = true
metadata_proxy_shared_secret = 123456abc
region_name = RegionOne
```
#### Các Networking service initialization script yêu cầu symbolic link /etc/neutron/plugin.ini tới ML2 plug-in config file /etc/neutron/plugins/ml2/ml2_conf.ini
#### Khởi tạo symlink cho ml2_config
```sh
ln -s /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```
#### Đồng bộ database
```sh
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
--config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```
#### Khởi động lại Compute API service:
```sh
systemctl restart openstack-nova-api.service openstack-nova-scheduler.service \
openstack-nova-consoleauth.service openstack-nova-conductor.service \
openstack-nova-novncproxy.service
```
#### Enable và restart Neutron service
```sh
systemctl enable neutron-server.service neutron-linuxbridge-agent.service \
neutron-l3-agent.service

systemctl start neutron-server.service neutron-linuxbridge-agent.service \
neutron-l3-agent.service
```
##Cấu hình trên Node Compute
```sh
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables -y

mv /etc/neutron/neutron.{conf,conf.bk}

cat << EOF >> /etc/neutron/neutron.conf
[DEFAULT]
transport_url = rabbit://openstack:123456abc@10.10.10.11:5672
auth_strategy = keystone
[agent]
[cors]
[database]
[keystone_authtoken]
www_authenticate_uri = http://10.10.10.11:5000
auth_url = http://10.10.10.11:5000
memcached_servers = 10.10.10.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 123456abc
region_name = RegionOne
[matchmaker_redis]
[nova]
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
#driver = messagingv2
[oslo_messaging_rabbit]
rabbit_ha_queues = true
rabbit_retry_interval = 1
rabbit_retry_backoff = 2
amqp_durable_queues= true
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[quotas]
[ssl]
EOF

chown root:neutron /etc/neutron/neutron.conf
mv /etc/neutron/plugins/ml2/linuxbridge_agent.{ini,ini.bk}

cat << EOF >> /etc/neutron/plugins/ml2/linuxbridge_agent.ini
[DEFAULT]
[agent]
[linux_bridge]
physical_interface_mappings = provider:eth1
[network_log]
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
[vxlan]
# enable_vxlan = true
## network dataVM
local_ip = 10.10.12.13
# l2_population = true
EOF

chown root:neutron /etc/neutron/plugins/ml2/linuxbridge_agent.ini
mv /etc/neutron/dhcp_agent.{ini,ini.bk}
cat << EOF >> /etc/neutron/dhcp_agent.ini
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
force_metadata = True
[agent]
[ovs]
EOF

mv /etc/neutron/metadata_agent.{ini,ini.bk}
cat << EOF >> /etc/neutron/metadata_agent.ini
[DEFAULT]
nova_metadata_host = 10.10.10.11
metadata_proxy_shared_secret = 123456abc
[agent]
[cache]
EOF
```
```sh
chown root:neutron /etc/neutron/metadata_agent.ini
```
#### Bổ sung cấu hình phần [neutron] trong /etc/nova/nova.conf
```sh
[neutron]
endpoint_override = http://10.10.10.11:9696
auth_url = http://10.10.10.11:5000
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = 123456abc
region_name = RegionOne

systemctl restart openstack-nova-compute.service libvirtd.service 
systemctl enable neutron-linuxbridge-agent.service \
neutron-dhcp-agent.service neutron-metadata-agent.service

systemctl start neutron-linuxbridge-agent.service \
neutron-dhcp-agent.service neutron-metadata-agent.service
```
## Cài đặt Cinder (Service Storage) (Chỉ cấu hình trên Node Controller)
```sh
mysql -u root -p

CREATE DATABASE cinder;
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY '123456abc';
GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY '123456abc';
exit
```
### sử dụng biến môi trường 
```sh
source admin-openrc

openstack user create --domain default --password 123456abc cinder
openstack role add --project service --user cinder admin
openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
```
### Tạo Block Storage service API endpoints:
```sh
openstack endpoint create --region RegionOne volumev2 \
public http://10.10.10.11:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 \
internal http://10.10.10.11:8776/v2/%\(project_id\)s
openstack endpoint create --region RegionOne volumev2 \
admin http://10.10.10.11:8776/v2/%\(project_id\)s

openstack endpoint create --region RegionOne volumev3 \
public http://10.10.10.11:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 \
internal http://10.10.10.11:8776/v3/%\(project_id\)s
openstack endpoint create --region RegionOne volumev3 \
admin http://10.10.10.11:8776/v3/%\(project_id\)s
```
### cài đặt package
```sh
yum install -y lvm2 device-mapper-persistent-data openstack-cinder targetcli
systemctl enable lvm2-lvmetad.service
systemctl start lvm2-lvmetad.service
```
### chờ cấu hình connect ceph
## Cài đặt Horizon (Service Dashboard) (Chỉ cấu hình trên Node Controller)
```sh
yum install -y openstack-dashboard
```
### tạo file redrect
```sh
filehtml=/var/www/html/index.html
touch $filehtml
cat << EOF >> $filehtml
<html>
<head>
<META HTTP-EQUIV="Refresh" Content="0.5; URL=http://10.10.10.11/dashboard">
</head>
<body>
<center> <h1>Redirecting to OpenStack Dashboard</h1> </center>
</body>
</html>
EOF

cp /etc/openstack-dashboard/{local_settings,local_settings.bk}
```
### sửa file cấu hình
```sh
vi /etc/openstack-dashboard/local_settings
 
################## 
# line 38
ALLOWED_HOSTS = ['*',]
OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'Default'
    
# line 171 add
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': ['10.10.10.11:11211',]
    }
}

#line 205
OPENSTACK_HOST = "10.10.10.11"
OPENSTACK_KEYSTONE_URL = "http://10.10.10.11:5000/v3"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

#line 481
TIME_ZONE = "Asia/Ho_Chi_Minh"
```
#############
```sh
echo "WSGIApplicationGroup %{GLOBAL}" >> /etc/httpd/conf.d/openstack-dashboard.conf
systemctl restart httpd.service memcached.service
```
### Hoàn tất cài đặt truy cập Dashboard
```sh
http://10.10.10.11
user: admin
password: 123456abc
```
### Tạo network
### Truy cập Admin --> Network --> Networks Chọn Create Network
Name: provider
Project: admin
Provider Network Type: flat
Physical Network: provider
Tick all --> next
Subnet Name: name subnet (sub11_provider)
Network Address : 10.10.11.0/24
Gate way: 10.10.11.1
Next
Allocation Pools: 10.10.11.20,10.10.11.250
