---
layout: post
title:  "Getting My Graphics Smaller - Experimenting with AVIF"
categories: blog image-compression
---

I thought I was going to move from using JPEG images to [AVIF](https://en.wikipedia.org/wiki/AVIF) in my blog.

## Why JPEG in the first place?

Most of my blog images are screenshots, which can be quite big. On the other hand, one of my goals is to keep my blog website small and simple. (This is why I went for [Jekyll](http://jekyllrb.com/) in the first place.) Because of things like color gradients and antialiasing, lossless compression doesn't work quite as well as I originally expected on those images.

While JPEG might not appear as a natural choice for screenshots, in the lowest quality setting it offers a more than 50% average size reduction over PNG, and I found I could live with the compression artifacts.

## A Better Solution

Then I came across [this article](https://www.simplethread.com/why-your-website-should-not-use-dithered-images/) which has a comprehensive comparison of various image formats that offer superior compression and quality. It also links to a [free converter](https://squoosh.app/)! All the numbers in this benchmark are in favor of AVIF. 
Here is one of the images from my previous blog post as JPEG:

![Screenshot as JPEG](/assets/2021-11-07-3-filter.jpeg)

And as AVIF:

![Screenshot as AVIF](/assets/2021-11-07-3-filter.avif)

The AVIF version has a lot less visible artifacts, and it is more than 2/3 smaller (51 KB vs. 167 KB)!

[Here](https://github.com/kornelski/cavif-rs) is a command line tool to convert PNG or JPEG files into AVIF.

## Browser Support

According to [Wikipedia](https://en.wikipedia.org/wiki/AVIF) most modern browsers support AVIF. But alas, not Safari! This is really a bummer. I will not be able to benefit from improved compression then.
