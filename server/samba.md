The Samba service has access to `/main-pool/trunk` and `/main-pool/yolo`, and
exposes it over the LAN. Firewall settings on the Unifi router makes sure only
trusted CIDRs can initiate connections to it.

# Container Setup

The samba server runs in a LXC container. The container is created using
Proxmox web UI, with ID 1002 and name `samba-server`. Note that here 1001 is
just an arbitrary container ID, and has nothing to do with UIDs or GIDs. The
container uses bridged networking and gets an IP in the same subnet as the
host. It runs Ubuntu 20.04. Locale is set to `en_US.UTF-8`.

# Mounting the Dataset

These datasets need to be manually bound into the LXC container:

```bash
root@samba-server:~# mkdir -p /mnt/main-pool-trunk /mnt/main-pool-yolo
```

```bash
songgao in home in ~
❯ sudo pct set 1002 -mp0 /main-pool/trunk,mp=/mnt/main-pool-trunk

songgao in home in ~
❯ sudo pct set 1002 -mp1 /main-pool/yolo,mp=/mnt/main-pool-yolo
```

# User Creation and UID Mapping

To avoid files showing up being owned by UIDs, users need to be created both on
the host and in the LXC. Groups are made to make it easier for different users
to both have write permissions:

```bash
songgao in home in ~ took 2s
❯ sudo groupadd --gid 2001 samba_rw

songgao in home in ~
❯ sudo groupadd --gid 2002 samba_ro

songgao in home in ~
❯ sudo useradd --gid 2001 samba_songgao

songgao in home in ~
❯ sudo useradd --gid 2001 samba_xinyuzhao

songgao in home in ~
❯ cat /etc/passwd | grep samba_
samba_songgao:x:1002:2001::/home/samba_songgao:/bin/sh
samba_xinyuzhao:x:1003:2001::/home/samba_xinyuzhao:/bin/sh
```

Edit `/etc/pve/nodes/home/lxc/1002.conf` on host to make the uid/gid mapping,
with special treatment for `samba_songgao` and `samba_xinyuzhao` (see doc
[here](https://pve.proxmox.com/wiki/Unprivileged_LXC_containers)):

```
# map uid 1002 and 1003 from host to 1002 and 1003 in LXC
lxc.idmap = u 0 100000 1002
lxc.idmap = u 1002 1002 2
lxc.idmap = u 1004 101004 64532

# map gid 2001 and 2002 from host to 2001 and 2002 in LXC
lxc.idmap = g 0 100000 2001
lxc.idmap = g 2001 2001 2
lxc.idmap = g 2003 102003 63533
```

And enable `subuid` and `subgid` from `root` to `1002` and `1003` by editing
lines from `/etc/subuid` and `/etc/subgid`.

Then create the same users inside LXC, and use the same UID/GID as the host
(since we mapped that way above):

```bash
root@samba-server:~# groupadd --gid 2001 samba_rw
root@samba-server:~# groupadd --gid 2002 samba_ro
root@samba-server:~# useradd --uid 1002 --gid 2001 samba_songgao
root@samba-server:~# useradd --uid 1003 --gid 2001 samba_xinyuzhao
```

Lastly, change `/main-pool/trunk` and `/main-pool/yolo` to group `samba_rw` and
make them writable by group:

```bash
songgao in home in ~
❯ sudo chown -R root:samba_rw /main-pool/trunk

songgao in home in ~
❯ sudo chown -R root:samba_rw /main-pool/yolo

songgao in home in ~
❯ sudo chmod -R g+w /main-pool/trunk

songgao in home in ~
❯ sudo chmod -R g+w /main-pool/yolo
```

# Install and Configure Samba

```bash
root@samba-server:~# apt update && apt install samba
```

In `/etc/samba/smb.conf`, disable `obey pam restrictions`. This is necessary to
make group write permission work. Changing the `umask` of users might work too
but this seems simpler. The line should look like this:

```bash
obey pam restrictions = no
```

Add this to the bottom of `/etc/samba/smb.conf`:

```
[trunk]
  comment = main-pool/trunk
  path = /mnt/main-pool-trunk
  read only = yes
  browsable = yes
  write list = @samba_rw
  create mask = 664
  directory mask = 775

[yolo]
  comment = main-pool/yolo
  path = /mnt/main-pool-yolo
  read only = yes
  browsable = yes
  write list = @samba_rw
  create mask = 664
  directory mask = 775
```

Configure samba users:

```bash
root@samba-server:~# smbpasswd -a samba_songgao
root@samba-server:~# smbpasswd -a samba_xinyuzhao
```

Allow samba traffic in firewall:

```bash
root@samba-server:~# ufw allow samba
```

Start the SMB server:

```bash
root@samba-server:~# service smbd start
```
