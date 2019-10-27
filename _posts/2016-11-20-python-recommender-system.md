---
layout: post
status: publish
published: true
title: A simple recommender system in Python.
author: Travis Mick
date: 2016-11-20
---

Inspired by [this post I found about clustering analysis](http://blog.revolutionanalytics.com/2013/12/k-means-clustering-86-single-malt-scotch-whiskies.html) over a dataset of Scotch tasting notes, I decided to try my hand at writing a recommender that works with [the same dataset](https://www.mathstat.strath.ac.uk/outreach/nessie/nessie_whisky.html). The dataset conveniently rates each whisky on a scale from 0 to 4 in each of 12 flavor categories.

<!-- more -->

The user simply lists a few whiskies they like, the script gets a general idea of what kind of flavor they enjoy, and then spits out a list of drams they might want to try. On the inside, the process works like this:

* Determine each whisky to be either above average or below average in each flavor category, assigning a "flavor rank" of +1 or -1 respectively.
* Determine the user's flavor profile: For each flavor category, find the mean and variance of the corresponding flavor ranks of the whiskies listed by the user.
* Compute a score for each whisky: First, multiply each of the whisky's flavor ranks by the corresponding mean from the user's flavor profile. Then, compute the average over all the flavors, weighed inversely by the variances.
* Sort the whiskies by score, and show the user the highest ranked ones.

I implemented this algorithm in Python. [The code is available on Github](https://github.com/tmick0/whisky-recommender/blob/master/whisky_recommender.py). You'll need to download the file whiskies.txt from the above dataset link. Here is an example of running the script:

```
$ python3 whisky_recommender.py --like Laphroig --like Lagavulin
We have detected your flavor preferences as:
- Tobacco (weight 1.00)
- Winey (weight 1.00)
- Medicinal (weight 1.00)
- Body (weight 1.00)
- Smoky (weight 1.00)
- Sweetness (weight -1.00)
- Floral (weight -1.00)
- Fruity (weight -1.00)
- Honey (weight -1.00)
- Spicy (weight -1.00)
- Malty (weight -1.00)
- Nutty (weight -1.00)

Our recommendations: 
- Clynelish (weight 6.00)
- Caol Ila (weight 6.00)
- Dalmore (weight 4.00)
- Isle of Jura (weight 4.00)
- GlenDeveronMacduff (weight 4.00)
- GlenScotia (weight 4.00)
- Highland Park (weight 4.00)
- Ardbeg (weight 4.00)
```

*Note that the misspelling of "Laphroaig" is not my fault -- the error is present in the original dataset.*

The script can easily be adapted to recommend anything with similar-looking data (i.e., where each instance is assigned scores over some set of attributes). There is nothing specific to whisky about the algorithm, just change some of the strings. ;)

