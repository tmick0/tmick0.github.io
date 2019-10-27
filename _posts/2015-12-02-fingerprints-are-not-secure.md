---
layout: post
status: publish
published: true
title: No, fingerprints are not secure
author: Travis Mick
date: 2015-12-02
---

Authentication is the process by which a system determines whether a particular user is allowed to access it. There are three widely agreed-upon methods to authenticate a user:

* Something you have.
* Something you know.
* Something you are.

When you use your key to unlock your front door, you are authenticating yourself using something you have. In information security, passwords are the most popular method of authentication; they are something you know. Authentication by something you are (i.e., biometrics) has historically been only a niche practice, but in recent years it has caught on in the realm of consumer electronics.

When Apple announced Touch ID in late 2013, security experts immediately voiced their concern. The authentication mechanism was [quickly compromised](http://www.ccc.de/en/updates/2013/ccc-breaks-apple-touchid), and there is still very little that Apple can do about it. Why, you ask? Because fingerprints are inherently insecure.

<!-- more -->

An authentication mechanism must meet three major requirements to be considered secure:

* It is secret.
* It is unique.
* It is revocable.

While fingerprints are unique (in theory -- more on that later), they are neither secret nor revocable.

You wear your fingerprints every day, and everybody can see them. But what implications could this possibly have? About a year ago, a German hacker famously announced that he had [compromised the fingerprint of defense minister Ursula von der Layen](http://www.theguardian.com/technology/2014/dec/30/hacker-fakes-german-ministers-fingerprints-using-photos-of-her-hands) using only a photograph. After obtaining a photograph of someone's fingerprint, it is trivial to create a fake. And that easily, your biometric security has been defeated -- permanently.

Permanently? Yes -- biometrics aren't revocable. If someone steals your password, you can easily change it. If they steal the keys to your house, you can change the locks. But if someone steals your fingerprint, you can't get new fingers. Just take a minute and let that sink in.

Now, more on the uniqueness aspect. We are all taught that our fingerprints are unique, which is essentially true. However, the way they are interpreted by computers is not exactly unique. When a fingerprint is used for authentication, an algorithm typically selects a handful of features it thinks are unique and distinct; these points are called *minutia* and are the only portions of the fingerprint considered for identity matching.

Because the environment will not always be the same when your fingerprint is scanned, each read will probably contain a different set of minutia. Whether it be caused by random noise, damage to the imaging lens, or something on your finger (like some dirt, or perhaps a burn or a cut), two scans will never be exactly the same. Thus, fingerprints can not be compared for an exact match the way that passwords are.

What are the implications of this kind of comparison? Well, it means there must be some *acceptance threshold* which determines how "alike" two fingerprints must be to be considered a match. And thus there will always be a chance that someone else's fingerprint will be able to authenticate your identity. We can always raise the threshold to reduce this probability, but it will in turn make it more likely that a valid fingerprint will be rejected. In the current state of the art, [a false positive rate (FPR) of 0.1% can correspond to a false negative rate (FNR) anywhere between 0.02% and 15%](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.211.1363&rep=rep1&type=pdf). Standard consumer devices probably lie on the higher end of that FNR range. I doubt that users would tolerate getting rejected 15% of the time, so it would be a safe bet to assume that the threshold is set so that the false positive rate is much higher than 1 in 1000.

So what have we learned here? Well, that's up to you to decide. In my opinion, the integration of fingerprint scanners into consumer devices is nothing more than a shifty marketing scheme. The general public thinks biometrics are super high-tech, and thus they are a means to inspire consumers to invest in a new gadget.

If you have any sensitive data on your phone, I suggest you at least put it behind a PIN. I reiterate, your fingerprint can be trivially stolen. It might be a good choice for second authentication mechanism (i.e., for two-factor authentication), but it should not be used by itself.

