---
layout: post
title: How to completely wipe osd Ceph disk
tags: [linux, ceph, storage, TodayILearned]
category: [linux, ceph]
date: 2023-10-23 20:50 +0200
---

# How to completely wipe osd Ceph disk

In case you want to reuse a storage device previously used as an ODS device, you may need to wipe it.  It should be enough to delete the LVM, but sometimes, you may need to remove the mapper.

## List and remove LV, VG and PV
```shell
lvscan 

...
  ACTIVE            '/dev/ceph-0d729569-60f4-421a-9051-19f2adc47590/osd-block-27a28ec4-42b9-4e87-9c7c-498f5d01389d' [<9.10 TiB] inherit
...
```

```shell
lvremove osd-block-27a28ec4-42b9-4e87-9c7c-498f5d01389d

Logical volume "osd-block-27a28ec4-42b9-4e87-9c7c-498f5d01389d" successfully removed

```

```shell
vgscan 

...
  Found volume group "ceph-0d729569-60f4-421a-9051-19f2adc47590" using metadata type lvm2
...
```

```shell
vgremove ceph-0d729569-60f4-421a-9051-19f2adc47590

Volume group "ceph-0d729569-60f4-421a-9051-19f2adc47590" successfully removed

```

```shell
pvscan 

...
  PV /dev/sda       VG ceph-0d729569-60f4-421a-9051-19f2adc47590   lvm2 [<9.10 TiB / <9.10 TiB free]
...
```


```shell
pvremove /dev/sda

Labels on physical volume "/dev/sdc" successfully wiped.

```

If you want to remove all the LVM available, you can use some for loops.\
>**Note**: make sure that you trully want to remove all LVMs.
{: .prompt-warning }
```shell
yes | for i in $(lvscan | cut -f 2,3 -d "/"); do lvremove -f /$i;done
yes | for i in $(vgscan | cut -f 6 -d' '| cut -f 2 -d'"' | grep ceph); do vgremove $i;done
yes | for i in $(pvscan | grep -vE "/dev/md|Total" | cut -f 4 -d' '); do pvremove $i;done
```

## Zap device

```shell
sgdisk --zap-all /dev/sda

Creating new GPT entries in memory.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.

```

## Delete mappers
Although the deletion of the LVM should delete the mapper, sometimes this is not the case:
```shell
lsblk

...
sda                                                                                                     8:0    0   9.1T  0 disk  
└─ceph--4a380bf8--ce7c--40e9--be55--cfb33c5236f2-osd--block--f3eeb397--811b--4f35--ad8b--f3299e420d9a 253:0    0   9.1T  0 lvm   
...

```

```shell
dmsetup info

...
Name:              ceph--0d729569--60f4--421a--9051--19f2adc47590-osd--block--27a28ec4--42b9--4e87--9c7c--498f5d01389d
State:             ACTIVE
Read Ahead:        256
Tables present:    LIVE
Open count:        0
Event number:      0
Major, minor:      253, 0
Number of targets: 1
UUID: LVM-wIoaoKgJWag0EcnCqLSBdNi3oKSxUlcCn15IsbldS8zSCCtaKfwQ6vupSODrzoUe
...
```

For the device mapper deletions, one can use the dmsetup tool.
```shell
dmsetup remove /dev/mapper/ceph--0d729569--60f4--421a--9051--19f2adc47590-osd--block--27a28ec4--42b9--4e87--9c7c--498f5d01389d

```

```shell
lsblk 

...
sda           8:0    0   9.1T  0 disk
...
```
