# OpenStack Operations and Maintenance

### Useful Additions and Operations

---

#### Floating IP Addresses

Adding in and configuring a subnet of floating IPs can be extremely useful for your users if they would like to have a persistent IP address.  It's also good for administrators as this allows VMs that need to be accessible but not necessarily publicly routable to not use an IP address, which are becoming increasingly scarce, and simply have them on an internal network.  To set this up, we do the following:

TO-DO: add in steps for this.

#### Auto-allocated Network

The auto-allocated network can save a large amount of time for the administrators and/or support personnel by having users simply create a network/router/subnet with one line instead of the gratuitously arduous way of creating them one by one with the CLI.  This is also necessary to be activated for popular additional interfaces like Exosphere [TO-DO: include link here].  The setup is relatively simple for how much work it cuts out for your users.  

TO-DO: Add in the steps for this

#### Adding and Managing Users

This is probably something that everyone will want.  In this section, we will go over adding users, allocations, adjusting quotas for specific use cases, etc.  

TO-DO: Add in steps for adding users, allocations, and adjusting quotas.


### Maintenance

---

#### NTP
	- What NTP Errors look like
	- How to fix

#### Rabbit MQ

	- Checks:
``` bash
rabbitmqctl cluster_status
```
	- What rabbitMQ errors look like
	- How to fix

#### Mysql

	- Checks for a single node database:
SSH to the MYSQL node and log into the database with:
`mysql`
Also, you can check the logs to make sure there are no discrepancies.

	- Checks for a multiple node cluster:
SSH to the MYSQL node and log into the database with:
`mysql`
and run
```
SHOW GLOBAL STATUS LIKE 'wsrep_%';
```
And verify the cluster count matches the number of nodes in your database cluster.

	- What MYSQL errors look like
	- How to fix
	- Database Discrepancies
		Throughout the life of your cloud, you may come into issues where the database and openstack reflect different information.  This could be a VM that shows a host on one machine but it's actually on another, a volume that is stuck in a `reserved` state but won't change back to active/available, etc.  To fix this, we need to log into the MYSQL cluster by running `mysql` and updating the discrepancy with `update ITEM set VALUE=DESIRED_VALUE  where id='UUID';`.  You can verify which specific item(s) you'll adjust by adjusting the formula to, `show ITEM where id='UUID';`.

#### Nova

	- Checks:
To check the nova status, source the Admin openrc file or app credential and run:
```
openstack compute service list
```

This will list all nova services and show if any are down or not running.  A "XXX" status simply means that service has missed a heartbeat message in rabbitmq to the control node.  A "down" status means the service has missed enough to be considered off or problematic.
	- What Nova errors look like
	- How to Fix
	- How to take down nodes for maintenance


#### Neutron

	- Checks:
To check the neutron services status, source the Admin openrc file or app credential and run:

```
openstack network agent list
```

This is similar to the previous command, except it shows all the neutron services.
	- What Neutron errors look like
	- How to fix

#### Keystone
	- What Keystone errors look like
	- How to fix


This page is a work in progress, more content to follow.
