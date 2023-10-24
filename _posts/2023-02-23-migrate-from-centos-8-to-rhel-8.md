---
layout: post
title: Migrate from CentOS 8 to RHEL 8
tags: [linux, CentOS, RHEL]
category: [linux, TIL]
date: 2023-02-20 10:04 +0200
---

# Migrate from CentOS 8 to RHEL 8

CentOS Linux 8 has reached end of life but can be migrated to an alternative RPM distribution. In this case to Red Hat Enterprise Linux (RHEL).

Based on the [RedHat article](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html-single/converting_from_an_rpm-based_linux_distribution_to_rhel/index?extIdCarryOver=true&sc_cid=701f2000001OH7EAAW "Converting from an RPM-based Linux distribution to RHEL"), the conversion should be simple. Make sure that you read that article, follow the prerequisites, and make a backup to your system.

### Install Convert 2 RHEL utility

```shell
sed -i 's/^mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=https://vault.centos.org|g' /etc/yum.repos.d/CentOS-*

curl -o /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release https://www.redhat.com/security/data/fd431d51.txt
curl --create-dirs -o /etc/rhsm/ca/redhat-uep.pem https://ftp.redhat.com/redhat/convert2rhel/redhat-uep.pem
curl -o /etc/yum.repos.d/convert2rhel.repo https://ftp.redhat.com/redhat/convert2rhel/<version_number>/convert2rhel.repo
yum -y install convert2rhel
```

### Fix  ```CRITICAL - No packages containing kernel modules available```

Although you can add credentials to the RHSM as arguments/config file or type them when prompted, for some reason, the Convert2RHEL utility exited with an error saying that it can not find kernel modules in the repositories.
![Screenshot 2023-02-20 at 20.40.24](/assets/img/posts/Error-no-kernel.png)

The solution was to add a RedHat subscription before converting it to the RHEL.

```shell
yum install subscription-manager

subscription-manager register
subscription-manager attach --pool xxxxxxxx

```

### Convert CentOS 8 operating system to RedHat Enterprise Linux 8  operating system (RHEL 8)

```shell
convert2rhel --keep-rhsm
```

After installing the subscription manager and attaching a subscription, I was able to convert successfully from CentOS 8 to RHEL 8.

![convert2rhel-success](/assets/img/posts/convert2rhel-success.png)

### Note for systems with RAID configured.

You might need to reinstall grub again if you have a software RAID in place because the utility might install the grub on only one disc.
