#  Installing Openstack (Victoria)  
---

This is a basic walkthrough of installing Openstack.  

## Pre-Requirements:
  This guide will assume you you are running a cluster with Ubuntu 20 with operational networking.  The installation should be similar for RHEL based distributions but the commands will be different, i.e. dnf, etc.

---

 Openstack uses a significant number of passwords.  Each service has about generally has a database password and a service user password.  You can generate a password by using:   `openssl rand -hex 10`
 These passwords will be stored in plaintext in the .conf files of the respective service on the controller nodes.

 Make sure your `/etc/hosts` on all the nodes to contain the address and hostname of the controller, hosts, and storage nodes (if applicable).  The hostname 'controller' has to be setup and linked to the headnode IP.  A basic visual looks like this:


![Screen Shot 2021-03-22 at 3 53 32 PM](https://user-images.githubusercontent.com/25777919/112050102-d7c59380-8b26-11eb-9af2-68b1431c4e41.png)



###  Headnode setup/configuration
 On the headnode, run the following:
``` bash
  apt install chrony mariadb-server memcached python-memcached wget qemu-img
  add-apt-repository cloud-archive:victoria
  add-apt-repository universe
  apt update
  apt dist-upgrade
```

``` bash
	apt install rabbitmq-server python3-PyMySQL openstack-selinux python3-openstackclient etcd
```

  Edit the `/etc/chrony.conf` file to include your NTP server.  For this example we use Google's public NTP server and allow the subnet 10.0.0.0/16 to use this server as a chronyc source:
```
	server time.google.com iburst
	allow 10.0.0.0/16
```

  Run:
``` bash
	service chronyd restart
```

  You can verify by running:
``` bash
	chronyc sources
```

  Create and edit /etc/mysql/mariadb.conf.d/99-openstack.cnf to include:
```
	[mysqld]
	bind-address = 10.0.0.10
	default-storage-engine = innodb
	innodb_file_per_table = on
	max_connections = 4096
	collation-server = utf8_general_ci
	character-set-server = utf8
```

**Make sure your bind-address is setup to the internal network of your headnode!**

  Run:
``` bash
  service mysql restart
	mysql_secure_installation
```

  verify you have the correct number of max connections set in mariadb with the following:
``` bash
	mysql -u root -p
```
  Enter your db password then:
``` mysql
	SHOW VARIABLES LIKE "max_connections";
```
  if it's not 4096, set it with:
``` mysql
	SET GLOBAL max_connections = 4096;
```

  Add the openstack user to rabbitmq and set that user to have configuration, write, and read permissions with the following:
``` bash
  rabbitmqctl add_user openstack {Rabbit_Password}
  rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

  edit /etc/memecached.conf to configure the service to use the management IP of the controller node:
```
	OPTIONS="-l 10.0.0.10,::1,controller"
```

  Restart memcached by running:
``` bash
	service memcached restart
```

  edit /etc/default/etcd to include:
```
ETCD_NAME="controller"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller=http://10.0.0.10:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.10:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.10:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.0.0.10:2379"
```

  run:
``` bash
        systemctl enable etcd
        systemctl start etcd
```


### Compute node pre-requirement setup & configuration
---

 On the compute nodes, run the following:
``` bash
	apt install chrony
  add-apt-repository cloud-archive:victoria
```

  edit the /etc/chrony/chrony.conf file to include:
```
	server controller iburst
	#pool 2.debian.pool.ntp.org offline iburst
```

  run:
``` bash
	server chrony restart
```

  you can verify by running:
``` bash
	chronyc sources
```

 Edit your `/etc/hosts` file to add in your nodes and associate an IP address to the controller name:
```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

{Internal Controller IP}    controller
{Internal compute1 IP}      compute1
{Internal compute2 IP}      compute2
```

 **Be sure to use the IP addresses of your nodes!**

## Installing Openstack services

First, we need to create the databases for the services in mariadb.  For this part, be sure to use your database passwords instead of 'KEYSTONE_DBPASS', 'CINDER_DBPASS', etc.

On the controller, connect to mysql:
``` bash
	mysql
```

and input your mysql password.

When you access the client, run:
``` mysql
	CREATE DATABASE keystone;
	GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
	GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';

	CREATE DATABASE glance;
	GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'GLANCE_DBPASS';
	GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'GLANCE_DBPASS';

	CREATE DATABASE nova_api;
	CREATE DATABASE nova;
	CREATE DATABASE nova_cell0;
	CREATE DATABASE placement;
	GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
	GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
	GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
	GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
	GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'NOVA_DBPASS';
	GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'NOVA_DBPASS';
	GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'PLACEMENT_DBPASS';
	GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'PLACEMENT_DBPASS';

	CREATE DATABASE neutron;
	GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'NEUTRON_DBPASS';
	GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'NEUTRON_DBPASS';

	CREATE DATABASE cinder;
	GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' IDENTIFIED BY 'CINDER_DBPASS';
	GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' IDENTIFIED BY 'CINDER_DBPASS';
```

then exit the database client.

Install necessary packages by running:
``` bash
	apt install keystone
```

  Then edit /etc/keystone/keystone.conf to contain the following:
   In the [database] section add:
```
	connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone
```

   In the [token] section add:
```
	provider = fernet
```

  Then populate the identity service database by running the following on the command line:
``` bash
	su -s /bin/sh -c "keystone-manage db_sync" keystone
```

  Then initialize the fernet key repos:
``` bash
	keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
	keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

  Bootstrap the identity service:
``` bash
	keystone-manage bootstrap --bootstrap-password ADMIN_PASS --bootstrap-admin-url http://controller:5000/v3/  --bootstrap-internal-url http://controller:5000/v3/ --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne
```

  edit the /etc/apache2/apache2.conf file to include:
```
	ServerName controller
```

  Run:
``` bash
	service apache2 restart
```

  Now, create a file called admin-openrc.sh in your home directory with the following:
```
	export OS_USERNAME=admin
	export OS_PASSWORD=ADMIN_PASS
	export OS_PROJECT_NAME=admin
	export OS_USER_DOMAIN_NAME=Default
	export OS_PROJECT_DOMAIN_NAME=Default
	export OS_AUTH_URL=http://controller:5000/v3
	export OS_IDENTITY_API_VERSION=3
```

  Now if you run:
``` bash
	source admin-openrc.sh
```

  You will be able to run openstack commands as the administrator.

  Now we need to create a few openstack pieces to get started. First, we need a service project to allow a space for service users to exist as well as a demo project for testing.  Run:
``` bash
	openstack project create --domain default --description "Service Project" service
	openstack project create --domain default --description "Demo Project" myproject
	openstack user create --domain default --password-prompt myuser
```

  This last command will ask for a password for the openstack user "myuser".

  Now we need a role for our new user.  You can name this myrole or something indicative of a user role.  Then we will apply a role assocation so that the myuser is in the myproject project. Run:
``` bash
	openstack role create myrole
	openstack role add --project myproject --user myuser myrole
```

  Now, we have added a user "myuser" to the project "myproject".  You can login with this user by creating an openrc.sh file with the following:
```
  export OS_USERNAME=myuser
  export OS_PASSWORD={PASSWORD ENTERED AT PROMPT}
  export OS_PROJECT_NAME=myproject
  export OS_USER_DOMAIN_NAME=Default
  export OS_PROJECT_DOMAIN_NAME=Default
  export OS_AUTH_URL=http://controller:5000/v3
  export OS_IDENTITY_API_VERSION=3
```

  Be sure to change the OS_PASSWORD to the password you entered when you created the "myuser" in openstack user create command.

Now we need to create some service accounts for the openstack services.

First run:
``` bash
	source admin-openrc.sh
```

  Then we can run:
``` bash
	openstack user create --domain default --password-prompt glance
```

  This will ask for a password.  Enter a password for the glance user, and then run:
``` bash
	openstack role add --project service --user glance admin
	openstack service create --name glance --description "OpenStack Image" image
	openstack endpoint create --region RegionOne image public http://controller:9292
	openstack endpoint create --region RegionOne image internal http://controller:9292
	openstack endpoint create --region RegionOne image admin http://controller:9292
```

  Now we will create and add the nova user:
``` bash
	openstack user create --domain default --password-prompt nova
```

  This will ask for a password.  Enter a password fo the glance user, then run:
``` bash
	openstack role add --project service --user nova admin
	openstack service create --name nova --description "OpenStack Compute" compute
	openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
	openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
	openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
	openstack user create --domain default --password-prompt placement
```

  This will ask for a password.  Enter a password for the placement user, then run:
``` bash
	openstack role add --project service --user placement admin
	openstack service create --name placement --description "Placement API" placement
	openstack endpoint create --region RegionOne placement public http://controller:8778
	openstack endpoint create --region RegionOne placement internal http://controller:8778
	openstack endpoint create --region RegionOne placement admin http://controller:8778
```

  Now we will create and add the neutron user:
``` bash
	openstack user create --domain default --password-prompt neutron
```

  This will ask for a password.  Enter a password for the neutron user, then run:
``` bash
	openstack role add --project service --user nova admin
	openstack service create --name nova --description "OpenStack Networking" network
	openstack endpoint create --region RegionOne network public http://controller:9696
	openstack endpoint create --region RegionOne network internal http://controller:9696
	openstack endpoint create --region RegionOne network admin http://controller:9696
```

  Now we will create and add the cinder user:
``` bash
	openstack user create --domain default --password-prompt cinder
```

  This will ask for a password.  Enter a password for the cinder user, then run:
``` bash
	openstack role add --project service --user cinder admin
	openstack service create --name cinderv2 --description "OpenStack Block Storage" volumev2
	openstack service create --name cinderv3 --description "OpenStack Block Storage" volumev3
	openstack endpoint create --region RegionOne volumev2 public http://controller:8776/v2/%\(project_id\)s
	openstack endpoint create --region RegionOne volumev2 internal http://controller:8776/v2/%\(project_id\)s
	openstack endpoint create --region RegionOne volumev2 admin http://controller:8776/v2/%\(project_id\)s
	openstack endpoint create --region RegionOne volumev3 public http://controller:8776/v3/%\(project_id\)s
	openstack endpoint create --region RegionOne volumev3 internal http://controller:8776/v3/%\(project_id\)s
	openstack endpoint create --region RegionOne volumev3 admin http://controller:8776/v3/%\(project_id\)s
```

  Install the openstack service packages by running:
``` bash
  apt install glance placement-api nova-api nova-conductor nova-novncproxy nova-scheduler neutron-server neutron-plugin-ml2 neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent openstack-dashboard cinder-api cinder-scheduler
```
``` bash
	yum install openstack-cinder openstack-nova-console ebtables python-rbd
```

  Edit /etc/glance/glance-api.conf and add in the [database] section:
```
	connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance
```
  **Replace GLANCE_DBPASS with your glance database password.**

  Then edit the [keystone_authtoken] section:
```
	www_authenticate_uri  = http://controller:5000
	auth_url = http://controller:5000
	memcached_servers = controller:11211
	auth_type = password
	project_domain_name = Default
	user_domain_name = Default
	project_name = service
	username = glance
	password = GLANCE_PASS
```
  **Replace GLANCE_PASS with your password for the glance user.**

  Edit the [paste_deploy] section to include:
```
	flavor = keystone
```

  Edit the [glance_store] section to include:
```
	stores = file,http
	default_store = file
	filesystem_store_datadir = /var/lib/glance/images/
```

  Now, we need to populate the image service database by running:
``` bash
	su -s /bin/sh -c "glance-manage db_sync" glance
```

  Start the image services and enable them so they run on startup:
``` bash
  service glance-api start
```

  To verify glance works, we can download a small image and try to store it.  Do this by running:
``` bash
	wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
```

  Now, we can add it to glance with:
``` bash
	openstack image create "cirros-test" --file cirros-0.4.0-x86_64-disk.img --disk-format raw --container-format bare --public
```

  And check with:
```
	openstack image list
```

  If you see the image in openstack, the Glance image store is working correctly.


Placement
  Edit the /etc/placement/placement.conf and edit the [placement_database] section to include:
```
connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement
```
  **Replace PLACEMENT_DBPASS to your database password.**

  Add the following into the [api] and [keystone_authtoken] sections:
```
[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = PLACEMENT_PASS
```

  **Replace PLACEMENT_PASS to your password for the placement user.**

Now, populate the placement database:
``` bash
  su -s /bin/sh -c "placement-manage db sync" placement
```

Once that's complete, restart the apache2 service with:
``` bash
  service apache2 restart
```

Now, onto Nova.  This is the compute/virtualization piece.
  Edit the /etc/nova/nova.conf file and add to the [DEFAULT] section:
```
	transport_url = rabbit://openstack:RABBIT_PASS@controller:5672/
	my_ip = 10.0.0.10
```
  **Replace RABBIT_PASS to your rabbit password.**

  Now add to the [api_database] section:
```
	connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api
```
  **Replace NOVA_DBPASS to your database password.**

  Add to the [database] section:
```
	connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
```
  **Replace NOVA_DBPASS to your database password.**

  Add to the [api] section:
```
	auth_strategy = keystone
```

  Add to the [keystone_authtoken] section:
```
  www_authenticate_uri = http://controller:5000/
  auth_url = http://controller:5000/
  memcached_servers = controller:11211
  auth_type = password
  project_domain_name = Default
  user_domain_name = Default
  project_name = service
  username = nova
  password = NOVA_PASS
```
  **Replace NOVA_PASS to your nova user passsword.**

  Add to the [vnc] section:
```
	enabled = true
	server_listen = $my_ip
	server_proxyclient_address = $my_ip
```

  Add to the [glance] section:
```
	api_servers = http://controller:9292
```

  Add to the [oslo_concurrencty] section:
```
	lock_path = /var/lib/nova/tmp
```

  Add to the [placement] section:
```
	region_name = RegionOne
	project_domain_name = Default
	project_name = service
	auth_type = password
	user_domain_name = Default
	auth_url = http://controller:5000/v3
	username = placement
	password = PLACEMENT_PASS
```
  **Replace PLACEMENT_PASS to your password for the placement user.**

  Add to the [neutron] section:
```
	auth_url = http://controller:5000
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	region_name = RegionOne
	project_name = service
	username = neutron
	password = NEUTRON_PASS
	service_metadata_proxy = true
	metadata_proxy_shared_secret = METADATA_SECRET
```
  **Replace NEUTRON_PASS with your neutron user's password.**
  Also, for METADATA_SECRET, set it to a suitable password/uuid.  You can create a uuid with:
``` bash
	uuidgen
```
  **You will need this later.**

  Populate the nova-api databases:
 ``` bash
 	su -s /bin/sh -c "nova-manage api_db sync" nova
```

Register the cell0 database:
``` bash
 su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
```

Create the cell1 cell:
``` bash
 su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
```

Populate the nova database:
``` bash
 su -s /bin/sh -c "nova-manage db sync" nova
```

Verify nova cell0 and cell1 are registered correctly:
``` bash
 su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
```
The table should contain cell1 and cell0.

Now, restart the compute services with:
``` bash
 service nova-api restart
 service nova-scheduler restart
 service nova-conductor restart
 service nova-novncproxy restart
```


####Neutron networking.

Edit the /etc/neutron/neutron.conf file:
  In the [database] section, add:
```
	connection= mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron
```
  **Replace NEUTRON_DBPASS with your database password.**

  In the [DEFAULT] section, add:
```
	core_plugin = ml2
	service_plugins = router
	allow_overlapping_ips = true
	transport_url = rabbit://openstack:RABBIT_PASS@controller
	auth_strategy = keystone
	notify_nova-on_port_status_changes = true
	notify_nova_on_port_data_changes = true
```
  **Replace RABBIT_PASS with your rabbitmq's password.**

  In the [keystone_authtoken] section, add:
```
	www_authenticate_uri = http://controller:5000
	auth_url = http://controller:5000
	memcached_servers = controller:11211
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = neutron
	password = NEUTRON_PASS
```
  **Replace NEUTRON_PASS with your neutron user's password.**

  In the [nova] section, add:
```
	auth_url = http://controller:5000
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	region_name = RegionOne
	project_name = service
	username = nova
	password = NOVA_PASS
```
  **Replace NOVA_PASS with your Nova user's password.**

  In the [oslo_concurrency] section, add:
```
	lock_path = /var/lib/neutron/tmp
```

 Now, edit the /etc/neutron/plugins/ml2/ml2_conf.ini file:
  In the [ml2] section, add:
```
	type_drivers = flat,vlan,vxlan
	tenant_network_types = vxlan
	mechanism_drivers = linuxbridge,l2population
	extension_drivers = port_security
```

  In the [ml2_type_flat] section, add:
```
	flat_networks = provider
```

  In the [ml2_type_vxlan] section, add:
```
	vni_ranges = 1:1000
```

  In the [securitygroup] section, add:
```
	enable_ipset = true
```

 Edit the /etc/neutron/plugins/ml2/linuxbridge_agent.ini file:
  In the [linux_bridge] section, add:
```
	physical_interface_mappings = provider:ens1
```
  **Be sure ens1 is the interface you want the linux_bridge agent to connect on.**

  In the [vxlan] section, add:
```
	enable_vxlan = true
	local_ip = OVERLAY_INTERFACE_IP_ADDRESS
	l2_population = true
```
 **Be sure to utilize the interface ip address to the address handling vxlan networks!**

  In the [securitygroup] section, add:
```
	enable_security_group = true
	firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

 Make sure your system kernel supports network bridge filters.  To do this, run:
``` bash
	sysctl net.bridge.bridge-nf-call-iptables
```
 If this returns "= 1" then it does.  Otherwise, run:
``` bash
	modprobe br_netfilter
	sysctl net.bridge.bridge-nf-call-iptables
```
 That should return 1.

 Edit the /etc/neutron/l3_agent.ini file:
  In the [DEFAULT] section, add:
```
	interface_driver = linuxbridge
```

 Edit the /etc/neutron/dhcp_agent.ini file:
  In the [DEFAULT] section, add:
```
	interface_driver = linuxbridge
	dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
	enable_isolated_metadata = true
```

 Edit the /etc/neutron/metadata_agent.ini file:
  In the [DEFAULT] section, add:
```
	nova_metadata_host = controller
	metadata_proxy_shared_secret = METADATA_SECRET
```
  **Replace METADATA_SECRET with the METADATA_SECRET you used in the /etc/nova/nova.conf file.**

 populate the neutron database by running:
``` bash
	su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

 restart the compute api service:
``` bash
	service nova-api restart
```

 Start and enable the networking services:
``` bash
  service neutron-server restart
  service neutron-linuxbridge-agent restart
  service neutron-dhcp-agent restart
  service neutron-metadata-agent restart
  service neutron-l3-agent restart
```


Configuring Horizon:
 Edit /etc/openstack-dashboard/local_settings:
  Edit the settings to have the following:
```
	OPENSTACK_HOST = "controller"
	ALLOWED_HOSTS = ['HOSTS']
```
   **Setting ALLOWED_HOSTS to '*' allows everyone to access the dashboard, but is a potential risk. Change HOSTS to hosts that you want to allow access.**

```
	SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

	CACHES = {
		'default': {
        	 'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
	         'LOCATION': 'controller:11211',
   	 }
	}
  OPENSTACK_KEYSTONE_URL = "http://%s/identity/v3" % OPENSTACK_HOST
	OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
	OPENSTACK_API_VERSIONS = {
	   "identity": 3,
	   "image": 2,
	   "volume": 2,
	}
	OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
	OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"
```

 Add the following to /etc/apache2/conf-available/openstack-dashboard.conf:
```
	WSGIApplicationGroup %{GLOBAL}
```

 Now, restart the web service:
``` bash
	service apache2 reload
```

After that, you should be able to access the dashboard by going to `http://controller/dashboard`
If that doesn't work, you're in trouble.

To configure Cinder:
 edit /etc/cinder/cinder.conf:
  Add to the [database] section:
```
	connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder
```
  **Replace CINDER_DBPASS with your cinder database password.**

  In the [DEFAULT] section, add:
```
	transport_url = rabbit://openstack:RABBIT_PASS@controller
 	auth_strategy = keystone
	my_ip = 10.0.0.10
```
  **Replace RABBIT_PASS with your rabbit password.**

  In the [keystone_authtoken] section, add:
```
  www_authenticate_uri = http://controller:5000
	auth_url = http://controller:5000
	memcached_servers = controller:11211
	auth_type = password
	project_domain_id = default
	user_domain_id = default
	project_name = service
	username = cinder
	password = CINDER_PASS
```
  **Replace CINDER_PASS with your Cinder user's password.**

  In the [oslo_concurrency] section, add:
```
	lock_path = /var/lib/cinder/tmp
```

 Populate the Block Storage database by running:
``` bash
	su -s /bin/sh -c "cinder-manage db sync" cinder
```

 Restart the compute service:
``` bash
	service nova-api restart
```

 Start and enable the cinder services:
``` bash
  service cinder-scheduler restart
  service apache2 restart
```


##Compute Nodes
Now, onto the compute nodes.  If you have a large number of compute nodes, you may want to use a deployment tool such as Openstack-Ironic, Openstack-Ansible, Cobbler, etc.  Otherwise, run through this guide on each compute.

First, install Openstack packages:
``` bash
  apt install nova-compute neutron-linuxbridge-agent

#	yum install openstack-nova-compute openstack-neutron-linuxbridge ebtables ipset
```

 Edit /etc/nova/nova.conf:
  In the [DEFAULT] section, add:
```
#	enabled_apis = osapi_compute,metadata
	transport_url = rabbit://openstack:RABBIT_PASS@controller
	my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS
#	use_neutron = true
#	firewall_driver = nova.virt.firewall.NoopFirewallDriver
#	vif_plugging_is_fatal: false
#	vif_pluggins_timout: 0
```

  In the [api] section, add:
```
	auth_strategy = keystone
```

  In the [keystone_authtoken] section, add:
```
  www_authenticate_uri = http://controller:5000/
  auth_url = http://controller:5000/
  memcached_servers = controller:11211
  auth_type = password
  project_domain_name = Default
  user_domain_name = Default
  project_name = service
  username = nova
  password = NOVA_PASS
```
  **Replace NOVA_PASS with your Nova user's password.**

  In the [vnc] section, add:
```
	enabled = true
	server_listen = 0.0.0.0
	server_proxyclient_address = $my_ip
	novncproxy_base_url = http://controller:6080/vnc_auto.html
```

  In the [glance] section, add:
```
	api_servers = http://controller:9292
```

  In the [oslo_condurrency] section, add:
```
	lock_path = /var/lib/nova/tmp
```

  In the [placement] section, add:
```
	region_name = RegionOne
	project_domain_name = Default
	project_name = service
	auth_type = password
	user_domain_name = Default
	auth_url = http://controller:5000/v3
	username = placement
	password = PLACEMENT_PASS
```
  **Replace PLACEMENT_PASS with the placement users' password.**

  In the [neutron] section, add:
```
	auth_url = http://controller:5000
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	region_name = RegionOne
	project_name = service
	username = neutron
	password = NEUTRON_PASS
```
  **Replace NEUTRON_PASS with your neutron user's password.**

  In the [cinder] section, add:
```
	os_region_name = RegionOne
```

  Check to ensure your nodes support hardware acceleration with:
```
	egrep -c '(vmx|svm)' /proc/cpuinfo
```
If this returns a value of 0, your hardware does not support hardware acceleration and should be configured to utilize QEMU instead of KVM.

  Start the compute services:
```
  service nova-compute restart
```

 Edit the /etc/neutron/neutron.conf file:
  In the [database] section, be sure any connection options are commented out.
  In the [DEFAULT] section, add:
```
	transport_url = rabbit://openstack:RABBIT_PASS@controller
	auth_strategy = keystone
```
  **Replace RABBIT_PASS with your rabbit's password.**

  In the [keystone_authtoken] section, add:
```
	www_authenticate_uri = http://controller:5000
	auth_url = http://controller:5000
	memcached_servers = controller:11211
	auth_type = password
	project_domain_name = default
	user_domain_name = default
	project_name = service
	username = neutron
	password = NEUTRON_PASS    
```
  **Replace NEUTRON_PASS with the neutron user's password.**

  In the [oslo_concurrency] section, add:
```
	lock_path = /var/lib/neutron/tmp
```

 Edit the /etc/neutron/plugins/ml2/linuxbridge_agent.ini file:
  In the [linux_bridge] section, add:
```
	physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME
```
  **Make sure the PROVIDER_INTERFACE_NAME is replaced with your underlying provider physical interface.**

  In the [vxlan] section, add:
```
	enable_vxlan = true
	local_ip = OVERLAY_INTERFACE_IP_ADDRESS
	l2_population = true
```
  **Make sure the OVERLAY_INTERFACE_IP_ADDRESS is replaced with your vxlan overlay network interface.**

  In the [securitygroup] section, add:
```
	enable_security_group = true
	firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

  Ensure the OS has the br_netfilter module loaded by running:
``` bash
	sysctl net.bridge.bridge-nf-call-iptables
```
  If that does not return a value of 1, then load the module with:
``` bash
	modprobe br_netfilter
```

  Restart the compute service with:
``` bash
	service nova-compute restart
```

  Retart the linuxbridge agents:
``` bash
	service neutron-linuxbridge-agent restart
```

  After setting up the compute nodes, go back to the CONTROLLER node and run:
``` bash
	source admin-openrc
	su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova
```

  This will add the computes to the headnode's compute database.


Now you can run through a VM installation on horizon to test your setup:

https://docs.openstack.org/horizon/victoria/user/log-in.html
