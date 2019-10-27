---
layout: post
status: publish
published: true
title: My first adventure with Let's Encrypt on nginx, dovecot, and postfix
author: Travis Mick
date: 2016-03-28
---

[Let's Encrypt](https://letsencrypt.org) is old news by now. It launched back in December, so it has been giving away free DV certificates for nearly four months now. Being a TA for a Computer Security course, it's about time that I actually tried it out.

## Prologue

Let's Encrypt is a free certificate authority. They grant TLS certificates that you can use to secure your webserver. They are Domain Validated (DV) certificates, which means they will verify that you control the domain name you are trying to certify.

<!-- more -->

To test out Let's Encrypt, I decided to deploy new certificates on some internal Zeall.us services (Gitlab and Roundcube). Previously, we were using self-signed certs, so we would have to approve a security exception on every visit.

Unfortunately, I chose the wrong day to test out Let's Encrypt. For some reason, their DNS client decided to die at the very moment I tried to get a certificate. I [wasn't the only one having this problem](https://community.letsencrypt.org/t/dns-problem-query-timed-out-looking-up-a/13354/2), but Google hadn't yet crawled the support site so I didn't see these other reports until the problem had already subsided. Once it cleared up, everything else was a breeze.

## Obtaining certificates

Obtaining certificates for our Gitlab and webmail instances was incredibly easy. I just cloned the [Let's Encrypt repository](https://github.com/letsencrypt/letsencrypt) from Github and used their super-simple command-line utility. It doesn't have built-in support for nginx (our webserver of choice), but you can still easily use the webroot plugin to validate your domain. You just tell it the directory from which your webserver serves, and it puts a file there which it uses for the validation. I ran it once for each of the services:

```bash
$ ./letsencrypt-auto certonly --email example@example.com --text --agree-tos --renew-by-default --webroot -w /opt/gitlab/embedded/service/gitlab-rails/public -d <our_gitlab_domain>
$ ./letsencrypt-auto certonly --email example@example.com --text --agree-tos --renew-by-default --webroot -w /srv/roundcube -d <our_webmail_domain>
```

It handles everything automatically, and sticks your new certificates and keys in their respective /etc/letsencrypt/live/\<domain_name\>/ directories.

## Configuring nginx

Prior to setting up the new certificates in nginx, I generated strong Diffie-Hellman key exchange parameters, as suggested by [this guide to nginx hardening](https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html):

```bash
$ sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 4096
```

Then I just put the following lines in each of my nginx site configs:

```
ssl on;
ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
ssl_session_timeout 5m;
ssl_session_cache shared:SSL:10m;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';
ssl_prefer_server_ciphers on;
ssl_dhparam /etc/ssl/certs/dhparam.pem;
```

Follow up with a `sudo service nginx reload`, and you're good to go. With this configuration, my domain got an A+ from [Qualys' SSL Labs Server Test](https://www.ssllabs.com/ssltest/).

## Configuring Dovecot

To supply the new certificate to my Dovecot IMAP server, I set the following configs in `/etc/dovecot/conf.d/10-ssl.conf`:

```
ssl_cert = </etc/letsencrypt/live/<domain>/fullchain.pem
ssl_key = </etc/letsencrypt/live/<domain>/privkey.pem
```

This is the same certificate I used for webmail, since the domain is the same. I have not yet investigated hardening techniques for Dovecot, but at a glance the settings seem to be quite similar to nginx.

This configuration was followed up with a `sudo service dovecot restart`, and then everything worked.

## Configuring Postfix

Finally, I supplied the same certificate to Postfix (SMTP) by putting the following in `/etc/postfix/main.cf`:

```
smtpd_tls_cert_file = /etc/letsencrypt/live/<domain>/fullchain.pem
smtpd_tls_key_file = /etc/letsencrypt/live/<domain>/privkey.pem
```

After a `sudo service postfix restart`, the certificates are live.

## Auto-renew

Let's Encrypt provides certificates with a term of only 90 days, so it is necessary to write a script to renew them automatically. I wrote the following script:

```bash
/opt/letsencrypt/letsencrypt-auto renew | tee -a ./lerenew.log
service nginx reload
service postfix reload
service dovecot reload
```

And a corresponding crontab:

```
0 0 * * 0 ~/lerenew.sh
```

Now once a week, any eligible certificates will be renewed and the services which use them will be reloaded.

## Conclusion

It has never been this easy to deploy TLS certificates. I highly recommend that anyone who is currently using a self-signed certificate switch to Let's Encrypt. If you're currently paying for a DV certificate, consider switching to Let's Encrypt to save some money -- but keep compatibility issues in mind.

