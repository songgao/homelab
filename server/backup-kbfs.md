A restic repo lives under `/main-pool/kbfs-backup-sink` for each KBFS TLF to be
backed up. There are multiple ways to read from KBFS but on Linux, FUSE seems
to make the most sense here considering the data needs to be fed into restic as
well. Here `keybase`, `kbfsfuse`, and the `restic` binary all run inside a LXC
container.

# Container Setup

Stuff that syncs KBFS TLFs into `/main-pool/kbfs-backup-sink` live inside a LXC
container. 

The container is created using Proxmox web UI, with ID 1001 and name
`kbfs-backup-syncer`. Note that here 1001 is just an arbitrary container ID,
and has nothing to do with the UID 1001 mentioned below. The container uses
bridged networking and gets an IP in the same subnet as the host. It runs
Ubuntu 20.04. Locale is set to `en_US.UTF-8`.

## Mounting the Dataset

The dataset directory needs to be bound manually into the LXC container:

```bash
root@kbfs-backup-syncer:~# mkdir /mnt/main-pool-kbfsbackup-sink
```

```bash
songgao in home in ~
❯ sudo pct set 1001 -mp0 /main-pool/kbfs-backup-sink,mp=/mnt/main-pool-kbfs-backup-sink
```

## User Creation and UID Mapping

We
[need](https://forum.proxmox.com/threads/lxc-container-bind-mount-map-all-uid-gid-to-a-single-uid-gid-on-the-host.64817/post-293034)
to change the owner of `/main-pool/kbfs-backup-sink` on host so the LXC has the
permission to access it. The default mapping is that UID in a LXC maps to
+100000 on host. So for example root (UID 0) in LXC becomes 100000 on host.
However that's not ideal. So we'll create a user in both host and the LXC and
manually map them.

```bash
songgao in home in ~ took 1m9s
❯ sudo useradd kbfsbot

songgao in home in ~
❯ cat /etc/passwd | grep kbfsbot
kbfsbot:x:1001:1001::/home/kbfsbot:/bin/sh
```

Edit `/etc/pve/nodes/home/lxc/1001.conf` on host to make the uid/gid mapping, with special treatment for `kbfsbot` (see doc [here](https://pve.proxmox.com/wiki/Unprivileged_LXC_containers)):

```
# uid map: from uid 0 map 1001 uids (in the ct) to the range starting 100000 (on the host), so 0..1000 (ct) → 100000..101000 (host)
lxc.idmap = u 0 100000 1001
lxc.idmap = g 0 100000 1001
# we map 1 uid starting from uid 1001 onto 1001, so 1001 → 1001
lxc.idmap = u 1001 1001 1
lxc.idmap = g 1001 1001 1
# we map the rest of 65534 from 1001 upto 101001, so 1001..65535 → 101001..165535
lxc.idmap = u 1002 101002 64534
lxc.idmap = g 1002 101002 64534
```

And enable `subuid` and `subgid` from `root` to `1001` by adding lines to following two files:

```
# /etc/subuid
root:1001:1

# /etc/subgid
root:1001:1
```

Then create the user in the LXC and confirm sure uid/gid are what we mapped. 

```bash
root@kbfs-backup-syncer:~# useradd -m kbfsbot
root@kbfs-backup-syncer:~# cat /etc/passwd | grep kbfsbot
kbfsbot:x:1001:1001::/home/kbfsbot:/bin/bash
```

Note that if `kbfsbot`'s user home dir was created before we did the mapping,
it'd become `nobody` after the mapping and the only way to fix it seems to be
1) reverse the mapping; 2) in LXC `chown -R` it to `root`; 3) redo the mapping;
4) `chown -R` it back to the (correctly mapped) `kbfsbot`. A more deterministic
way to do this is to just manually specify UID when creating the users (see
[samba.md](samba.md)).

Lastly we need to change the dataset to be owned by `kbfsbot:kbfsbot`:

```bash
songgao in home in ~
❯ sudo chown -R kbfsbot:kbfsbot /main-pool/kbfs-backup-sink
```

## FUSE Device

We need to make `/dev/fuse` available to the LXC. Without making the container
privileged, we can achieve this
([ref1](https://www.reddit.com/r/Proxmox/comments/bifi27/can_not_create_tun_device_in_lxc_container/em20chd/),
[ref2](https://wiki.ubuntu.com/FuseUserns)) by adding this line to
`/etc/pve/lxc/1001.conf`:

```
lxc.mount.entry: /dev/fuse dev/fuse none bind,create=file 0 0
```

## Packages

Install packages: `jq`, `restic`,

# Keybase Setup

Install the `.deb` package as `root`.

Keybase services need to run as a user other than `root`, for several reasons:
1. `run_keybase` doesn't work as `root`. 2. Although `KEYBASE_ALLOW_ROOT` makes
it run, KBFS mounting will fail. As a result it's run as `kbfsbot`.

In this container we're logged in as a bot `songgao_kbfsbot`, which is added as
a `reader` to teams we need to backup.

# Restic Setup

We make a directory for each team folder that needs to be backed up, and the
`restic` repo lives inside that folder along with any notes or logs. For each
such folder, we manually initialize the `restic` repo:

```bash
restic init --repo /mnt/main-pool-kbfs-backup-sink/gaos.docs/repo
```

Additionally, we make a `_logs` directory for logging.

Then we have a script to take care of running `restic`. *(TODO: make it print
progress to TTY while still logging summary to the log file once
[this](https://github.com/restic/restic/issues/1455) is resolved.)*

```bash
#!/bin/bash

set -e

ts=$(date --rfc-3339=seconds | sed 's/ /T/')
log="/mnt/main-pool-kbfs-backup-sink/_logs/${ts}.log"
echo "log file: $log"

tlfs="gaos.docs"

for tlf in $tlfs; do
  tlf_path="/keybase/team/$tlf"

  echo                                                         | tee -a "$log"
  echo "====== $tlf_path ======"                               | tee -a "$log"

  cd "/mnt/main-pool-kbfs-backup-sink/$tlf"
  echo "- working directory: $(pwd)"                           | tee -a "$log"

  rev=$(cat "$tlf_path/.kbfs_status" | jq ".Revision")
  echo "- current revision: $rev"                              | tee -a "$log"

  repo="$(pwd)/repo"
  echo "- restic repo: $repo"                                  | tee -a "$log"

  echo "- starting restic backup ..."                          | tee -a "$log"

  restic -r "$repo" backup --tag "rev:$rev" "$tlf_path" 2>&1   | tee -a "$log"

  echo "- backup done!"                                        | tee -a "$log"

  rev=$(cat "$tlf_path/.kbfs_status" | jq ".Revision")
  echo "- current revision: $rev"                              | tee -a "$log"
done
```
