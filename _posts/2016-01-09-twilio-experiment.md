---
layout: post
status: publish
published: true
title: A fun experiment with Twilio
author: Travis Mick
date: 2016-01-09
---

I first heard about [Twilio](https://www.twilio.com/) a long, long time ago. As Google Voice faded out of relevance, it took the lead in the *mobile-communication-as-a-service* market. However, I had never had the chance (or inclination) to play around with its API until today.

About 12 hours after we landed back in the US from our holiday in Mexico, Lynsey departed once again -- this time to the Plant and Animal Genome conference (PAG) in San Diego. She asked me to supply her with pictures of our cats for the duration of her trip. I told her I would send her a cat pic every hour, on the hour.

I didn't realize what I had gotten myself into until I had already deposited $20 into a new Twilio account and spent 2 hours coding away... Though my goal was just to send some photos of cats, I had developed a [pretty general application](http://github.com/tmick0/twiliq) that lets you build a queue of MMSes to be disseminated at a constant rate.

<!-- more -->

This was not only my first experiment with Twilio -- it was also my first adventure with Flask, a framework for building simple HTTP services in Python. I must say that both Flask and Twilio were extremely easy to use, and I would not hesitate to use them again.

I will certainly be finding some more uses for Twilio in the future -- it will be invaluable for the SMS-based 2FA we plan to integrate into [Zeall](http://zeall.us). But for the time being, it will be serving as a medium for the delivery of photos of cats.

If anyone is interested in my Twilio-based app, here is a link again: [https://github.com/tmick0/twiliq](https://github.com/tmick0/twiliq)

