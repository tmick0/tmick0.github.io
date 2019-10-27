---
layout: post
status: publish
published: true
title: Quick postfix & dovecot config with virtual hosts (Ubuntu 16.04)
author: Travis Mick
date: 2016-08-23
---

This morning, I received an email from my VPS host notifying me that they will no longer accept PayPal. Instead, my only payment option would be Bitcoin. Not willing to go through this trouble, I decided to migrate from this host (which I had been using for my personal servers for about five years now) to DigitalOcean (which fortunately accepts normal forms of payment).

Part of my server migration was to move email for two of my domains: le1.ca and lo.calho.st. Setting up a new mailserver is a notoriously arduous task, so I'm documenting the process in this post -- mostly for my future reference, but also to benefit anyone who might stumble upon my blog in their own confusion.

Since I'm serving mail for two domains, I will be using a simple "virtual hosts" configuration. I'll talk about the process in four parts: local setup, postfix, dovecot, and DNS configuration.

<!-- more -->

## Initial Setup

We must prepare a few things before we're ready to configure the actual mail services. The first thing I'd like to do is install a SSL certificate, so that we can access the mail server securely. This is easy with Let's Encrypt:

```bash
$ sudo apt-get install letsencrypt # (if you don't have it installed already)
$ letsencrypt certonly --standalone -d mail.le1.ca -d mail.lo.calho.st
```

I'm using the standalone option here, because I do not have an HTTP daemon installed on this server. If you're also serving HTTP from your mail machine, you will need to [follow the appropriate instructions for your configuration](https://certbot.eff.org/).

Now that we have an SSL cert, we can set up the directory structure for our virtual mailboxes. I'm going to make the root for this environment `/var/spool/mail`, and in that directory, I'm going to create one folder for each domain I'm serving:

```bash
$ mkdir -p /var/spool/mail/le1.ca
$ mkdir -p /var/spool/mail/lo.calho.st
```

The next step is to configure passwords for each of your users. I'm doing this the easy way, since I'm the only person using the mail server. If you need users to be able to change their own passwords, this is going to be more difficult; you'll need to find an alternate authentication configuration.

You will need to calculate a password hash for each user you want to serve:

```bash
$ doveadm pw -u foo@le1.ca
```

Each time, it should spit out a hash that looks like this:

```
{CRAM-MD5}a7cb902940b3f6662c48ace840a4e3e410241e875d720cb45b2d95a3e1ddfc8b
```

For each of your domains, create a `shadow` file in the corresponding `/var/spool/mail/*` directory. For example, I will create `/var/spool/mail/le1.ca/shadow`. Add a `username:password` line for each virtual user you want to create.

```
foo@le1.ca:{CRAM-MD5}a7cb902940b3f6662c48ace840a4e3e410241e875d720cb45b2d95a3e1ddfc8
```

Next, let's create a "virtual mail" user and configure him as the owner of these mail directories.

```bash
$ sudo adduser vmail
$ sudo chown -R vmail:vmail /var/spool/mail/
```

Find out the UID and GID of this new user and remember it for later. An easy way to do this is `grep vmail /etc/passwd`. The UID and GID are the third and fourth columns, respectively.

That's it for initial setup. Now let's deal with postfix.

## Configuring Postfix

Postfix is responsible for delivering received mail to the correct locations on our server, as well as sending mail on our behalf.

There's some initial configuration we can do interactively before we start poking around in config files. If you haven't installed postfix yet, this configuration will start automatically when you do `apt-get install postfix`. If you do already have it installed, then you can run `dpkg-reconfigure postfix` to get there.

Here are the responses you should enter to this wizard:

* **Type of mail configuration:** Internet Site
* **System mail name:** Use your "primary" mail domain here. I used "le1.ca"
* **Root and postmaster recipient:** Enter your primary email address here. It will be used as the destination for administrative mail.
* **Destinations to accept mail for:** Enter any aliases of your machine here, but *don't enter the domains you're going to use in your virtual mailboxes.* A good list is something like `my-hostname, localhost.localdomain, localhost`
* **Everything else:** Leave it at the default value

The main configuration file for postfix is `/etc/postfix/main.cf`. Let's break down the config into three sections: SSL, virtual mailboxes, and SASL. I'm only including lines that need to be modified here. Everything else should stay at its default value.

First, SSL-related configs:

```
smtpd_tls_cert_file=/etc/letsencrypt/live/[your_mailserver_name]</strong>/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/[your_mailserver_name]</strong>/privkey.pem
smtpd_use_tls=yes
```

Virtual mailbox configs:

```
virtual_mailbox_domains = /etc/postfix/virtual_domains
virtual_mailbox_maps = hash:/etc/postfix/virtual_mailbox
virtual_mailbox_base = /var/spool/mail
virtual_uid_maps = static:1001 # Replace with the UID of your vmail user
virtual_gid_maps = static:1001 # Replace with the GID of your vmail user
```

SASL configs (to pass authentication duties to Dovecot):

```
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_security_options = noanonymous
smtpd_sasl_local_domain =
smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
smtpd_recipient_restrictions = permit_sasl_authenticated, permit_mynetworks, reject_unauth_destination
```

Hopefully you noticed that the virtual mailbox configs refer to some new files. We're going to have to create `/etc/postfix/virtual_domains` and `/etc/postfix/virtual_mailbox` in order to create the mappings for our domains.

The `virtual_domains` file is easy, just a list of the domains you're serving. In my case:

```
le1.ca
lo.calho.st
```

The `virtual_mailbox` file is also pretty easy. It just maps email addresses to mailbox folders, relative to the `virtual_mailbox_base` directory. For example, if I want both foo@le1.ca and bar@le1.ca to go to the mailbox `le1.ca/foobar`, and baz@lo.calho.st to go to `lo.calho.st/baz`:

```
foo@le1.ca le1.ca/foobar/
bar@le1.ca le1.ca/foobar/
baz@lo.calho.st lo.calho.st/baz/
```

You can create as many aliases as you want here. Whatever is on the right will end up determining your login username (in this case, foobar@le1.ca and baz@lo.calho.st are my usernames), and the addresses on the left create aliases.

Postfix now requires us to hash this map file. Run the following command from within `/etc/postfix`:

```bash
$ postmap virtual_mailbox
```

This should create a file called `virtual_mailbox.db`.

The last part of the postfix config is to enable the "user-facing" SMTP port, 587. Just paste this block in `master.cf` somewhere:

```
submission inet n - y - - smtpd
-o smtpd_tls_security_level=encrypt
-o smtpd_sasl_auth_enable=yes
-o smtpd_sasl_type=dovecot
-o smtpd_sasl_path=private/auth
-o smtpd_sasl_security_options=noanonymous
-o smtpd_sasl_local_domain=$myhostname
-o smtpd_client_restrictions=permit_sasl_authenticated,reject
-o smtpd_sender_login_maps=hash:/etc/postfix/virtual_mailbox
-o smtpd_recipient_restrictions=reject_non_fqdn_recipient,reject_unknown_recipient_domain,permit_sasl_authenticated,reject
```

The next step is to configure dovecot.

## Configuring Dovecot

Dovecot is much easier to configure than postfix. Create `/etc/dovecot/dovecot.conf` with the following content (delete it if it already exists):

```
protocols = imap
log_path = /var/log/dovecot.log

ssl = required
ssl_cert = </etc/letsencrypt/live/[your_mailserver_name]/fullchain.pem
ssl_key = </etc/letsencrypt/live/[your_mailserver_name]/privkey.pem
ssl_dh_parameters_length=2048
ssl_cipher_list = ALL:!LOW:!SSLv2:!EXP:!aNULL

disable_plaintext_auth = no
mail_location = maildir:/var/spool/mail/%d/%n

auth_verbose = yes
auth_mechanisms = plain login

passdb {
 driver = passwd-file
 args = /var/spool/mail/%d/shadow
}

userdb {
 driver = static
 args = uid=vmail gid=vmail home=/var/spool/mail/%d/%n
}

protocol lda {
 postmaster_address = root@le1.ca
}

service auth {
 unix_listener /var/spool/postfix/private/auth {
 mode = 0660
 user = postfix
 group = postfix
 }
}
```

Now you should be ready to restart postfix and dovecot:

```bash
$ sudo service dovecot restart
$ sudo service postfix restart
```

If everything went smoothly, we're ready to configure DNS records.

## DNS Records

For each of your domains, you will need three DNS records:

* Either a CNAME or an A record pointing to the mailserver
* An MX record designating the mailserver as the destination for the domain's mail
* A SPF record designating the server as a permitted origin for mail

The A record is easy. Just create an A record for "mail.yourdomain.com" pointing to the IP address of the server.

The MX record might be even easier: set the mailserver for "yourdomain.com" to "mail.yourdomain.com"

The SPF record is also very straightforward. Many people overlook its importance, but without it, mail from your domain is likely to go into recipients' spam folders. There exist [wizards to create custom SPF records](http://www.spfwizard.net/), but I find the one below to be sufficient for most purposes. Just insert this as a TXT record for "yourdomain.com":

```
"v=spf1 mx a ptr ~all"
```

And that's it. Allow a few minutes for your DNS records to propagate, then test out your new mailserver.

