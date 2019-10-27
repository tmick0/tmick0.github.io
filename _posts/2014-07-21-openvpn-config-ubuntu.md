---
layout: post
status: publish
published: true
title: Quick and easy OpenVPN server config (Ubuntu)
author: Travis Mick
date: 2014-07-21
---

VPNs allow you to route your Internet traffic through an encrypted tunnel to a remote server, enhancing your privacy while online. Often a VPS is around the same price per month as a dedicated VPN service, but gives you much more freedom and utility as a poweruser. This post will overview a basic routed OpenVPN configuration on an Ubuntu machine.

<!-- more -->

## Step 1: Drivers and packages

First you need to enable TUN/TAP on your VPS. Usually your host will give you an option for this in your Control Panel. If not, you may have to file a support ticket. We will need TUN to emulate a new network device for the VPN to use. To check whether you have TUN enabled, do `sudo cat /dev/net/tun`. If you get the message "File descriptor in bad state," then TUN is working. If you get "No such device," then the TUN driver is missing.

Now we will need to install OpenVPN, as well as Easy-RSA which we will use for our PKI:

```bash
$ sudo apt-get install openvpn easy-rsa
```

## Step 2: PKI and keys

Set up the PKI environment and generate a CA certificate:

```bash
$ sudo -s # become root
$ mkdir /etc/openvpn/easy-rsa/
$ cp -r /usr/share/easy-rsa/* /etc/openvpn/easy-rsa/
$ editor /etc/openvpn/easy-rsa/vars # set up parameters for your CA
$ cd /etc/openvpn/easy-rsa/
$ source vars
$ ./clean-all
$ ./build-ca # the CA certificate is now built
```

Build a key and certificate for the server:

```bash
$ ./build-key-server SERVER_NAME_HERE
$ ./build-dh
$ cd keys/
$ cp SERVER_NAME_HERE.crt SERVER_NAME_HERE.key ca.crt dh1024.pem /etc/openvpn/ 
```

Make keys and certificates for each client, and transfer them to the remote hosts (for example using SCP):

```bash
$ ./build-key CLIENT_NAME
$ scp /etc/openvpn/ca.crt CLIENT_MACHINE: 
$ scp /etc/openvpn/easy-rsa/keys/CLIENT_NAME.key CLIENT_MACHINE:
$ scp /etc/openvpn/easy-rsa/keys/CLIENT_NAME.crt CLIENT_MACHINE:
```

## Step 3: Server config

Copy the default the server configuration:

```bash
$ cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/
$ gzip -d /etc/openvpn/server.conf.gz
```

Now edit /etc/openvpn/server.conf and set the correct filenames for the server certificates and keys:

```
ca ca.crt
cert SERVER_NAME.crt
key SERVER_NAME.key 
dh dh1024.pem
```

## Step 4: iptables

You may have to set up some routes and firewall rules to get your VPN to work as expected (using NAT):

```bash
$ /sbin/iptables -A FORWARD -o tun0 -j ACCEPT
$ /sbin/iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
$ /sbin/iptables -A FORWARD -s 10.8.0.0/24 -j ACCEPT
$ /sbin/iptables -A FORWARD -j REJECT
$ /sbin/iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o venet0:0 -j MASQUERADE
$ /sbin/iptables -t nat -A POSTROUTING -j SNAT --to-source YOUR_SERVER_IP_HERE
```

If you're using an iptables manager, you can easily add these rules to that service's configuration. You can also put them in a new initscript, in an on-boot cronjob, or some other script that gets run when the server boots.

## Step 5: Start the daemon and connect clients

Now just start OpenVPN:

```bash
$ service openvpn start
```

And then configure your client with the keys you scp'd. Everything should be peachy.

