#############
Các thao tác được thực hiện với tài khoản root.
Chạy các lệnh dưới ngay sau khi cài đặt xong máy ảo.
Mô hình phải đảm bảo cấu hình đúng dải IP ở trên.
Phiên bản hệ điều hành cho các máy là Ubuntu Server 14.04-x 64 bit
root@controller:~# lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 14.04.3 LTS
Release:        14.04
Codename:       trusty
1.	THIẾT LẬP MÔI TRƯỜNG CHO CÁC NODE
Thiết lập IP, hostname
	Bạn có thể sửa IP phù hợp với máy bạn, tốt nhất là sử dụng theo IP mà chúng tôi hướng dẫn.
	Thiết lập hostname với tên là controller
echo "controller" > /etc/hostname
hostname -F /etc/hostname
	Thiết lập địa chỉ IP

	Sao lưu file cấu hình của card mạng
cp /etc/network/interfaces /etc/network/interfaces.bak 

	Sử dụng script dưới để cấu hình IP tĩnh cho card mạng.
cat << EOF > /etc/network/interfaces

# NIC loopback
auto lo
iface lo inet loopback

# NIC MGNG
auto eth0
iface eth0 inet static
address 10.10.10.120
netmask 255.255.255.0

# NIC EXTERNAL
auto eth1
iface eth1 inet static
address 202.92.4.120
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 8.8.8.8


EOF

	Cấu hình file /etc/hosts để phân giản IP cho các node

cat << EOF > /etc/hosts 
127.0.0.1   controller localhost
10.10.10.120    controller
10.10.10.121    compute1

EOF


Cài đặt NTP server
	Chạy lệnh dưới để cài đặt NTP server thực hiện trên controller
apt-get -y install chrony
	Chỉnh sửa file vi /etc/chrony/chrony.conf
server 2.vn.pool.ntp.org iburst
server 3.asia.pool.ntp.org iburst
server 2.asia.pool.ntp.org iburst
	Restart lại dịch vụ
service chrony restart
	Khai báo các gói để cài đặt OpenStack Liberty
apt-get -y install software-properties-common
add-apt-repository -y cloud-archive:mitaka
	
	Update và khởi động lại node controller
apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y && init 6
	Cài đăt các gói phần mềm
Cài đặt gói OpenStack Client
apt-get install -y python-openstackclient
2.	CÀI ĐẶT TRÊN NODE CONTROLLER	
a.	Cài đặt mysql 
	Trong quá trình cài đặt yêu cầu nhập mật khẩu My SQL, sử dụng mật khẩu Cloud1o1iNET để thống nhất.
apt-get -y install mariadb-server python-pymysql
	Tạo file với nội dung sau
cat << EOF  > /etc/mysql/conf.d/mysqld_openstack.cnf
[mysqld]
bind-address = controller
[mysqld]
default-storage-engine = innodb
innodb_file_per_table
collation-server = utf8_general_ci
init-connect = 'SET NAMES utf8'
character-set-server = utf8
EOF
	Khởi động lại MYSQL
service mysql restart
	Cài đặt Message queue
b.	Cài đặt gói rabbitmq
apt-get -y install rabbitmq-server
	Tạo tài khoản openstack cho rabbitmq
rabbitmqctl add_user openstack Cloud1o1iNET
	Cấp quyền cho tài khoản openstack
rabbitmqctl set_permissions openstack ".*" ".*" ".*"

c.	Cài đặt memcache
apt-get -y install memcached python-memcache
	Mở file cấu hình vi /etc/memcached.conf thay đổi dòng -l 127.0.0.1 thành dòng dưới
-l 10.60.0.220
	Restart memcache
service memcached restart

	Cài đặt dịch vụ Keystone
	Tạo database cho keystone
	Đăng nhập vào MariaDB
mysql -u root -pCloud1o1iNET
	Tạo DB tên là keystone và gán quyền
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'Cloud1o1iNET';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'Cloud1o1iNET';
flush privileges;
quit
d.	Cài đặt Keystone
	Cấu hình không cho Keystone tự động khởi động.
echo "manual" > /etc/init/keystone.override
	Cài đặt các gói dành cho Keystone
apt-get -y install keystone apache2 libapache2-mod-wsgi
	Sao lưu file cấu hình của keystone.
mv /etc/keystone/keystone.conf /etc/keystone/keystone.conf.bak
	Tạo file keystone mới bằng lệnh vi /etc/keystone/keystone.conf chứa nội dung dưới.
[DEFAULT]
log_dir = /var/log/keystone
admin_token = Cloud1o1iNET
public_bind_host = controller
admin_bind_host = controller
[assignment]
[auth]
[cache]
[catalog]
[cors]
[cors.subdomain]
[credential]
[database]
connection = mysql+pymysql://keystone:Cloud1o1iNET@controller/keystone
[domain_config]
[endpoint_filter]
[endpoint_policy]
[eventlet_server]
[eventlet_server_ssl]
[federation]
[fernet_tokens]
[identity]
[identity_mapping]
[kvs]
[ldap]
[matchmaker_redis]
[matchmaker_ring]
[memcache]
servers = localhost:11211
[oauth1]
[os_inherit]
[oslo_messaging_amqp]
[oslo_messaging_qpid]
[oslo_messaging_rabbit]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
[policy]
[resource]
[revoke]
driver = sql
[role]
[saml]
[signing]
[ssl]
[token]
provider = uuid
driver = memcache
[tokenless_auth]
[trust]
[extra_headers]
Distribution = Ubuntu
	Đồng bộ database cho keystone
su -s /bin/sh -c "keystone-manage db_sync" keystone
	Cấu hình apache cho Keystone
	Thêm dòng dưới vào /etc/apache2/apache2.conf sau dòng Global configuration
# Global configuration
ServerName controller

	Tạo file /etc/apache2/sites-available/wsgi-keystone.conf với nội dung sau
	
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
	ErrorLog /var/log/apache2/keystone.log
	CustomLog /var/log/apache2/keystone_access.log combined

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
	ErrorLog /var/log/apache2/keystone.log
	CustomLog /var/log/apache2/keystone_access.log combined

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
	Cấu hình virtual host cho keystone
ln -s /etc/apache2/sites-available/wsgi-keystone.conf /etc/apache2/sites-enabled
	Khởi động lại apache
service apache2 restart
	Xóa SQLite mặc định của keystone
rm -f /var/lib/keystone/keystone.db

	Khai báo biến môi trường để cài đặt KeyStone

export OS_TOKEN=Cloud1o1iNET
export OS_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3

	Tạo user, endpoint, role, tenant cho Keystone

openstack service create --name keystone --description "OpenStack Identity" identity

openstack endpoint create --region RegionOne identity public http://controller:5000/v3
openstack endpoint create --region RegionOne identity internal http://controller:5000/v3
openstack endpoint create --region RegionOne identity admin http://controller:35357/v3

	Tạo domain
openstack domain create --description "Default Domain" default
	Tạo admin project
openstack project create --domain default --description "Admin Project" admin
	Tạo user admin
openstack user create admin --domain default --password Cloud1o1iNET 
	Tạo role admin
openstack role create admin
	Gán user admin vào role admin thuộc project admin
openstack role add --project admin --user admin admin
	Tạo project có tên là service để chứa các user service của openstack
openstack project create --domain default --description "Service Project" service
	Tạo project tên là demo
openstack project create --domain default --description "Demo Project" demo
	Tạo user tên là demo
openstack user create demo --domain default --password Cloud1o1iNET
	Tạo role tên là user
openstack role create user
	Gán tài khoản demo có role là user vào project demo
openstack role add --project demo --user demo user
	Kiểm chứng lại việc cài đặt KeyStone
	
	Disable chuỗi chứng thực token tạm thời. Mở file /etc/keystone/keystone-paste.ini, sau đó xóa bỏ dòng admin_token_auth từ các session [pipeline:public_api], [pipeline:admin_api], and [pipeline:api_v3] 
		Hủy 02 biến môi trường đã khai báo trước đó
unset OS_TOKEN OS_URL
	Gõ lần lượt 2 lệnh dưới để nhận kết quả trả về
openstack --os-auth-url http://controller:35357/v3 \
--os-project-domain-name default --os-user-domain-name default \
--os-project-name admin --os-username admin token issue
	và 
openstack --os-auth-url http://controller:5000/v3 \
--os-project-domain-name default --os-user-domain-name default \
--os-project-name demo --os-username demo token issue

	Tạo file admin-openrc với nội dung dưới bằng lệnh vi admin-openrc
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=Cloud1o1iNET
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
	Tạo file demo-openrc với nội dung dưới bằng lệnh vi demo-openrc
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=Cloud1o1iNET
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
	Phân quyền cho file admin-openrc và demo-openrc
chmod +x admin-openrc
chmod +x demo-openrc
	Chạy lênh dưới để khai báo biến môi trường
source admin-openrc
	Kiểm tra xem keystone hoạt động tốt hay chưa bằng lệnh
openstack token issue
	Kết quả tương tự như dưới
	+------------+----------------------------------+
	| Field      | Value                            |
	+------------+----------------------------------+
	| expires    | 2015-11-17T09:53:40.242778Z      |
	| id         | de796ac24b2545efb99487d9ff4e981a |
	| project_id | c685a5fa3e474261b678aeb59332ce0d |
	| user_id    | 818e335d15484101b6a2a69e5f9d4f61 |
	+------------+----------------------------------+
		
e.	Cài đặt GLANCE

Glance chỉ cần cài đặt trên Controller
Tạo database và phân quyền bằng các lệnh dưới
	
mysql -u root -pCloud1o1iNET

CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'Cloud1o1iNET';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'Cloud1o1iNET';
quit;

	Tạo user, endpoint, gán role cho glance trong keystone
openstack user create glance --domain default --password Cloud1o1iNET 
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image service" image

openstack endpoint create --region RegionOne image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne image admin http://controller:9292

	Cài đặt các gói trong glance
apt-get -y install glance
	Sao lưu file cấu hình gốc của glance
mv /etc/glance/glance-api.conf /etc/glance/glance-api.conf.bak
	Tạo file glance-api.conf với bằng lệnh vi /etc/glance/glance-api.conf với nội dung sau
	
[DEFAULT]
notification_driver = noop

[cors]
[cors.subdomain]
[database]
backend = sqlalchemy
connection = mysql+pymysql://glance:Cloud1o1iNET@controller/glance

[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

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
password = Cloud1o1iNET

[matchmaker_redis]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_policy]
[paste_deploy]
flavor = keystone

[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]

	Sao lưu file /etc/glance/glance-registry.conf
mv /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf.bak

	Tạo file mới bằng lệnh vi /etc/glance/glance-registry.conf với nội dung sau:

	
	[DEFAULT]
notification_driver = noop

[database]
backend = sqlalchemy
connection = mysql+pymysql://glance:Cloud1o1iNET@controller/glance

[glance_store]
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = glance
password = Cloud1o1iNET

[matchmaker_redis]
[oslo_messaging_amqp]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_policy]
[paste_deploy]
flavor = keystone

[profiler]

	Đồng bộ database cho Glance
su -s /bin/sh -c "glance-manage db_sync" glance
	Xóa file SQLite mặc định
rm -f /var/lib/glance/glance.sqlite
	Khởi động lại Glance
service glance-api restart
service glance-registry restart
	Khai báo thêm biến môi trường cho Glance
echo "export OS_IMAGE_API_VERSION=2" | tee -a admin.sh
source admin.sh
	Tải image cirros và up image cho glance
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

glance image-create --name "cirros" \
--file cirros-0.3.4-x86_64-disk.img \
--disk-format qcow2 --container-format bare \
--visibility public --progress
	d. Cài đặt NOVA
	Cài đặt NOVA trên CONTROLLER

	Tạo database cho NOVA
mysql -u root -pCloud1o1iNET

CREATE DATABASE nova_api;
CREATE DATABASE nova;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'Cloud1o1iNET';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'Cloud1o1iNET';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'Cloud1o1iNET';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'Cloud1o1iNET';

FLUSH PRIVILEGES;

exit;
	Tạo user, gán role, endpoint cho nova
openstack user create nova --domain default --password Cloud1o1iNET
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute

openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1/%\(tenant_id\)s
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1/%\(tenant_id\)s

	Cài đặt các gói nova trên Controller

apt-get -y install nova-api nova-cert nova-conductor nova-consoleauth nova-novncproxy nova-scheduler
	Sao lưu file /etc/nova/nova.conf
mv /etc/nova/nova.conf  /etc/nova/nova.conf.bak

	Tạo file nova.conf với lệnh vi /etc/nova/nova.conf với nội dung sau:
	
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
log-dir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
#verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=osapi_compute,metadata
rpc_backend = rabbit
my_ip = 10.60.0.220
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api_database]
connection = mysql+pymysql://nova:Cloud1o1iNET@controller/nova_api

[database]
connection = mysql+pymysql://nova:Cloud1o1iNET@controller/nova

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = Cloud1o1iNET

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = Cloud1o1iNET

[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

	Đồng bộ database cho NOVA
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage db sync" nova

	Khởi động lại các dịch vụ của NOVA trên Controller Node
service nova-api restart
service nova-cert restart
service nova-consoleauth restart
service nova-scheduler restart
service nova-conductor restart
service nova-novncproxy restart

	- Xóa file SQLite mặc định
rm -f /var/lib/nova/nova.sqlite

3.	. CÀI ĐẶT TRÊN NOVA TRÊN COMPUTE NODE

	Khai báo các gói để cài đặt OpenStack Liberty
apt-get -y install software-properties-common add-apt-repository -y cloud-archive:mitaka

	Thiết lập IP
cat << EOF > /etc/network/interfaces

# NIC loopback
auto lo
iface lo inet loopback

# NIC MGNG
auto eth0
iface eth0 inet static
address 10.10.10.121
netmask 255.255.255.0

# NIC EXTERNAL
auto eth1
iface eth1 inet static
address 202.92.4.121
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 8.8.8.8
EOF
	Thiết lập hostname
	Cấu hình file /etc/hosts để phân giản IP cho các node

cat << EOF > /etc/hosts 
127.0.0.1   controller localhost
10.10.10.120    controller
10.10.10.121    compute1

EOF
	Update và khởi động lại node Compute node

apt-get update -y && apt-get upgrade -y && apt-get dist-upgrade -y && init 6
	Đăng nhập lại vào compute node và thực hiện các cài đặt tiếp theo

	Cài đặt gói the OpenStack client

apt-get install nova-compute
	Sao lưu file config cho nova
mv /etc/nova/nova.conf /etc/nova/nova.conf.bak
	Sửa file vi /etc/nova/nova.conf file với nội dung dưới
	
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=ec2,osapi_compute,metadata

rpc_backend = rabbit
auth_strategy = keystone
my_ip = 10.60.0.221
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = Cloud1o1iNET

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = Cloud1o1iNET

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

	Khởi động lại nova-compute trên Controller
service nova-compute restart
	Xóa file SQLite mặc định
rm -f /var/lib/nova/nova.sqlite

	Kiểm tra lại dịch vụ của NOVA trên Controller
nova service-list


	- Kết quả sẽ như sau:

	+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
	| Id | Binary           | Host       | Zone     | Status  | State | Updated_at                 | Disabled Reason |
	+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
	| 1  | nova-cert        | controller | internal | enabled | up    | 2015-11-17T10:00:41.000000 | -               |
	| 2  | nova-consoleauth | controller | internal | enabled | up    | 2015-11-17T10:00:44.000000 | -               |
	| 3  | nova-scheduler   | controller | internal | enabled | up    | 2015-11-17T10:00:44.000000 | -               |
	| 4  | nova-conductor   | controller | internal | enabled | up    | 2015-11-17T10:00:36.000000 | -               |
	| 6  | nova-compute     | compute1   | nova     | enabled | up    | 2015-11-17T10:00:44.000000 | -               |
	+----+------------------+------------+----------+---------+-------+----------------------------+-----------------+
	
	
4.	. CÀI ĐẶT NEUTRON TRÊN CONTROLLER NODE

	Tạo database cho Neutron
mysql -u root -pCloud1o1iNET

CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'Cloud1o1iNET';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'Cloud1o1iNET';
exit;

	Tạo user, gán role, endpoint cho neutron
openstack user create neutron --domain default --password Cloud1o1iNET 
openstack role add --project service --user neutron admin
openstack service create --name neutron --description "OpenStack Networking" network

	Tạo endpoint
openstack endpoint create --region RegionOne network public http://controller:9696
openstack endpoint create --region RegionOne network internal http://controller:9696
openstack endpoint create --region RegionOne network admin http://controller:9696

	Cài đặt các thành phần cho NEUTRON trên Controller Node
apt-get -y install neutron-server neutron-plugin-ml2 \
neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
neutron-metadata-agent

	Sao lưu file cấu hình
mv /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
	Tạo file neutron.conf với lệnh vi /etc/neutron/neutron.conf chứa nội dung sau
	
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
backend = rabbit
auth_strategy = keystone

notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True


[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf

[cors]
[cors.subdomain]
[database]
connection = mysql+pymysql://neutron:Cloud1o1iNET@controller/neutron
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = Cloud1o1iNET

[nova]
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = Cloud1o1iNET

[oslo_concurrency]

[oslo_messaging_amqp]

[oslo_messaging_notifications]
[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = Cloud1o1iNET

[oslo_policy]

[quotas]

[ssl]

	Cấu hình cho (ML2) plug-in
mv /etc/neutron/plugins/ml2/ml2_conf.ini  /etc/neutron/plugins/ml2/ml2_conf.ini.bak
	Tạo file vi /etc/neutron/plugins/ml2/ml2_conf.ini với nội dung sau

[DEFAULT]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vlan]
[ml2_type_geneve]
[ml2_type_flat]
flat_networks = public
vni_ranges = 1:1000

[securitygroup]
enable_ipset = True

	Configure the Linux bridge agent

	Sao lưu cấu hình cho file linuxbridge_agent.ini
mv /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak

	- Sửa file linuxbridge_agent.ini  bằng lệnh vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini với nội dung dưới

[DEFAULT]
[linux_bridge]
physical_interface_mappings = provider:em1

[vxlan]
enable_vxlan = True
local_ip = 10.60.0.220
l2_population = True

[agent]
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver


	Cấu hình cho layer-3 agent

	Sao lưu file cấu hình
mv /etc/neutron/l3_agent.ini /etc/neutron/l3_agent.ini.bak

	-Sửa file /etc/neutron/l3_agent.ini bằng lệnh vi /etc/neutron/l3_agent.ini với nội dung dưới

[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
external_network_bridge =

[AGENT]

	Cấu hình DHCP agent

	Sao lưu file dhcp_agent.ini
mv /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini.bak
	Sửa file dhcp_agent.ini bằng lệnh vi /etc/neutron/dhcp_agent.ini với nội dung dưới
[DEFAULT]
interface_driver = neutron.agent.linux.interface.BridgeInterfaceDriver
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True

verbose = True
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf

[AGENT]

	Cấu hình metadata agent
	Sao lưu file cp /etc/neutron/metadata_agent.ini
mv /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini.bak
	-Sửa file sau với lệnh vi /etc/neutron/metadata_agent.ini chứa nội dung dưới

	[DEFAULT]
nova_metadata_ip = controller
metadata_proxy_shared_secret = Cloud1o1iNET

[AGENT]

	hêm vào file /etc/nova/nova.conf trên node Controller đoạn dưới cùng dưới
	
[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = Cloud1o1iNET

service_metadata_proxy = True
metadata_proxy_shared_secret = Cloud1o1iNET

	Đồng bộ database cho NVOA
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
  
	Khởi động lại nova-api
service nova-api restart
	Khởi động lại các dịch vụ của NEUTRON trên CONTROLLER NODE
service neutron-server restart
service neutron-plugin-linuxbridge-agent restart
service neutron-dhcp-agent restart
service neutron-metadata-agent restart
service neutron-l3-agent restart
	Xóa file SQLite mặc định của OpenStack
rm -f /var/lib/neutron/neutron.sqlite


5.	. CÀI ĐẶT THÀNH PHẦN CỦA NEUTRON TRÊN COMPUTE NODE

a.	Cài đặt linuxbridge-agent trên node Compute
apt-get -y install neutron-linuxbridge-agent

	Sao lưu file /etc/neutron/neutron.conf
mv /etc/neutron/neutron.conf /etc/neutron/neutron.conf.bak
	Sửa file với lệnh vi /etc/neutron/neutron.conf chứa nội dung sau.

[DEFAULT]
core_plugin = ml2

rpc_backend = rabbit
auth_strategy = keystone


[agent]
root_helper = sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf

[cors]
[cors.subdomain]
[database]
[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password 
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = Cloud1o1iNET
[matchmaker_redis] 
             
[nova]

[oslo_concurrency] 
[oslo_messaging_amqp]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = Cloud1o1iNET

[oslo_policy]
[quotas]
[ssl]

b.	Configure the Linux bridge agent

	Sao lưu file /etc/neutron/plugins/ml2/linuxbridge_agent.ini
mv /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini.bak
	Sửa file bằng lệnh vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini với nội dung sau:

	[DEFAULT]
[agent]

[linux_bridge]
physical_interface_mappings = provider:em1

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

[agent]
[vxlan]
enable_vxlan = True
local_ip = 10.60.0.221
l2_population = True


	Thêm vào dưới cùng file /etc/nova/nova.conf trên Compute node với nội dung dưới
[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = Cloud1o1iNET
 
	Khởi động lại nova-compute
service nova-compute restart
	Khởi động lại Linux bridge agent
service neutron-plugin-linuxbridge-agent restart
6.	. Cai dat dashboad tren CONTROLLER

apt-get -y install openstack-dashboard
	Đăng nhập vào controller với IP 202.92.4.120/horizon
	
