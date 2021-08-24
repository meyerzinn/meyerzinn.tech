+++
author = "Meyer"
date = 2021-08-19T00:11:54+00:00
summary = "Key takeaways from interning at Cloudflare with the Magic Transit team."
draft = false
slug = "internship-takeaways"
title = "Summer as a transit magician"

+++

There are a lot of things I expected when I accepted a summer internship with Cloudflare's Magic Transit team. I figured they would send me some nice swag (thanks for the socks!), and I would have to write some code.

What I did *not* anticipate was shipping that code to Cloudflare’s edge to improve routing of trillions of bytes of data for real customers every day. I also did not expect my work to have a substantial, quantifiable impact on the product within twelve short weeks.

I wrote a [Cloudflare blog post](https://blog.cloudflare.com/making-magic-transit-health-checks-faster-and-more-responsive/) with the technical details of my work, but I wanted to share some takeaways from this summer that make it in:

1. Planning and testing are important. I was hired to help monitor Internet weather, not cause a storm. Corollary: Sometimes moving slowly in the right direction is better than moving at break-neck speed into a wall.
2. Things break in weird ways when you scale them up by 10,000x.
3. The key to understanding distributed systems is observability: anticipating every problem you might encounter is impossible, so it’s important to keep a lot of data and logs that will help you identify problems as they arise.
