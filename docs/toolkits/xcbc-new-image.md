Overview
========

While the basic XCBC Installation provides for a standard compute image, it's often the case
that HPC systems rely on heterogeneous architectures, which can require different node images. Some
sites may also wish to provide separate visualization or login nodes.

Additional roles in the XCBC exist for building two other predefined node types: Login, and GPU nodes. 
These each may require site-specific configuration, but act as guides for setting up these more customized 
appliances.

## Where to install software?
It's important to balance the size of the compute node
image with the ease of installing software. In many cases
it is better to install to the shared filesystem, rather than
adding additional bloat to the compute image, which slows down
the boot/build process, and takes up space on the headnode
root partition (depending on your filesystem configuration) and
in the Warewulf database.

Step-by-Step Image Building (Warewulf)
======================================

This guide provides a minimal set of guidelines for building a custom node image. 
The pieces that are not generalizable are left as an exercise for the reader, based on your 
local requirements.

To initialize a new chroot environment, based on the same warewulf template as the original compute nodes:
```wwmkchroot compute-nodes /opt/ohpc/admin/images/new-image #new-image can be whatever... as can this whole path!```

It's important to be aware of the amount of disk space available in /opt when adding new images - these
can balloon quite rapidly! 

It's easiest to install new packages into the new image chroot via yum - from *outside* the image:
``` yum -y --installroot=/opt/ohpc/admin/images/new-image install chrony kernel lmod-ohpc grub2 freeipmi ipmitool```

yum will not function correctly inside the chroot environment without various changes, which could lead to breaking 
the headnode.

If you need to install software that *isn't* available from yum into your new image, copy the source into the appropriate
place, chroot into the environment, and build away:
```cp /root/package.tar.gz /opt/ohpc/admin/images/new-image/root/```
```chroot /opt/ohpc/admin/images/new-image```
```cd /root ```
```tar xfvz ./package.tar.gz ```
```cd package```
```./configure```
``` ... etc. ```

It is preferable, however, to install software in a shared filesystem rather than inside of the image, 
since large image result in slower boot times and more RAM used for the tmpfs.

If you need to build additional drivers for something like a GPU, it may be necessary to mount additional
directories into the chroot environment.

Make sure you'll be able to ssh over to the new image:
``` cp /root/.ssh/authorized_keys /opt/ohpc/admin/images/new-image/root/.ssh/authorized_keys ```

Also, copy over your default /etc/fstab for your environment:
```cp /opt/ohpc/admin/images/centos7-compute/etc/fstab /opt/ohpc/admin/images/new-image/root/.ssh/authorized_keys ```
(for example).

You may also need to enable any important services in the chroot environment, so that they will start during
the stateless boot. This include things like slurmd, chrony, and munge in particular.
```chroot /opt/ohpc/admin/images/new-image ```
```systemctl enable slurmd chronyd munge```

Once you've installed all the customized software you'll need, you can create the new VNFS via:
``` wwvnfs -y --chroot /opt/ohpc/admin/images/new-image ```

That will add a vnfs image with the name `new-image` to the database (there's a --name flag if you want a different name than the last directory in the path-to-chroot)
Confirm the new vnfs via `wwsh vnfs list`

If you'd like to boot an existing node with the new image, you can modify the node entry in the WW database, via:
``` wwsh provision set compute-0 --wwvnfs=new-image```

Before doing this, of course, be sure the node will not be scheduled by slurm:
``` scontrol update nodename=compute-0 state=down reason=testing-new-image #reason is necessary but no specific list of flags need be used```

