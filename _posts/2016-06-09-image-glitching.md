---
layout: post
status: publish
published: true
title: Adventures in image glitching
author: Travis Mick
date: 2016-06-09
image: /images/glitch_test-1.jpg
---

[Databending](https://en.wikipedia.org/wiki/Databending) is a type of [glitch art](https://en.wikipedia.org/wiki/Glitch_art) wherein image files are intentionally corrupted in order to produce an aesthetic effect. Traditionally, these effects are produced by manually manipulating the compressed data in an image file. As a result, this is a trial-and-error process; often, edits will result in the file being completely corrupted and unopenable.

Someone recently asked me whether I knew why databending different types of image files produces different effects -- and particularly, why PNG glitches are the most interesting. I didn't know the answer, but the question inspired me to do a little research (mostly reading the Wikipedia articles about the compression techniques used in [different](https://en.wikipedia.org/wiki/JPEG) [image](https://en.wikipedia.org/wiki/Portable_Network_Graphics) [formats](https://en.wikipedia.org/wiki/GIF)). I discovered that most compression techniques are not all that different. Most of them just employ some kind of run-length encoding or dictionary encoding, and then a prefix-free coding step. The subtle differences between the compression algorithms could not explain the wildly different effects we observed (except for in JPEGs, perhaps, since the compression is done in the frequency domain). However, PNG used a pre-filtering step which made it stand out.

<!-- more -->

PNG's pre-filtering is basically a heuristic to try to make the image data more compressible. It tries to translate each pixel of the image into some delta based on pixels in the previous row or column. Since the core compression algorithms have no idea about the two-dimensional nature of an image, this two-dimensional filtering step actually makes a pretty big impact.

There are [five main line filters](https://en.wikipedia.org/wiki/Portable_Network_Graphics#Filtering) defined by the PNG format. Most PNG encoders will try to optimize their choices of filters for each line in the image, in order to reduce the compressed size of the image. However, each filter also yields its own unique glitches.

Thus, an idea was spawned, and I wrote a Python script that glitches images not by manipulating their raw representations, but by applying one of the PNG filters and modifying the pixels in that representation.

## PNG Filter Glitches

Since this article is about glitching -- not image compression -- we're going to focus on the different visual effects we get from each filter. Usually, PNG codecs alternate between filters for each line, however for the sake of demonstration I configured my script to use one filter throughout the whole image.

All of the following glitches are made from a scaled-down version of [this wallpaper from Wallhaven](https://alpha.wallhaven.cc/wallpaper/361124).

Two of the simplest PNG filters are the Up filter and the Sub filter. While Up creates a delta from the adjacent pixel in the previous row, Sub creates a delta from the previous pixel in the same row. The results of glitching on the filters are similar -- Up gives us vertical lines, and Sub gives us horizontal lines.

![Glitch result from the Up filter](/images/glitch_up.jpg)  
*Glitch result from the Up filter*

![Glitch result from the Sub filter](/images/glitch_sub.jpg)  
*Glitch result from the Sub filter*

A more interesting one is the Average filter. This filter looks at the both the above-pixel and the left-pixel and takes their average, then computes a delta using that value. Glitching on the Average filter results in smears of altered color that slowly defuse away across the image.

![Glitch result from the Average filter](/images/glitch_avg.jpg)  
*Average*

My favorite is actually the Paeth filter, which heuristically chooses either the above-pixel, left-pixel, or above-and-to-the-left pixel as the basis for the delta, depending on a function of all three. Glitches applied to this filter give us rectangular disturbances which slowly fade out, however stack together to create harsher effects.

![Glitch result from the Paeth filter](/images/glitch_paeth.jpg)  
*Paeth*

I mentioned that there are five filters, but so far I've only shown four. The reason for this is simple -- the fifth filter is the Null filter, which has no effect whatsoever.

## Mixing Filters

To simulate the mixed effects one can observe in a typical PNG glitch, I implemented a glitching procedure which uses a random filter on each line. The results are pretty intriguing:

![Random line filters](/images/glitch_randlines.jpg)  
*Random line filters*

In this example, I've only allowed a 5% chance for the filter to change from one line to the next; without this provision, the results are quite boring.

To make the glitches even cooler, I started stacking filters on top of each other. Here's an example with an extra pass through the Average filter applied after the randomized filtering stage:

![Randomized + Average](/images/glitch_randlines1.jpg)  
*Randomized + Average*

## Glitch Glitches

Overall, my favorite filters were the ones that had bugs when I wrote them. My original implementation of the Paeth filter had an error due to the use of unsigned integers in the predictor (when really I needed signed ones). However, this version resulted in some cool effects that acted like a combination of the normal Paeth and Average behaviors:

![Buggy Paeth filter](/images/glitch_badpaeth.jpg)  
*Buggy Paeth filter*

I also wrote a buggy Average filter that resulted in some bold vertical lines and some subtler horizontal artifacts:

![Buggy Average filter](/images/glitch_badaverage1.jpg)  
*Buggy Average filter*


## Try it Yourself

I've [posted my glitch scripts on GitHub](https://github.com/tmick0/ppg), so you can try them out too. The original version was pure Python and incredibly slow, but I've now converted the core functionality into a C module and it is several orders of magnitude faster. This is also my first foray into the intricacies of the Python C API, but that's a topic for another post...

To conclude, have two more glitches achieved by mixing various filters:

![glitched image](/images/glitch_test-2.jpg)  

![glitched image](/images/glitch_test-1.jpg)  

