# OmniPath Drive Installation

#### Building Intel OPA Drivers into WW Node Images

TODO: Add instructions on omnipath driver installation here!

After installing the OPA drivers into the $CHROOT environment, it's 
sometimes necessary to add a customized service file to ensure the drivers
are loaded during the stateless boot. 

If your OmniPath interfaces do not load on boot with the new image,
create the following service file in 
/etc/systemd/system/ (inside the $CHROOT environment!)
and symlink to
/etc/systemd/system/default.target.wants/ with the following contents:

```
[Unit]
Description=Service to Start OPA Device
After=sshd.service

[Service]
Type=oneshot
ExecStart=-/usr/sbin/modprobe -r ib_ipoib
ExecStart=-/usr/sbin/modprobe -r hfi1
ExecStart=/usr/sbin/modprobe hfi1
ExecStart=/usr/sbin/ifup ib0 
ExecStartPost=/bin/mount -a

[Install]
WantedBy=default.target
```

This will ensure that the Intel OPA Drivers are loaded during boot, and
the ib0 device starts in 'up' state. The 'mount -a' line is optional, 
depending on whether or not you're using the OmniPath network for
storage sharing (leave it in if you are!).
