---
layout: post
title: How to reset APIC node without admin password
tags: [sdn, network, TodayILearned]
category: [network, cisco]
date: 2023-11-01 07:50 +0200
---

# How to reset standby APIC node without admin password

In case you want to reset an CISCO APIC node but you don't have the admin password you can use the special rescue-user.

Note: this works for an standby APIC node. If the node was part of the fabric the ‘rescue-user’ login account will utilize the same password that was previously set for the admin account.

## Access the node

You will need physical access (console) or CIMC-KVM / CIMC-SOL to the device.

## Wipe & reset the device config

```shell

acidiag touch clean
This command will wipe out this device. Proceed? [y/N] y
```

```shell
acidiag touch setup
This command will reset the device configuration, Proceed? [y/N] y

```

