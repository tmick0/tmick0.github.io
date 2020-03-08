---
layout: post
status: publish
published: true
title: An automated workflow for fetching and mosaicing Landsat imagery
author: Travis Mick
date: 2020-03-07
image: /images/pansharpened_false_color.jpg
---

![A false color image of Albuquerque](/images/pansharpened_false_color.jpg)

One extension I had been planning for my previous project involving
[terrain meshes]({% post_url 2019-10-05-printing-terrain-meshes %})
was the addition of satellite imagery to add color to the models.
[Landsat](https://landsat.gsfc.nasa.gov/landsat-8/landsat-8-overview/),
in addition to enabling important climatology research and other science missions, turned out to be a great
source of open data to use for this.

<!-- more -->

As a step toward integrating this data with my terrain mesh generator, I've implemented
an automated workflow for fetching, radiometrically correcting, and mosaicing these images.

AWS generously provides free access to [its S3-based Landsat mirror](https://registry.opendata.aws/landsat-8/),
however the [Worldwide Reference System](https://landsat.gsfc.nasa.gov/the-worldwide-reference-system/)
Landsat uses to index products makes it a bit less than straightforward to identify the right data
to fetch. Therefore, I implemented a parser for the provided scene list that resolves requested
geodetic extents to a set of footprints which cover them. The script then fetches all of the data
needed to generate the requested output product.

After obtaining the data, a bit of massaging is required to make it useful. The data has to be 
[radiometrically calibrated](https://en.wikipedia.org/wiki/Radiometric_calibration#Satellite_sensor_calibration)
and then warped to a more flexible map projection.

Finally, I've implemented a [pansharpening](https://en.wikipedia.org/wiki/Pansharpened_image) operation
which uses Landsat 8's panchromatic band to improve the spatial resolution of its spectral bands and produce
a high-fidelity output product.

The code that handles all of this has been [released to Github](https://github.com/tmick0/landsat_fetch) for all to use.
