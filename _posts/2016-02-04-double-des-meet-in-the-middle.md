---
layout: post
status: publish
published: true
title: Demonstrating the double-DES meet-in-the-middle attack
author: Travis Mick
date: 2016-02-04
excerpt: DES (Data Encryption Standard) is an old-school block cipher which has been around since the 1970s. It only uses a 56-bit key, which is undeniably too short for use in the modern day. Between the realization that DES is weak in the late 90s and the invention of AES in the early 2000's, Triple-DES had a brief time to shine. The reasoning behind its design is not immediately obvious -- Double-DES has a critical vulnerability that employs a little cleverness to reduce the search space by several orders of magnitude.
---

## Introduction

DES (Data Encryption Standard) is an old-school block cipher which has been around since the 1970s. It only uses a 56-bit key, which is undeniably too short for use in the modern day. Between the realization that DES is weak in the late 90s and the invention of AES in the early 2000's, Triple-DES had a brief time to shine.

Triple-DES is just what it sounds like: you run the DES algorithm three times. You use two 56-bit keys, **K1** and **K2**, and apply them in the order **K1**-**K2**-**K1**. The result is a cipher with 112 bits of key strength.

Students often ask me, why not just encrypt twice: **K1**, **K2**, done? The reason is that this construction is vulnerable to a particular chosen-plaintext attack, which we call the meet-in-the-middle attack. That is, if the attacker knows your plaintext in addition to your ciphertext, he doesn't have to try all $$2^{112}$$ keys. The maximum amount of work he has to perform is actually only $$2^{56}$$ -- not much more than to break single DES.

## Notation

Before I proceed to an explanation of the meet-in-the-middle attack, let's get some basic cryptography notation out of the way.

Let's call our plaintext **P**, and our ciphertext **C**. Let's call our DES encryption function **E**. It takes in a plaintext and a key; in return, it produces a ciphertext. Similarly, there is a decryption function **D** which takes a ciphertext and a key and spits out a plaintext.

We can then say:

* **E(P, K) = C**
* **D(C, K) = P**
* **D(E(P, K), K) = P**
* **E(D(C, K), K) = C**

You get the idea. Let's move on.

## Double-DES

Double DES uses two keys, **K1** and **K2** We produce a ciphertext using the following formula:

* **C = E(E(P, K1), K2)**

We just encrypt twice, using our two keys. Easy.

Since each key is 56 bits, an attacker that doesn't know the plaintext would need to guess a total of 112 bits of key to break the cipher. Therefore, the total keyspace is $$2^{112}$$, and an average of $$2^{111}$$ keys would have to be guessed before the true **K1** and **K2** are obtained.

## Meet-in-the-middle

Let's redefine Double-DES as a two-step process:

* **M = E(P, K1)**
* **C = E(M, K2)**

We will refer to the result of the first encryption as **M**, as it is our "middle" value. Now, for this attack to work, we assume our adversary has access to both **P** and **C**, but wants to determine **K1** and **K2**

Consider the following statement -- we can easily derive it from the second formula we wrote above:

* **M = D(C, K2)**

An attacker can easily exploit this fact, and essentially make **K1** independent of **K2**. He can guess them each separately, instead of trying to guess them at the same time. But how?

First, the attacker guesses every possible **K1**. Let's call each of his guesses **K1'**. For each guess, he computes **M' = E(P, K1')** to obtain a potential "middle" value, and stores the result in a table along with the corresponding **K1'**.

After filling out the table, with each of the $$2^{56}$$ possible **K1'** keys, the attacker moves on to guessing **K2**. Again, let's refer to each guess as **K2'** He computes **M' = D(C, K2')** then checks whether this **M'** matches any **M'** he stored in the table previously.

If a match is found, then both keys have been broken. That is, if **E(P, K1') = D(C, K2')**, then **K1 = K1'** and **K2 = K2'**.

How much work has the attacker done? Well, generating the table required $$2^{56}$$ encryptions, then finding a match would require $$2^{55}$$ decryptions on average. Overall, we will only have done only $$2^{56.6}$$ cryptographic operations. This is in stark contrast to the $$2^{111}$$ operations which we would expect to perform in a ciphertext-only attack with two 56-bit keys.

## Pseudocode

In short, this is the attack:

```python
def meet_in_the_middle(C, P):
  table = {}
  for i in range(0, 2^56):
    table[E(P, i)] = i
  for i in range(0, 2^56):
    if D(C, i) in table:
      return table[D(C, i)], i
```

That's it.

## Demonstration

I've written a quick demonstration of this attack in Python using the pycrypto library. The source code is [on my Github](https://github.com/tmick0/class-resources/blob/master/ddes/ddes.py). For the demo, I have only used 18 bits of the key (just so you don't have to wait forever to see the results). There are also some optimizations which can be made (e.g., DES takes the key as a 64-bit input even though only 56 of those bits are used... we can skip some of the inputs that we have tested to make it faster). However, it works.

## Triple-DES

So why isn't Triple-DES vulnerable to this attack? Well, it's easy. We can only "skip" one encryption using this strategy. Since Triple-DES is defined as **E(D(E(P,K1),K2),K1)**, we can't split it up in such a way that we won't, at some point, have to guess both **K1** and **K2** at the same time. A formal proof of this claim is, unfortunately, out of scope of this article.

## Last word

This attack isn't specific to DES. It works on any composition of two ciphers which is constructed as **E(E(P, K1), K2)**. So don't ever, ever, ever try to do that.

