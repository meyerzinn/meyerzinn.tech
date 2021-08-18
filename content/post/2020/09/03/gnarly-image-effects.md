+++
author = "Meyer"
categories = ["computer-science", "graphics"]
date = 2020-09-03T10:40:20Z
description = ""
draft = false
image = "/images/2020/09/hd-1.png"
slug = "gnarly-image-effects"
summary = "Gallery of image effects for my CS 314H project."
tags = ["computer-science", "graphics"]
title = "Gnarly image manipulation"

+++


For CS 314H, our first project involves programming routines to perform visual transformations for images. While I can't share any of the code, I will present a few of my favorite examples:

{{< gallery caption="A motorcycle with a single Gaussian blur and edge detector applied (middle). After another edge detection pass and some smoothing/Gaussian blurs, the outline on the right emerged. Source image courtesy of Harley-Davidson." >}}
{{< galleryImg  src="/images/2020/09/harley-davidson-2S4FDh3AtGw-unsplash-1.jpg" >}}
{{< galleryImg  src="/images/2020/09/gaussian-and-edge-detection---hd.png">}}
{{< galleryImg  src="/images/2020/09/hd.png" >}}
{{< /gallery >}}

Of course, often times our greatest feats are the results of our most egregious blunders:

{{< gallery caption="Affine identity transform... unless?" >}}
{{< galleryImg  src="/images/2020/09/11.jpg">}}
{{< galleryImg  src="/images/2020/09/failed_affine_identity-1.png">}}
{{< /gallery >}}

I wanted to try and replicate the famous Obama [Hope poster](https://en.wikipedia.org/wiki/Barack_Obama_%22Hope%22_poster), but I found an [entire paper](http://pages.cs.wisc.edu/~dyer/cs534-spring11/hw/hw5/projects/paprocki_bric.pdf) from the University of Washington on the subject, so I settled for a posterized flower:

{{< gallery caption="A posterized flower. Source image courtesy of Arthur Rachbauer" >}}
{{< galleryImg  src="/images/2020/09/arthur-rachbauer-vYyHLDPKWd4-unsplash.jpg" >}}
{{< galleryImg  src="/images/2020/09/posterize.jpeg" >}}
{{< /gallery >}}

Here is yet another classic edge detection nightmare (diagonal lines everywhere), surprisingly overcome by a simple Gaussian filter and some downsampling:

{{< gallery caption="Downsampling, Gaussian blur and edge detector applied to a building. Source image courtesy Stephan Schmid." >}}
{{< galleryImg  src="/images/2020/09/original-building-small.jpeg" >}}
{{< galleryImg  src="/images/2020/09/building.jpeg" >}}
{{< /gallery >}}



