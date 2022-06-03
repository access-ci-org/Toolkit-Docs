# OpenStack Automated Installation

## Editorial and General Notes

This automated installation guide uses the [Kolla Ansible](https://docs.openstack.org/kolla-ansible/yoga) project. This guide references Kolla Ansible's documentation (herein also called the "upstream" documentation) heavily. In some sections, it refers you (the reader) to a section of upstream documentation, to avoid duplicating it here. In other sections, it will provide a simplified version of the upstream documentation.

A major new version of OpenStack is released approximately every 6 months. These versions are named according to successive letters of the alphabet (e.g. Wallaby, Xena, Yoga). This guide is written for OpenStack Yoga release (April 2022). The deployment procedure may change slightly for later releases.

This guide is written to deploy OpenStack to hosts running the Ubuntu Server 20.04 operating system. As of June 2022, Ubuntu 22.04 was recently released but not fully tested with Kolla Ansible yet. Ubuntu 20.04 receives security and bug fix updates through 2025.

Most of the commands in this guide require running as the root user. So, when this guide provides shell commands, it is generally assumed that you run those commands from a root shell (obtained by typing `sudo su` and pressing enter).

### Prerequisite Skills

This guide assumes that you possess basic Linux systems administrator skills, namely:

- Installing a Linux-based server operating system
- Using a command-line shell (a.k.a. terminal or bash prompt)
- Knowing which user you are running commands as, and becoming the root user when needed
- Editing text files using a terminal-based editor like `nano` or `vi`
- Copying and pasting text into and out of a terminal
- Setting up TCP/IP networking, including static IP addresses
- Setting up SSH connections, including with public-key authentication

A full orientation to these concepts is outside the scope of this guide. If you need further guidance, please ask for help or search the web!

## Host Layout

We will do a multi-node deployment with three physical servers (hosts) arranged as follows:

| host     | IP address     | Purpose(s)                   |
|----------|----------------|------------------------------|
| ctl0     | 192.168.122.10 | control plane and deployment |
| compute0 | 192.168.122.11 | compute                      |
| compute1 | 192.168.122.12 | compute                      |

## Host Setup

First, install the Ubuntu Server operating system on each host. [Here is a guide for installing Ubuntu Server](https://ubuntu.com/tutorials/install-ubuntu-server#1-overview).

### Host Networking

Each host needs two network interfaces:

- One for management and OpenStack APIs
- One for instance floating IP addresses

The management interface needs a static IP address (one that doesn't change over time).

#### Determining Network Interface Names

First, determine your network interface names. On each host, enter `ip link show`, and you'll see something like this:

```
root@ctl0:/home/deployer# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
link/ether 52:54:00:05:b6:9b brd ff:ff:ff:ff:ff:ff
3: enp7s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
link/ether 52:54:00:d2:45:d3 brd ff:ff:ff:ff:ff:ff
```

Ignoring the loopback interface (named `lo`), this server has two network interfaces named `enp1s0` and `enp7s0`. Write down these interface names for later. let's use the first one as the management interface, and the second for instance floating IP addresses.

#### Setting Static IP Address for Management Interface

We need to set static (as opposed to dynamic, or DHCP-assigned) IP addresses for each host to provide a reliable way of connecting to them.

Set a static IP address for the first network interface. Using your preferred text editor, modify the netplan configuration (e.g. in `/etc/netplan/00-installer-config.yaml`) as follows.

```
network:
  ethernets:
    enp1s0:
      addresses:
        - 192.168.122.10/24
      routes:
        - to: default
          via: 192.168.122.1
  version: 2
```

Then run `netplan apply`.

If you did this via SSH, you likely need to kill and restart your SSH session, connecting to the server at its new IP address.

## Set Up SSH Connectivity Between Hosts

- Generate an SSH keypair from the deployment host
- Copy the resulting public key to the `known_hosts` file on each of the other hosts
- Use the keypair to make an initial SSH connection from the deployment host to each of the other hosts (in order to establish a known host key relationship) 

## Install Kolla Ansible

Follow all the steps in these sub-sections on the deployment host.

### Install Dependencies

Follow these steps on the deployment host: <https://docs.openstack.org/kolla-ansible/yoga/user/quickstart.html#install-dependencies>

It is best to use a Python virtual environment for the Python dependencies. This avoids polluting the Python packages that are controlled by the system's package manager.

### Install Kolla Ansible itself

Follow these steps on the deployment host: <https://docs.openstack.org/kolla-ansible/yoga/user/quickstart.html#install-kolla-ansible-for-deployment-or-evaluation>

### Install Ansible Galaxy Requirements

Follow these steps on the deployment host: <https://docs.openstack.org/kolla-ansible/yoga/user/quickstart.html#install-ansible-galaxy-requirements>

## Prepare Configuration

### Build Ansible Inventory File

```
[control]
10.0.0.10

[network:children]
control

[compute]
10.0.0.[11:12]

[monitoring]
10.0.0.10

[storage:children]
compute

[deployment]
localhost ansible_connection=local become=true
```

### Test Inventory and SSH Connectivity

Run this from the deployment host:
```
ansible -i multinode all -m ping
```
The output should show a successful connection for all nodes.

### Generate Kolla Passwords

Run this from the deployment host:
```
kolla-genpwd
```

### Global Configuration

TODO

The following steps simplify this section: <https://docs.openstack.org/kolla-ansible/yoga/user/quickstart.html#kolla-globals-yml>

For simplicity, we use the same distro inside containers that we use on the hosts:  

```
kolla_base_distro: "ubuntu"
```

#### Network Configuration

Enter the network interface names that you determined earlier, up in the "Networking" section:

```
network_interface: "eth0"
network_external_interface: "eth1"
```

Designate an unused IP address for internal API endpoints (TODO explain this earlier): 
```
kolla_internal_vip_address: "10.1.0.250"
```

## Deploy OpenStack

Follow these steps: <https://docs.openstack.org/kolla-ansible/yoga/user/quickstart.html#deployment>

