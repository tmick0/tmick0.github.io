---
layout: post
status: publish
published: true
title: Using bcache to back a SSD with a HDD on Ubuntu.
author: Travis Mick
date: 2017-01-06
---
Recently, another student asked me to set up a PostgreSQL instance that they could use for some data mining. I initially put the instance on a HDD, but the dataset was quite large and the import was incredibly slow. I installed the only SSD I had available (120 GB), and it sped up the import for the first few tables. However, this turned out to not be enough space.

I did not want to move the database permanently back to the HDD, as this would mean slow I/O. I also was not about to go buy another SSD. I had heard of bcache, a Linux kernel module that lets a SSD act as a cache for a larger HDD. This seemed like the most appropriate solution -- most of the data would fit in the SSD, but the backing HDD would be necessary for the rest of it. This article explains how to set up a bcache instance in this scenario. This tutorial is written for Ubuntu Desktop 16.04.1 (Xenial), but it likely applies to more recent versions as well as Ubuntu Server.

<!-- more -->

## Preparation

If you have any existing data on the SSD or HDD, back it up elsewhere. Remove any associated mounts from `/etc/fstab`.

If you don't already have it installed, you need to `sudo apt-get install bcache-tools`.

## Partitions

On my machine, the HDD is `/dev/sdb` and the SSD is `/dev/sdc`. With bcache, you can either use entire disks or individual partitions. In my case, I'm using just one HDD partition, `/dev/sdb5`, but allowing the entire SSD to be used. Note that the backing HDD or partition has to be at least as large as the caching SSD or partition.

Surely your setup is different, so replace `/dev/sdb5` with your HDD partition, and `/dev/sdc` with your caching SSD or partition.

I gave 250 GiB to `/dev/sdb5`; no partitioning is necessary on `/dev/sdc` if you are using the entire drive for caching.

You will need to remove any existing filesystem on both devices/partitions:

```bash
$ sudo wipefs -a /dev/sdc
$ sudo wipefs -a /dev/sdb5
```

This is necessary because bcache will refuse to instantiate if it looks like a filesystem already exists on the device.

## Creating the bcache

A bcache instance looks and acts like a regular block device; instead of being named `/dev/sdXX` like a disk partition, it will be named `/dev/bcacheX`. The bcache kernel module will handle the underlying hardware and magic of the bcache device, we just have to set it up once.

We will be using "writeback" cache mode to enhance write performance; note that this is less safe than the default "writethrough" mode. If you're worried about this, omit the `--writeback` flag. We will also enable the TRIM functionality of the SSD, to further enhance long-term write performance. If your SSD does not support TRIM, omit the `--discard` flag.

The bcache device can be optimized for your disk sector size and your SSD erase block size. In this case, my HDD has 4 KB sectors. I was unable to find the erase block size of my SSD, so I am using the default; however, if you know it you can append, for example, `--bucket 2M`, if your erase block size is 2 MB. Similarly, you should change the HDD sector size in this command if you know it, or remove the `--block 4k` flag if you don't.

```bash
$ sudo make-bcache -C /dev/sdc -B /dev/sdb5 --block 4k --discard --writeback
```

Now you should see that the device `/dev/bcache0` has been created.

## Create and mount the filesystem

Now, we need to format our new bcache device. I'll be using ext4.

```bash
$ sudo mkfs.ext4 /dev/bcache0
```

Now add the appropriate fstab line. I'll be mounting the bcache device on `/home/postgres` since that's where my PostgreSQL installation previously lived. Another good place for general use would be, for example, `/media/bcache`.

First, you will need to create an empty directory for the mountpoint:

```bash
$ sudo mkdir -p /home/postgres
```

Then, open `/etc/fstab` in your favorite editor (as root, of course) and append the corresponding line (altering the mount options as necessary):

```
/dev/bcache0 /home/postgres ext4 defaults,noatime,rw 0 0
```

Now, we can test that we can mount the new device:

```bash
$ sudo mount /home/postgres
```

The device and mount should both persist through reboot. You may copy any data back to the bcache device at this time.

## Acknowledgments

This article was adapted from the following resources:
* [https://wiki.ubuntu.com/ServerTeam/Bcache](https://wiki.ubuntu.com/ServerTeam/Bcache)
* [https://wiki.archlinux.org/index.php/Bcache](https://wiki.archlinux.org/index.php/Bcache)

