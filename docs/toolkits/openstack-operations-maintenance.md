# OpenStack Operations and Maintenance

Rabbit MQ

Checks:
rabbitmqctl cluster_status



Mysql

Checks for a single node database:
SSH to the MYSQL node and log into the database with:
`mysql`
Also, you can check the logs to make sure there are no discrepancies.

Checks for a multiple node cluster:
SSH to the MYSQL node and log into the database with:
`mysql`
and run
```
SHOW GLOBAL STATUS LIKE 'wsrep_%';
```

And verify the cluster count matches the number of nodes in your database cluster.

Nova

To check the nova status, source the Admin openrc file or app credential and run:
```
openstack compute service list
```

This will list all nova services and show if any are down or not running.  A "XXX" status simply means that service has missed a heartbeat message in rabbitmq to the control node.  A "down" status means the service has missed enough to be considered off or problematic.


Neutron

To check the neutron services status, source the Admin openrc file or app credential and run:

```
openstack network agent list
```

This is similar to the previous command, except it shows all the neutron services.

This page is a work in progress, more content to follow.
