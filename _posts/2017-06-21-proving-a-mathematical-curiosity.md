---
layout: post
status: publish
published: true
title: Proving a mathematical curiosity.
author: Travis Mick
date: 2017-06-21
---

Today, a thread full of cool math facts appeared on Reddit. As usual, [someone mentioned the fact that `111111111 * 111111111 = 12345678987654321`](https://www.reddit.com/r/AskReddit/comments/6il1jx/whats_the_coolest_mathematical_fact_you_know_of/dj720mc/). In another reply, someone pointed out that this also works in other bases. For some reason, I decided that I needed to prove that it works in *all* bases.

<!-- more -->

To begin, I needed a general formula for values of the 111... terms. This was fairly straightforward: for a base $$b$$, we want $$b-1$$ base-$$b$$ digits, all ones. To standardize the base, we multiply each digit by an increasing power of $$b$$ and sum. Since each digit is one, we get a nice geometric series which can easily be solved.

$$ \sum\limits_{i=1}^{b-1} b^{i-1} = \frac{b-b^b}{b-b^2}$$

When we multiply this number by itself, we are squaring it, so we end up with $$\left((b-b^b)/(b-b^2)\right)^2$$.

The hard part was writing a general form for the $$1234 \dots (b-1) \dots 4321$$ number. To deal with this, I broke it down into two parts, as illustrated below.

| Digit value      | 1            | 2            |           | $$(b-2)$$        | $$(b-1)$$        | $$(b-2)$$   |           | 2       | 1       |
|                  | $$b^{2b-3}$$ | $$b^{2b-4}$$ | $$\dots$$ | $$b^{2b-(b+1)}$$ | $$b^{2b-(b+2)}$$ |             | $$\dots$$ |         |         |
|                  |              |              |           |                  | $$b^{b-2}$$      | $$b^{b-3}$$ |           | $$b^1$$ | $$b^0$$ |

I calculated the values of the most-significant digits starting at the left, and the values of the least-significant digits starting at a right. To make the math come out nicely, I actually included the center digit in both formulas. That's okay, since we can subtract it off once to make up for the duplicate. Now we have a summation formula for the value of the square.

$$ \left( \sum\limits_{i=1}^{b-1} ib^{2b-(i+3)} \right) + \left( \sum\limits_{i=1}^{b-1} ib^{i-1} \right) - b^{b-2} $$

With a little thinking (or the help of a computer algebra system), we can get a neat closed form.

$$ \left( \sum\limits_{i=1}^{b-1} ib^{2b-(i+3)} + ib^{i-1} \right) - b^{b-2} = \left(\frac{b-b^b}{b^2-b}\right)^2 $$

We can see that this is quite similar to the expression we got for the square above; the only difference is that the $$b-b^2$$ denominator has changed to $$b^2-b$$. Fortunately, this negation goes away when squaring, so we can trivially prove that the two expressions are equal.

And there we have it: proof that this curiosity is true in any base of at least two.
