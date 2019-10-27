---
layout: post
status: publish
published: true
title: Optimizing MySQL and Apache for a low-memory VPS.
author: Travis Mick
date: 2016-10-17
---

## Diagnosing the problem.

My last post had a plug about the migration of our Wordpress instance to a new server. However, it didn't go completely smoothly. The site had gone down a few times in the first day after the migration, with Wordpress throwing "Error establishing a database connection." Sure enough, MySQL had gone down. A simple restart of MySQL would bring the site back up, but what caused the crash in the first place?

<!-- more -->

A peek into `/var/log/mysql/error.log` yielded this:

```
2016-10-12T21:20:50.588667Z 0 [ERROR] InnoDB: mmap(137428992 bytes) failed; errno 12
2016-10-12T21:20:50.588702Z 0 [ERROR] InnoDB: Cannot allocate memory for the buffer pool
2016-10-12T21:20:50.588728Z 0 [ERROR] InnoDB: Plugin initialization aborted with error Generic error
2016-10-12T21:20:50.588749Z 0 [ERROR] Plugin 'InnoDB' init function returned error.
2016-10-12T21:20:50.588758Z 0 [ERROR] Plugin 'InnoDB' registration as a STORAGE ENGINE failed.
2016-10-12T21:20:50.588767Z 0 [ERROR] Failed to initialize plugins.
2016-10-12T21:20:50.588772Z 0 [ERROR] Aborting
```

So it looks like an out-of-memory error. This VPS only has 512 MB of RAM, so I wouldn't be surprised. Clearly, some tuning would be necessary. First, we'll reduce the size of MySQL's buffer pool, then shrink Apache's worker pool, and finally add a swapfile just in case memory pressure remains a problem.

## Optimizing MySQL.

The error that we saw was for the allocation of a buffer pool for InnoDB, one of MySQL's storage engines. We can see from the log that it's trying to allocate somewhere around 128 MB using `mmap`. This corresponds to the default value of the `innodb_buffer_pool_size` configuration option. Let's go ahead and trim this down to about 20 MB -- it'll reduce MySQL's performance, but we don't have much of a choice on a machine this small.

On Ubuntu, I put this option in `/etc/mysql/mysqld.conf.d/mysqld.cnf`:

```
innodb_buffer_pool_size = 20M
```

Issue `sudo service mysql restart`, and rejoice as MySQL no longer uses 25% of your RAM.

## Optimizing Apache.

Most of Apache's memory usage comes from the fact that it preemptively forks worker processes in order to handle requests with low latency. This is handled by the `mpm_prefork` module, and if enabled, its config can be found in `/etc/apache2/mods-enabled/mpm_prefork.conf` (on Ubuntu, at least).

By default, Apache will create 5 processes at startup, keep a minimum of 5 idle processes at all times, allow up to 10 idle processes before they're reaped, and spawn up to 256 processes at a time under load. Let's reduce these to something more sane given the constraints of our system:

```apache
<IfModule mpm_prefork_module>
 StartServers 3
 MinSpareServers 3
 MaxSpareServers 5
 MaxRequestWorkers 25
 MaxConnectionsPerChild 0
</IfModule>
```

Now, sudo service apache2 restart` and you're done.

## Creating a swapfile.

Most VPSes don't give you a swap partition by default, like you would probably create on a dedicated server or your desktop. We can create one using a file on an existing filesystem, in order to make sure there's extra virtual memory available in case our tuning doesn't handle everything perfectly.

First, let's pre-allocate space in the filesystem. We can do this using the `fallocate` command. I made a 2 GB swapfile:

```bash
$ sudo fallocate -l 2G /swapfile
```

Now, give it some sane permissions:

```bash
$ sudo chmod 600 /swapfile
```

Next, format the file so it looks like a swap filesystem:

```bash
$ sudo mkswap /swapfile
```

And finally, tell the OS to use it as swap space:

```bash
$ sudo swapon /swapfile
```

Now, we've got swap for the time being, but it won't persist when we reboot the system. To make it persist, simply add a line to `/etc/fstab`:

```
/swapfile none swap sw 0 0
```

Congratulations, you're now the proud owner of some swap space.

Hopefully this post will help anyone suffering stability problems with MySQL and Apache on a small VPS. I've adapted the instructions here from several sources, most notably [this StackOverflow post](http://stackoverflow.com/questions/12114746/mysqld-service-stops-once-a-day-on-ec2-server/12683951#12683951), and [this article from DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-add-swap-space-on-ubuntu-16-04).

