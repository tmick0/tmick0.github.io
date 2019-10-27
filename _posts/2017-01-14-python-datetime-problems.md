---
layout: post
status: publish
published: true
title: The problem with Python's datetime class.
author: Travis Mick
date: 2017-01-14
---

This might sound like a strong opinion, but I'm just going to put it out there: **Python should make `tzinfo` mandatory on all `datetime` objects**.

To be fair, that's just an overzealous suggestion prompted by my frustration after spending two full days debugging timestamp misbehaviors. There are plenty of practical reasons to keep timezone-agnostic `datetime`s around. Some projects will never need timestamp localization, and requiring them to use `tzinfo` everywhere will only needlessly complicate things. However, if you think you might *ever* need to deal with timezones in your application, then you *must* plan to deal with them from the start. My real proposition is that **a team should assess its needs and set internal standards regarding the use of timestamps before beginning a project**. That's more reasonable, I think.

<!-- more -->

## The problem.

If you're handling timestamps in Python, chances are you are using its standard `datetime` class. The `datetime` honestly has a pretty great feature set: it lets you do arithmetic with dates, stringify dates, etc.; pretty much anything you need to do with a date, `datetime` will do for you. However, lots of problems arise when you use "naive" `datetime` objects, i.e., `datetime`s without any timezone awareness.

Python 2.x had a similar problem differentiating between different types of strings. It's a long story, but essentially whether a string contained binary or text, it was still a string. People who knew what they were doing with strings didn't have a problem, but it was far from idiot-proof. In fact, you didn't really need to be an idiot to fall into the trap -- just naive. This caused lots of problems, so eventually Python 3.x decided to make `str` and `bytes` into totally different things.

Naivety is also detrimental in the use of the `datetime`. The only place where it works as intended, without a hassle, is in an application where you never have to do any kind of localization or timezone conversion. Once you start trying to convert naive `datetime`s between timezones, you'll find that you've been shot in the foot. My personal opinion is that footguns should not exist, or at least not in the standard libraries of high-level languages like Python.

I wish I could seriously propose that Python eliminate the naive `datetime`, but this would only cause problems. Naive `datetime`s are great, since they don't ever require you to look at a timezone database (`tzdb`). Once you start dealing with timezones, you have to worry about the `tzdb` being up to date. If you don't have complete control over the environment your code is running in, then you can expect inconsistent behavior between users. Whether this is a problem depends on the nature of your project, and I'm not about to enumerate all the possibilities --- you can weigh the consequences yourself.

In short, I propose that anyone starting a new project should decide -- at its very beginning -- what to do with timestamps. In most cases, I think that naive `datetime`s should be avoided altogether --- explicit timezone information (`tzinfo`) should be included absolutely anywhere `datetime`s are used. You should use naive `datetime`s only if you will never need to convert between timezones, you can't trust users to have an up-to-date `tzdb`, *and* having inconsistent *tzdb*s between users would likely create other problems.

Unfortunately, I didn't have the foresight to disallow naive `datetime`s in my project at its inception; therefore, I ran into a problem two years down the road at which point I had to do *a lot* of refactoring. The remainder of this article details the problems I encountered and the subsequent process of eliminating all naive `datetime`s from my codebase.

## The dilemma.

When I first started using `datetime`s I didn't know any better. I simply called `datetime.now()` whenever I needed a timestamp. At that time (no pun intended), my app was only displaying times for a single timezone. Eventually, I realized that I should be converting timestamps to users' local timezones, and my naivety came back to bite me in the ass.

If you didn't know already, `datetime.now()` gives you the current time in your local timezone. However, it does not have this timezone information attached by default: it gives you a naive `datetime` object.

I tried to convert one of these naive `datetime`s using the `pytz` library (which handles timezone magic):

```python
>>> import pytz
>>> from datetime import datetime
>>> now = datetime.now()
>>> now
datetime.datetime(2017, 1, 14, 15, 15, 11, 475618)
>>> pytz.timezone("America/New_York").localize(now)
datetime.datetime(2017, 1, 14, 15, 15, 11, 475618, tzinfo=<DstTzInfo 'America/New_York' EST-1 day, 19:00:00 STD>)
>>> pytz.timezone("Australia/Sydney").localize(now)
datetime.datetime(2017, 1, 14, 15, 15, 11, 475618, tzinfo=<DstTzInfo 'Australia/Sydney' AEDT+11:00:00 DST>)
```

Note that my local timezone is MST; however, the `datetime` has no idea about this and therefore *doesn't actually do any conversion* when I ask for another timezone. All of the `datetime`s it returned are the same, except for their attached `tzinfo`s.

My first idea was to inject my local timezone into all the naive `datetime` objects:

```python
>>> now = datetime.now(pytz.timezone("America/Denver"))
>>> now
datetime.datetime(2017, 1, 14, 15, 20, 8, 410761, tzinfo=<DstTzInfo 'America/Denver' MST-1 day, 17:00:00 STD>)
>>> pytz.timezone("America/New_York").normalize(now)
datetime.datetime(2017, 1, 14, 17, 20, 8, 410761, tzinfo=<DstTzInfo 'America/New_York' EST-1 day, 19:00:00 STD>)
>>> pytz.timezone("Australia/Sydney").normalize(now)
datetime.datetime(2017, 1, 15, 9, 20, 8, 410761, tzinfo=<DstTzInfo 'Australia/Sydney' AEDT+11:00:00 DST>)
```

Now the conversion works. However, there are also lots of places in my codebase where I'm accepting or returning Unix timestamps. If you don't know, Unix timestamps are always UTC. If you don't ask otherwise, `datetime` will convert them into local time for you, again without a `tzinfo`:

```python
>>> import time
>>> unixtime = time.time()
>>> unixtime
1484432537.234377
>>> datetime.fromtimestamp(unixtime)
datetime.datetime(2017, 1, 14, 15, 22, 17, 234377)
```

This isn't so bad, we can fix it mostly the same way as the `datetime.now()`:

```python
>>> datetime.fromtimestamp(unixtime, pytz.timezone("America/Denver"))
datetime.datetime(2017, 1, 14, 15, 22, 17, 234377, tzinfo=<DstTzInfo 'America/Denver' MST-1 day, 17:00:00 STD>)
```

You can even convert it to another timezone:

```python
>>> datetime.fromtimestamp(unixtime, pytz.timezone("America/New_York"))
datetime.datetime(2017, 1, 14, 17, 22, 17, 234377, tzinfo=<DstTzInfo 'America/New_York' EST-1 day, 19:00:00 STD>)
```

But what if we want to convert a localized `datetime` into a Unix timestamp? If you're familiar with the C `strftime` API, you'll be tempted to use `strftime("%s")`:

```python
>>> datetime.now().strftime("%s")
'1484432862'
```

That time, we got the correct result. But watch this:

```python
>>> t = time.time()
>>> datetime.fromtimestamp(time.time(), pytz.timezone("America/Denver")).strftime("%s")
'1484433034'
>>> datetime.fromtimestamp(time.time(), pytz.timezone("America/New_York")).strftime("%s")
'1484440239'
```

What's going on here? We created a single Unix timestamp (`t`), and converted it to two separate `datetime`s in two different timezones. We already know that conversion from Unix time into any timezone works correctly. We should have gotten the same result when we converted back. However, it turns out that *you can only convert a `datetime` to a Unix timestamp if it is in your local timezone*.

Actually, `strftime("%s")` is unsupported in Python. It ends up just stripping the `tzinfo`, thereby creating a naive timestamp in an arbitrary timezone, and calling the C `strftime` which assumes it's being given a local timestamp. Obviously this doesn't work.

Now how do you create a Unix timestamp the correct way? It's ugly:

```python
>>> t = datetime.now(pytz.timezone("America/Denver"))
>>> (t - datetime(1970, 1, 1, tzinfo=pytz.UTC)).total_seconds()
1484433334.448718
```

In short, you need to take a timezone-aware `datetime`, subtract the Unix epoch from it (thereby obtaining a `timedelta`), and convert it to seconds. Luckily for us, any arithmetic done with timezone-aware `datetime`s is automatically converted to UTC.

Fortunately, it fails if you pass it a naive `datetime`:

```python
>>> t = datetime.now()
>>> (t - datetime(1970, 1, 1, tzinfo=pytz.UTC)).total_seconds()
Traceback (most recent call last):
 File "<stdin>", line 1, in <module>
TypeError: can't subtract offset-naive and offset-aware datetimes
```

Unfortunately, I'm sure a lot of beginners are still going to get screwed, since the most popular StackOverflow answers for this situation give you incorrect solutions like the following:

```python
>>> t = datetime.now()
>>> (t - datetime(1970, 1, 1)).total_seconds()
1484408399.491469
```

It doesn't fail, since both timestamps are naive. However, the result is wrong: since I used my local time, the result is the number of seconds since 1970-1-1 *in my timezone*, rather than in UTC.

## The solution.

Upon discovering how difficult it is to do anything nontrivial with timestamps correctly, I decided to eliminate naive `datetime`s from my codebase altogether and standardize an API for doing common tasks with timezone-aware `datetime`s. This would help prevent other contributors to my project from shooting themselves in the foot (and by extension, shooting me).

The `timehelper` class I created is meant to be used any time you want to:

* Get the current time,
* Localize and format a timestamp,
* Parse a Unix timestamp, or
* Create a Unix timestamp.

Any use of the builtin `datetime` functions to do these things will now result in a failed code review, because they're all nearly impossible to get right.

The `timehelper` itself is very simple:

```python
import pytz, psycopg2
from datetime import datetime

class timehelper(object):
 
  @staticmethod
  def localize_and_format(tz, fmt, dt):
 
    # disallow naive datetimes
    if dt.tzinfo is None:
      raise ValueError("Passed datetime object has no tzinfo")
 
    # workaround for psycopg2 tzinfo
    if isinstance(dt.tzinfo, psycopg2.tz.FixedOffsetTimezone):
      dt.tzinfo._utcoffset = dt.tzinfo._offset
 
    return pytz.timezone(tz).normalize(dt).strftime(fmt)
 
  @staticmethod
  def now():
    return datetime.utcnow().replace(tzinfo=pytz.UTC)
 
  @staticmethod
  def to_posix(dt):
    return (dt - datetime(1970, 1, 1, tzinfo=pytz.UTC)).total_seconds()
 
  @staticmethod
  def from_posix(p):
    return datetime.fromtimestamp(p, pytz.UTC)
```

Its usage is simple, too:

* Instead of calling `datetime.now()`, just call `timehelper.now()`. You'll automatically be given a timezone-aware UTC `datetime`. The goal of this is to use UTC everywhere within the codebase.
* To convert from a Unix timestamp to a UTC `datetime`, use `timehelper.from_posix()`.
* To convert from a `datetime` to a Unix timestamp, use `timehelper.to_posix()`.
* To localize a timestamp to a timezone and format it at the same time, use `timehelper.localize_and_format()`. I decided to always localize and format together in order to help enforce the goal of using UTC everywhere.

You might notice that there's some special magic in the `localize_and_format()` method for dealing with `tzinfo` objects created by `psycopg2`. For some reason, its API has a slight mismatch against that of `pytz`. If you aren't using `psycopg2`, you can strip out that `if` statement. But if you are, make sure all the timestamp-containing columns in PostgreSQL are declared as `timestamp with time zone`, rather than simply `timestamp`. This is another footgun; traditionally, Postgres used timezones implicitly, but this was reverted in order to comply with SQL standards.

## The conclusion.

It took me several hours of research to figure out how to properly deal with timestamps in Python. Its `datetime` API is full of gotchas, and a naive developer can easily succumb to its apathy. It turns out that I had many subtle bugs in my codebase before I revisited all code pertaining to timestamps.

As it's unlikely that naive `datetime`s will ever actually be removed from Python, I recommend that everyone create standards for `datetime` manipulation within their projects. Doing so may prevent tricky bugs and large rewrites later on.

If you happen to stumble upon this article in your own search for `datetime` incantations, feel free to use my above `timehelper` class. Consider it public domain.

