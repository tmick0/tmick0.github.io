---
layout: post
status: publish
published: true
title: Tools to check for compromised Keepass passwords
author: Travis Mick
date: 2020-11-19
---

It's 2020, and by now everyone should know the basics of password hygiene: use strong passwords, don't reuse them across sites, etc. A password manager such as [Keepass](https://keepass.info/) is an essential tool for keeping track of all the passwords you inevitably end up with under this set of rules. 

Unfortunately, no matter how good your passwords are, there's always a chance that they'll get compromised. For example, a website might have a breach and someone can obtain your password regardless of how hard it would have been to guess. In cases like these, services like [Have I Been Pwned](https://haveibeenpwned.com) can let you know this has happened. However, if your password is leaked in something like a large [credential-stuffing](https://en.wikipedia.org/wiki/Credential_stuffing) dump, it might not be obvious which sites you may need to change your password on. 

<!-- more -->

For situations like this, Have I Been Pwned also provides [password lists](https://haveibeenpwned.com/Passwords) which contain the cryptographic hashes of all known compromised passwords. However, you still can't immediately identify which of your passwords have been breached.

For this purpose, I have written a set of [very simple tools](https://github.com/tmick0/kdbxtools) to dump hashes of your own Keepass-managed passwords and check them against the Have I Been Pwned list.

Here's an example using a database that only contains Keepass's builtin example passwords.

The first script simply dumps a CSV of your password hashes and some metadata:

```
$ kdbx2csv ~/Desktop/Database.kdbx my_passwords.csv
Password: ****
$ cat my_passwords.csv 
8BE3C943B1609FFFBFC51AAD666D0A04ADF83C9D,User Name,Sample Entry
8CB2237D0679CA88DB6464EAC60DA96345513964,Michael321,Sample Entry #2
```

The second script then checks this CSV against the list from Have I Been Pwned:

```
$ checkhashes my_passwords.csv pwned-passwords-sha1-ordered-by-hash-v7.txt 
8BE3C943B1609FFFBFC51AAD666D0A04ADF83C9D,136802,User Name,Sample Entry
8CB2237D0679CA88DB6464EAC60DA96345513964,2493390,Michael321,Sample Entry #2
```

We can see that both of the example entries' passwords ("Password" and "12345") are unsafe, occurring in breaches 136,802 and 2,493,390 times respectively.

The code and further instructions are available on Github: [https://github.com/tmick0/kdbxtools](https://github.com/tmick0/kdbxtools)
