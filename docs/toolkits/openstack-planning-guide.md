# OpenStack Cloud Planning Guide

## Orientation to OpenStack

[![OpenStack Diagram](../img/openstack-diagram.jpeg)](../img/openstack-diagram.jpeg)

OpenStack is a set of inter-operable software services that combine to deliver a cloud infrastructure solution. On top of OpenStack, users build virtual information technology infrastructure to support their computing, data storage, and networking needs.

The most fundamental OpenStack services to know about are:

| Service type | OpenStack name | What it delivers to users               | AWS Equivalent |
|--------------|----------------|-----------------------------------------|----------------|
| identity     | Keystone       | Users and access control for the cloud  | IAM            |
| disk image   | Glance         | Operating systems for virtual computers | N/A            |
| computing    | Nova           | Virtual computers                       | EC2            |
| networking   | Neutron        | Software-defined network resources      | VPC            |

Some other commonly-adopted services include:

| Service type            | OpenStack name | What it delivers to users               | AWS Equivalent     |
|-------------------------|----------------|-----------------------------------------|--------------------|
| dashboard               | Horizon        | A web interface for using OpenStack     | Management Console |
| block storage           | Cinder         | Virtual disk drives                     | EBS                |
| object storage          | Swift          | Data storage buckets                    | S3                 |
| filesystem              | Manila         | Virtual shared folders                  | EFS                |
| bare metal              | Ironic         | Real, physical computers                | EC2 Bare Metal     |
| DNS                     | Designate      | DNS host records                        | Route 53           |
| orchestration           | Heat           | Automated control of OpenStack services | CloudFormation     |
| container orchestration | Magnum         | Push-button Kubernetes clusters         | EKS                |


### Control Plane and Data Plane

In a large distributed system like an OpenStack cloud, there is a logical separation between the Control Plane and Data Plane. The control plane consists of the set of OpenStack services (and their dependencies) that _manage_ users' resources on the cloud. The data plane consists of the users' resources themselves. The data plane includes virtual computers, networks, and various types of data storage.

First, let's look at the control plane. It's a sub-set of the diagram at the top of this page.

[![OpenStack Diagram](../img/openstack-diagram-control-plane-only.jpeg)](../img/openstack-diagram-control-plane-only.jpeg)

An OpenStack control plane has a few layers. At the bottom is at least one physical computer (a.k.a. server) to run the control plane services.

The next layer is infrastructure services. These are not part of OpenStack per se, but OpenStack services depend on them to deliver a functioning cloud.

| Service type      | Name(s)                                         | What it does                                              |
|-------------------|-------------------------------------------------|-----------------------------------------------------------|
| Database          | MariaDB or MySQL (optionally in Galera cluster) | Stores the persistent state of the control plane          |
| Message broker    | RabbitMQ                                        | Delivers messages between service components              |
| API load balancer | HAProxy                                         | Routes and secures network requests to OpenStack services |





### Load Balancing and High Availability

The entire OpenStack control plane can be load-balanced and/or high-availability.

When configured correctly, this means that every

This means

LB/HA architecture makes sense for larger clouds with downtime-intolerant workloads and hundreds of compute nodes. LB/HA is not essential when you are just getting started, and it is more complex to understand, deploy, manage, and troubleshoot.

If you are 

### Infrastructure Services


### Networking

### Types of Storage

This guide will set up storage for images (stored on the control plane host) and instance root disks (stored locally on compute hosts). Here is an orientation to other types of storage you may need in your cloud.

- Instance root disks
- Image storage
- Block storage (Cinder volumes)
- Object storage (Swift, Ceph RADOS)
- Shared filesystem (Manila, CephFS)

## Choosing a Deployment Method

We offer two deployment options: manual (via shell commands) and automated (via Ansible and Docker). You can think of these as using either hand tools or power tools to build your cloud.

Use the manual deployment when:
- You want to learn about each of the OpenStack services, and how they fit together
- You don't trust the automated deployment for whatever reason

Use the automated deployment when:
- You want a working cloud ASAP, and maybe learn how the pieces work later
- You want to use containers for your cloud's control plane
- You will have more than a small handful of nodes to deploy
- You also want automated upgrades for new versions of OpenStack

### References

- [Deploying OpenStack - what options do we have?](https://www.youtube.com/watch?v=8ODdvCogwl8) (from 2019 summit)
  - [Summary](https://imgur.com/Ux5Kyey) from one of the slides
- [OpenStack-Ansible docs](https://docs.openstack.org/project-deploy-guide/openstack-ansible/latest/)
- [kolla-ansible docs](https://docs.openstack.org/kolla-ansible/latest/)
- [Kayobe: An Introduction](https://www.stackhpc.com/pages/kayobe.html)
