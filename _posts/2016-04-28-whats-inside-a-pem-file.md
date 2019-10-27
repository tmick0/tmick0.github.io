---
layout: post
status: publish
published: true
title: What's inside a PEM file?
author: Travis Mick
date: 2016-04-28
excerpt: As you probably know, PEM is a base64-encoded format with human-readable headers, so you can kind of figure out what you're looking at if you open it in a text editor. That base-64 payload actually has the potential to contain a lot of information. But anyway, what is inside that public key? How can we find out?
---

As you probably know, PEM is a base64-encoded format with human-readable headers, so you can kind of figure out what you're looking at if you open it in a text editor.

For example, let's look at an RSA public key:

```
-----BEGIN PUBLIC KEY-----
MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEA4YFgwNrEkMdynjtDsM0q
b+Hedk8p4pySfxakYTfSQPEyGxxnQGcVMV2ZEjPR4nZeqJrtNTlixhK2YWqunE6I
KopVDq3WvtPKweNEeZ8B2lA2I8FFrpZSjI/Tosq8/MbTd/Y/C4Q8Qcz78MF/NH17
/E82K3ca9/LM2b4KGTEIhsLUff7OGrJM7lPcQZN3EOdUeQnzT9uTh8Z9oFqChfJP
pLwwSebfrRB7VMXjeKHZmubSO5pULHLdZLbkgLSmnhbgBjO6apG0tkYyOeWd6L8F
MzA21WkXJdANrr1s/yv5zS9hx1q9jSM8Me9QA2/iaAbgem7VwQ2YlPiXEvUq48oB
VsKXMpHQ6A2cUygs+PiSFuUzNjTIebWFTWmKKuoRx0O2m63fAZJaT2aJA4G0HqdJ
ZQ2Aqr4Acs1+28IhLxUbMAlHJ4N2XPnE2WpQYbtUR4zZMXU+bVIToXuqHCLo4pf/
qEIK/xzr/S8WdvMvRVSOtVIIQwyaMDUxsnnKozYSVHvzYsxQo3b3VD5OOqmg1mx1
+Z/PLFViLkBjo+ZMkl5dFbsgYyHmkn/uvCV19IpjkdDNfFgdrOlSdNTnlGU7su5L
L31k/IwSvD0PR0egxiv8HhegaYwqgujVylB0gntyBsrVVHfE3Wr2+aJlR3YmrdCZ
lsAiSbnFxgGtfB6INHepFdkCAwEAAQ==
-----END PUBLIC KEY-----
```

We can see that we have a public key just from the header. But what's all the base64 stuff? If it's a public key, we know that there should be a modulus and an exponent hidden in there. And there's also probably some kind of hint that this is an RSA key, as opposed to some other type of key.

That base-64 payload actually has the potential to contain a lot of information. It's an ASN.1 (Abstract Syntax Notation One) data structure, encoded in a format called DER (Distinguished Encoding Rules), and then finally base64-encoded before the header and footer are attached.

ASN.1 is used for a lot of stuff besides keys and certificates; it is a generic file format that can be used to serialize any kind of hierarchical data. DER is just one of many encoding formats for an ASN.1 structure -- e.g. there is an older format called BER (Basic Encoding Rules) and an XML-based format called XER (you can probably guess what that stands for).

But anyway, what is inside that public key? How can we find out?

## Parsing PEM files

OpenSSL comes with a utility called asn1parse. It can understand plain DER or PEM-encoded ASN.1. Let's feed our public key into it and take a look:

```bash
$ openssl asn1parse -in public.pem -i
    0:d=0  hl=4 l= 546 cons: SEQUENCE          
    4:d=1  hl=2 l=  13 cons:  SEQUENCE          
    6:d=2  hl=2 l=   9 prim:   OBJECT            :rsaEncryption
   17:d=2  hl=2 l=   0 prim:   NULL              
   19:d=1  hl=4 l= 527 prim:  BIT STRING
```

The column that contains "cons" and "prim" gives us information about the hierarchical structure of the data. "Cons" stands for a "constructed" field, i.e., a field that encapsulates other fields; on the other hand, "prim" means "primitive."

The column after "cons" or "prim" tells us what type of data is in that field. The -i flag I've supplied causes that column to be indented according to how deep we are in the hierarchical structure. So overall, what are we looking at?

There is one root SEQUENCE object. That SEQUENCE contains another SEQUENCE and a BIT STRING. That internal SEQUENCE has an OBJECT and a NULL terminator. The OBJECT field is actually an Object Identifier -- it contains some constant that tells us what kind of object we're decoding. As we may have suspected, it tells us that we're decoding an RSA key.

Here's where stuff gets really interesting: That BIT STRING field actually contains more ASN.1 data. Let's jump in and parse it:

```bash
$ openssl asn1parse -in public.pem -strparse 19 -i 
    0:d=0  hl=4 l= 522 cons: SEQUENCE          
    4:d=1  hl=4 l= 513 prim:  INTEGER           :E18160C0DAC490C7729E...
  521:d=1  hl=2 l=   3 prim:  INTEGER           :010001
```

(The -strparse 19 flag means "parse the data in the field located at offset 19 in the original structure." If there was another BIT STRING inside this one, we could add another -strparse argument to recurse into it.)

What do we have here? A SEQUENCE containing two INTEGERs. It turns out that the first INTEGER is our modulus, and the second INTEGER is the exponent.

You can actually infer the effective length of the key from this information. The third column indicates the length of the modulus is 513 bytes; one of these bytes is just padding, so that means our key is 512 bytes (or 4096 bits) in strength.

## Reverting to DER

As I explained earlier, PEM is just a wrapper around DER. So can we dump the raw DER and parse it that way? You sure can. You just cut out the PEM headers and base64-decode the payload, and what comes out is DER.

```bash
$ grep -v "PUBLIC KEY" public.pem | openssl base64 -d > public.der
```

Now we can inspect the raw DER:

```bash
$ hexdump public.der 
0000000 8230 2202 0d30 0906 862a 8648 0df7 0101
0000010 0501 0300 0282 000f 8230 0a02 8202 0102
0000020 e100 6081 dac0 90c4 72c7 3b9e b043 2acd
0000030 e16f 76de 294f 9ce2 7f92 a416 3761 40d2
0000040 32f1 1c1b 4067 1567 5d31 1299 d133 76e2
0000050 a85e ed9a 3935 c662 b612 6a61 9cae 884e
0000060 8a2a 0e55 d6ad d3be c1ca 44e3 9f79 da01
0000070 3650 c123 ae45 5296 8f8c a2d3 bcca c6fc
...
```

Or, we can use asn1parse if we actually want to understand it:

```bash
$ openssl asn1parse -in public.der -inform DER -i
    0:d=0  hl=4 l= 546 cons: SEQUENCE          
    4:d=1  hl=2 l=  13 cons:  SEQUENCE          
    6:d=2  hl=2 l=   9 prim:   OBJECT            :rsaEncryption
   17:d=2  hl=2 l=   0 prim:   NULL              
   19:d=1  hl=4 l= 527 prim:  BIT STRING   

$ openssl asn1parse -in public.der -inform DER -strparse 19 -i
    0:d=0  hl=4 l= 522 cons: SEQUENCE          
    4:d=1  hl=4 l= 513 prim:  INTEGER           :E18160C0DAC490C7729E...
  521:d=1  hl=2 l=   3 prim:  INTEGER           :010001
```

This is the exact same data we saw earlier, but there's your proof that PEM really is just DER underneath a layer of base64.

