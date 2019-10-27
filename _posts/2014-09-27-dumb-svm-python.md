---
layout: post
status: publish
published: true
title: A dumb SVM classifier for Python
author: Travis Mick
date: 2014-09-27
---
Tonight I got bored and implemented a linear SVM in Python. Though Python has its own facilities for solving quadratic programming problems, I chose to write a module which interfaces with Octave instead. My implementation simply writes an Octave script then runs it in order to solve the QP. All other aspects of the SVM are implemented in pure Python.

<!-- more -->

To be honest, it didn't even occur to me to try to do the QP in Python until I already posted my code on Github. I already knew about Python's qp package, but it wasn't exactly in the front of my mind while writing this module, since I had been messing around with Octave for a few days prior.

Anyhow, the source and more information is at [https://github.com/tmick0/wrappy-svm](https://github.com/tmick0/wrappy-svm).

