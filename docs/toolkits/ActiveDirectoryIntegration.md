# Active Directory Integration

#### Headnodes and Login Nodes

The login node (or headnode) integration proceeds as usual with **realm join** to allow users in Active Directory ssh access. 

For allowing only a certain AD group to login, it's easiest to use:

```
access_provider = simple
simple_allow_groups = groupname1, groupname1
```

in the domain section of /etc/sssd.conf

#### Compute Nodes

The tricky bit comes with getting the compute nodes synced up. Our typical solution is to have users added to the local /etc/passwd file on their first login, which is then synced to the compute nodes as usual. Having the computes join the realm directly would cause a lot of extra requests against AD, which can cause problems. It *may* be possible to have SSSD on the headnode act as a proxy for the computes, but I'm not sure if that would fix the "flood of AD requests" issue. 

By default, Warewulf runs a script from /usr/bin/cluster-env each time a user logs in, which populates local ssh keys. This runs as the login user, so it's not possible to do the actual add to passwd here, but I add a line like

```
echo $(getent passwd $_UID) >> /tmp/new_users.${_UID}
```

to the end of the ssh-key creation stanza. 
Then, I'll create a script, at /usr/bin/sync-users with the following:

```
#!/bin/bash
while read -r user
do
  username=$(echo $user | cut -f 1 -d':')
  if [[ -z $(getent passwd $username) ]]
  then
    echo "Inserting new user: $username into passwd"
    echo $user >> /etc/passwd
  fi 
done < <(paste -d'\n' /tmp/new_users* 2> /dev/null)

rm /tmp/new_users*

wwsh file sync
```

Then, just have that script run every 5 minutes or so in crontab. 
