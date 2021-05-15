# Basics About my Home Server

## Hardware

Used *real* servers are really cool with fancy parts that just fit together
right in and optimized airflow, etc. They also tend to be more reliable.
However, they rarely yield best bang for the buck, and are super power hungry
for the performance they provide.

So I ended up just putting together a PC using consumer parts. There's no
server grade hard drive or ECC RAM, but it's good enough for my needs for now.
Here's the parts list:

| Component    | Product                                              | Count |
|--------------|------------------------------------------------------|-------|
| CPU          | AMD Ryzen 3700X                                      | 1     |
| Motherboard  | ASRock B550M Pro4 M-ATX                              | 1     |
| Memory       | G.Skill Ripjaws V 32 GB (2x16GB) DDR4-3600 CL16      | 1     |
| Storage      | Seagate IronWolf Pro 4 TB 3.5" 7200RPM               | 2     |
|              | Seagate IronWolf Pro 125 SSD 240GB NAS 2.5" SATA     | 1     |
|              | Samsung SSD 970 EVO 250GB                            | 1     |
| Case         | Fractal Design Node 804 M-ATX                        | 1     |
| PSU          | Fractal Design Ion+ 560 W 80+ Platinum Fully Modular | 1     |
| Graphic Crad | ZOTAC GeForce GT 710 ZT-71304-20L                    | 1     |

Wanted to get a Ryzen 5000 series CPU but it was out of stock everywhere when I
bought this.

ECC RAMs are still super expensive and I ended up skipping them.

The graphic card is just for installing the OS, etc. I didn't have a card
around so I just got the cheapest 1x card I could find.

I thought about getting a MB with 2.5GbE but it seems the concensus is their
quality is not consistent where most brands use crappy chips. Linux support for
these newer chips also doesn't seem great. Plus that I don't have any 2.5Gbe
networking gears so I ended up just sticking with 1GbE and planning on adding a
PCI-E card for 2.5GbE in the future if I need.

The two IronWolf HDDs are to be configured as ZFS mirros, and the IronWolf SSD
is used as ZIL SLOG for the pool. The Samsung SSD is used to run the operating
system and hold misc small stuff.

## Operating System

Proxmox, unlicensed.

# More

## Storage

[ZFS setup](zfs.md)

## Backups

[KBFS Backup](backup-kbfs.md)

## Services

[Samba](samba.md)
