---
layout: post
status: publish
published: true
title: Roll your own dynamic DNS (Ubuntu)
author: Travis Mick
date: 2014-07-24
---

Dynamic DNS, or DDNS, is a type of DNS configuration which allows hosts with dynamic IP addresses to automatically update their DNS records. Often users will rely on services such as DynDNS or No-IP to manage this type of setup, but it is actually relatively easy to run your own DDNS server. Of course, this requires that you have your own domain name and access to at least one host with a static IP (to use as the DNS server).

<!-- more -->

## Step 1: Set up DNS records for your dynamic subdomain

I do not recommend hosting all of your DNS records yourself. I manage most of my records on Cloudflare, but delegate my dynamic subdomain to my own nameserver. In this step, you'll need to decide two things: what subdomain you want your dynamic hosts to live on, and what server you will host their DNS records on.

Let's assume you'll use ddns.example.com as your dynamic subdomain, and ns1.example.com as the nameserver. With this setup you will need to add a NS record delegating ddns.example.com to ns1.example.com. Then your dynamic host will be named, for example, host1.ddns.example.com. However, you can also add a CNAME pointing ddns.example.com to host1.ddns.example.com, allowing you to use a shorter name to talk to the machine.

## Step 2: Install and configure your nameserver

Now, on ns1.example.com, install the bind9 package (as it is named on Ubuntu). Edit /etc/bind/named.conf.local and add the following:

```
zone "ddns.example.com" {
  type master;
  file "/var/lib/bind/ddns.example.com";
  update-policy {
    grant *.ddns.example.com self ddns.example.com.;
  };
};
```

This just allows your the hosts on your ddns subdomain to update their own records (after they authenticate themselves). Now to generate the key for authentication:

```bash
$ ddns-confgen -r /dev/urandom -q -a hmac-md5 -k host1.ddns.example.com \
  -s host1.ddns.example.com. | tee -a /etc/bind/ddns.example.com.keys   \
   > /etc/bind/key.host1.ddns.example.com
```

This can be repeated for however many hosts you need on the subdomain. Then copy the key.*.ddns.example.com files to their respective hosts, and delete them off the nameserver. Now, we have to make sure to set secure permissions on the nameserver's copy of the keys:

```bash
$ chown root:bind /etc/bind/ddns.example.com.keys
$ chmod u=rw,g=r,o= /etc/bind/ddns.example.com.keys
```

Now, add a line to /etc/bind/named.conf.local to use the keys you just created

```
include "/etc/bind/ddns.example.com.keys";
```

The server is now configured, but we still have to create a zonefile which will store the DNS records.

## Step 3: Create the zonefile

Create /var/lib/bind/ddns.example.com and add the following information:

```
$ORIGIN .
$TTL 300             ; 5 minutes
ddns.example.com IN SOA  ns1.example.com. root.example.com. (
          1          ; serial
          3600       ; refresh (1 hour)
          600        ; retry (10 minutes)
          604800     ; expire (1 week)
          300        ; minimum (5 minutes)
          )
NS      ns1.example.com.
$ORIGIN ddns.example.com.
$TTL 60 ; 1 minute
```

That's it. Now start up bind and we'll move to the dynamic host to configure it.

## Step 4: Configure the dynamic host

First, let's try to manually write a record to server. Write a simple text file with this content:

```
server ns1.example.com
zone ddns.example.com.
update delete host1.ddns.example.com.
update add host1.ddns.example.com. 60 A 192.168.0.1
update add host1.ddns.example.com. 60 TXT "Updated on $(date)"
send
```

Now, pipe it into nsupdate:

```bash
$ nsupdate -k /path/to/key.host1.ddns.example.com < file_you_just_wrote.txt
```

If it worked, then everything is set up correctly! However, it would be nice to have the IP update automatically, right? Fortunately, there's a script that can do that! It just queries OpenDNS for your IP address, then sends the update to your nameserver. Just set up a cronjob (for example, every 15 minutes) to run it, and your DDNS will be completely automated. *(Sorry, the site I was hosting the script on was no longer active, otherwise there would be a link right here.)*

