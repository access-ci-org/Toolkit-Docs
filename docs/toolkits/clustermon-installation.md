# The Cluster Monitoring Toolkit

This toolkit provides two main tools for monitoring the
state and useage of your cluster: 
[Ganglia](http://ganglia.sourceforge.net/) and [OpenXDMoD](https://open.xdmod.org). 

Ganglia is used primarily to monitor load over time, and is useful for 
at-a-glance visualization of the current state of the cluster. 

OpenXDMoD is used to aggregate scheduler data into reports, 
and investigate useage metrics across departments, projects, or PIs.
This can be very useful in applying for additional funding, or
working to improve the general health of your research computing program.

## Ansible Playbook

The main playbook here is split between two roles, one for Ganglia, and one for OpenXDMoD.

They are triggered with the usual `-t` flag to ansible:

``` 
ansible-playbook -t ganglia,openxdmod clustermon.yml
```

The Ganglia role installes the ganglia meta daemon (gmetad), and the 
ganglia monitor daemon (gmond) on the headnode, as well as adding gmond
to the compute node images. A reboot of compute nodes will be required after this 
installation. The web app is also installed on the headnode, under a name-based virtual 
host. The servername is configured under `group_vars/all`.

The OpenXDMoD role installs the XDMoD webapp on the headnode, under a name-based
virtual host. The servername is configured under `group_vars/all`. This role also
creates the `xdmod` mysql user, and sets up regular cron jobs for ingesting and
shredding data from the scheduler (slurm by default). For more details on 
XDMoD installation, please see 
[the official XDMoD Configuration Guide](https://open.xdmod.org/8.0/configuration.html).

## Configuration

### Ganglia
Ganglia requires little configuration out of the box, but there
are many optional customizations available for the web interface. 
For more information, see:

### OpenXDMoD
After the initial installation, you will want to create an admin user
for OpenXDMoD. This refers only to the web interface, where you
can manage XDMoD users and generate detailed reports.

Run the `xdmod-setup` utility, and select `5` to create an Admin user. You
will be able to log in to the web interface with this.
