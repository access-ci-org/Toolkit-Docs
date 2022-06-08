# OpenStack Automated Installation

## Editorial and General Notes

This automated installation guide uses the [Kolla Ansible](https://docs.openstack.org/kolla-ansible/yoga) project. This guide references Kolla Ansible's documentation (herein also called the "upstream" documentation) heavily. In some sections, it will refer you (the reader) to a specific section of upstream documentation to avoid duplicating it here. In other sections, this guide will provide a simplified version of the upstream documentation.

On the size and shape of your cloud:

- For simplicity, this guide shows deployment of an example cloud with one control node and two compute nodes. If you have more than two compute nodes, this guide is still for you!
- The only potential complication is if you want more than one control node to make a load-balanced, highly-available (LB/HA) control plane. LB/HA architecture makes sense for larger clouds with downtime-intolerant workloads and hundreds of compute nodes. Kolla Ansible fully supports deploying a LB/HA control plane, but that is beyond the scope of this guide. LB/HA is not essential when you are just getting started, and it is more complex to understand, deploy, manage, and troubleshoot.

On software versions:

- A major new version of OpenStack is released approximately every 6 months. These versions are named according to successive letters of the alphabet (e.g. Wallaby, Xena, Yoga). This guide is written for OpenStack Yoga release (April 2022). The deployment procedure may change slightly for later releases.
- A new LTS (long-term support) version of Ubuntu is released every 2 years. This guide is written to deploy OpenStack to servers running the Ubuntu Server 20.04 operating system. As of June 2022, Ubuntu 22.04 is newly released, but not yet fully tested with Kolla Ansible. Ubuntu 20.04 receives security and bug fix updates until 2025.

On users and SSH:

- Many of the commands in this guide require running as the root user. So, when this guide provides shell commands, it is generally assumed that you run those commands from a root shell (obtained by typing `sudo su` and pressing enter).
- Generally, this guide uses SSH to connect to each node as the `deployer` user, then uses `sudo su` to become the root user on those nodes, because this is simplest to configure.  You can use SSH a different way, but if you do, be prepared to modify some of the example commands below.

### Prerequisite Skills

This guide assumes that you possess basic Linux systems administrator skills and understand their underlying concepts, namely:

- Installing a Linux-based server operating system ([guide for installing Ubuntu server](https://ubuntu.com/tutorials/install-ubuntu-server)).
- Making SSH connections, and using SSH inside a command-line shell (a.k.a. terminal or bash prompt) on your particular desktop operating system ([guide to using SSH](https://www.digitalocean.com/community/tutorials/how-to-use-ssh-to-connect-to-a-remote-server)).
    - Telling the difference between a local shell on your computer, and a remote SSH session (look at the prompt).
    - Copying and pasting text into and out of an SSH session, on your particular desktop operating system.
- Knowing which user you are running commands as ([guide for determining current user](https://www.howtogeek.com/410423/how-to-determine-the-current-user-account-in-linux/)), and becoming the root user when needed ([guide to running commands as root](https://kb.iu.edu/d/amyi)).
- Editing text files using a terminal-based editor like `nano` ([guide to nano](https://kb.iu.edu/d/aeug)) or `vi` ([guide to vi](https://kb.iu.edu/d/adxz)). _nano is easier for beginners._
- Setting up TCP/IP networking, including static IP addresses.
- Setting up SSH public-key authentication ([guide to public key authentication](https://kb.iu.edu/d/aews)).

A full orientation is outside the scope of this guide. If you need further guidance, please try the links above, search the web, and/or ask for help.

## Node and Network Layout

We will do a multi-node deployment with three physical servers (a.k.a. nodes or hosts) arranged as follows:

| node     | IP address     | Purpose(s)                   |
|----------|----------------|------------------------------|
| ctl0     | 192.168.122.10 | control plane and deployment |
| compute0 | 192.168.122.11 | compute                      |
| compute1 | 192.168.122.12 | compute                      |

In this example, the management network is a private `/24` subnet (netmask of `255.255.255.0`), with the default gateway `192.168.122.1`.

## Basic Node Setup

### Install Operating System

First, install the Ubuntu Server operating system on each node. [Here is a guide for installing Ubuntu Server](https://ubuntu.com/tutorials/install-ubuntu-server#1-overview).  

This guide assumes that you set up a local user named `deployer` during setup. If you create a different username, be prepared to change the username in some of the commands below. Record the password that you set, you will need to enter it later.

Don't worry about configuring the network interfaces during operating system installation, we will do that soon.

### Configure Node Networking

Each node needs two network interfaces, each connected to a physical network:

- One for management and OpenStack APIs
- One for instance floating IP addresses

The management interface needs a static IP address (one that doesn't change over time).

#### Determine Physical Network Interface Names

First, determine your network interface names. On each node, enter `ip link show`, and you'll see something like this:

```shell
root@ctl0:/home/deployer# ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
link/ether 52:54:00:05:b6:9b brd ff:ff:ff:ff:ff:ff
3: enp7s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
link/ether 52:54:00:d2:45:d3 brd ff:ff:ff:ff:ff:ff
```

Ignoring the loopback interface (named `lo`), this server has two network interfaces named `enp1s0` and `enp7s0`. Your interface names will probably be different! Write down these interface names for later. Let's use the two interfaces as follows:

| interface order | example interface name | purpose               |
|-----------------|:-----------------------|-----------------------|
| first           | enp1s0                 | management, APIs      |
| second          | enp7s0                 | floating IP addresses |

#### Set Static IP Address for First Network Interface

We need to set static (as opposed to dynamic, or DHCP-assigned) IP addresses for each node to provide a reliable way of connecting to them.

Set a static IP address for the first network interface. Using your preferred text editor, modify the netplan configuration (e.g. in `/etc/netplan/00-installer-config.yaml`) as follows. If you prefer, replace 1.1.1.1 with your local orgnaization's nameserver.

```yaml
network:
  ethernets:
    enp1s0:
      addresses:
        - 192.168.122.10/24
      routes:
        - to: default
          via: 192.168.122.1
      nameservers:
        addresses:
          - 1.1.1.1
  version: 2
```

Then run `netplan apply`.

If you did this via SSH, you likely need to kill and restart your SSH session, connecting to the server at its new IP address.

#### Configure Hostname Resolution

On each node, add something like this to the bottom of the `/etc/hosts` file:

```
192.168.122.10 ctl0
192.168.122.11 compute0
192.168.122.12 compute1
```

This lets the nodes connect to each other by their name instead of IP address.

Test it with the `ping` commands, e.g. by running `ping compute1` from ctl0. _You must be able to ping every node by its hostname from every other node_, otherwise you will have trouble later in the process.

### Configure SSH Connectivity Between Nodes

First, generate an SSH keypair on the deployment node, ctl0. Do this by running `ssh-keygen` as the root user, and answering the question prompts. For example:

```shell
root@ctl0:/home/deployer# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:95GBN6RU4eRJb19+NdpbKJXqTlz9xZtQH+uHG9Gf9ac root@ctl0
The key's randomart image is:
+---[RSA 3072]----+
|          ..*.   |
|         . O o . |
|          o B =+o|
|           . B+BB|
|        S . =o+o%|
|         . + +oo&|
|            = o**|
|           o   +o|
|            . E  |
+----[SHA256]-----+
root@ctl0:/home/deployer#
```

Second, copy the resulting public key (stored in `/root/.ssh/id_rsa.pub` on ctl0) to the `/home/deployer/.ssh/authorized_keys` file on each of the other nodes. You can use the `ssh-copy-id` command to expedite this, e.g. with `ssh-copy-id deployer@compute0`.

Third, make an initial SSH connection from the deployment node (root user) to each of the other node, e.g. with `ssh deployer@compute0`. When prompted, enter the password, and also accept the host key if prompted. This will establish a host key relationship so that Ansible can connect later.

## Install Kolla Ansible

### Install Dependencies

Follow these steps on the deployment node: <https://docs.openstack.org/kolla-ansible/yoga/user/quickstart.html#install-dependencies>

From here on, use the "using a virtual environment" steps -- this avoids polluting the Python packages that are controlled by the system's package manager. Note that you'll need to replace `/path/to/venv` with a real filesystem path. This guide will use `/root/kolla-ansible-venv`.

So, enter these commands:

```bash
apt update
apt install python3-dev libffi-dev gcc libssl-dev
apt install python3-venv
python3 -m venv /root/kolla-ansible-venv
source /root/kolla-ansible-venv/bin/activate
pip install -U pip
pip install 'ansible>=4,<6'
```

### Install Kolla Ansible itself

Follow these steps on the deployment node: <https://docs.openstack.org/kolla-ansible/yoga/user/quickstart.html#install-kolla-ansible-for-deployment-or-evaluation>

```
pip install git+https://opendev.org/openstack/kolla-ansible@stable/yoga
mkdir -p /etc/kolla
chown root:root /etc/kolla
cp -r /root/kolla-ansible-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp /root/kolla-ansible-venv/share/kolla-ansible/ansible/inventory/* /etc/kolla
```

### Install Ansible Galaxy Requirements

Follow these steps on the deployment node: <https://docs.openstack.org/kolla-ansible/yoga/user/quickstart.html#install-ansible-galaxy-requirements>

```
kolla-ansible install-deps
```

## Prepare Configuration

### Build Ansible Inventory File

```ini
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

Run this from the deployment node:
```
ansible -i multinode all -m ping
```
The output should show a successful connection for all nodes.

### Generate Kolla Passwords

Run this from the deployment node:
```
kolla-genpwd
```

### Global Configuration

TODO

The following steps simplify this section: <https://docs.openstack.org/kolla-ansible/yoga/user/quickstart.html#kolla-globals-yml>

For simplicity, we use the same distro inside containers that we use on the bare nodes:  

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

