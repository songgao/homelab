# Pool Creation

💡 Pay attention to use disk IDs because by default ZoL uses the `/dev/sdx` format

💡 Explicitly use the `ashift=12` option since ZoL's detection of 4KB sector size [isn't reliable](https://wiki.archlinux.org/title/ZFS#Advanced_Format_disks)

Example:

```bash
sudo zpool create -o ashift=12 main-pool mirror ata-ST4000NE001-2MA101_WJG2AWQM ata-ST4000NE001-2MA101_WJG1XW8G
```

# Creating Datasets

💡 Don’t create an encrypted pool (root dataset), but create an encrypted dataset. Reason is sub-datasets of an encrypted dataset can’t be unencrypted. Also might wanna change encryption algorithm in the future.

```bash
sudo zfs create -o encryption=aes-256-gcm -o keyformat=passphrase main-pool/kbfs-backup-sink
sudo zfs create -o encryption=aes-256-gcm -o keyformat=passphrase main-pool/trunk
sudo zfs create -o encryption=aes-256-gcm -o keyformat=passphrase main-pool/yolo
```

`main-pool/kbfs-backup-sink`: The dataset hosting restic repos for KBFS backup. restic already encrypts but we might add notes or logs so we might as well make the dataset itself encrypted too.

`main-pool/trunk`: The main dataset for important data. This dataset is backed up to a cloud storage. In case this pool fails, we can recover the data. But it also means data living here incurs a monthly cost.

`main-pool/yolo`: This dataset is not backed up anywhere. This pool has redundancy so it's still somewhat robust, but if the server gets destroyed in e.g. an earthquake, this dataset will be gone.

# Useful Dataset Commands

After each reboot, ZFS dataset encryption key(s) need to be reloaded. For example:

```bash
sudo zfs load-key main-pool/trunk
```

Listing dataset status with some custom fields:

```bash
sudo zfs list -o name,avail,used,usedchild,mountpoint,mounted,encryption,keystatus -t filesystem,volume
```

# Some Performance Observations

Write throughout, sync:

- Encrypted, no SLOG: 132 MB/s
- Encrypted, w/ SLOG: 365 MB/s
- Unencrypted seems marginally slower??
