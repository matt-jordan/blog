---
title: Being Responsible About Security
date: 2017-09-27 20:21:05
tags:
    - Asterisk
    - Programming
---

I wrote a rather lengthy blog post for the [Asterisk Blog](http://blogs.asterisk.org/2017/09/27/rtp-security-vulnerabilities/) about the recent RTP related security vulnerabilities in Asterisk. I won't go into more detail here than I did there, but given that there was a public disclosure of flaws in the first security release that occurred without warning to the project team, things were not as smooth as we would have liked.

To find vulnerabilities, you have to build tools and perform actions that in a non-controlled environment would clearly be malicious. That's not bad in and of itself; what you do with those tools and the information you gain from them determines your ethical bent.

If you look at some of the major vulnerabilities, you can see that things have gone a bit "commercial". Vulnerabilities now get [names](https://blogs.akamai.com/2012/05/what-you-need-to-know-about-beast.html), [marketing sites](http://heartbleed.com/), hell - even [stickers](https://www.redbubble.com/shop/heartbleed+bug+stickers). The researchers who found the RTP vulnerability in Asterisk quickly piggy-backed on this, dubbing the vulnerability ["RTPBleed"](https://rtpbleed.com/).

I get that researchers need to get paid, and marketing certainly is important for those who are trying to run a business. But as a business, do you owe the projects and products that you find vulnerabilities in anything? Or do you only owe your customers who paid you to find the vulnerabilities? How do those conflicts get resolved?

I sure don't know, and I doubt I have a good, well formed opinion on it. But there's something slightly sour when websites and Github projects go up quickly, and the developers responsible for delivering the working software are unaware of the full impact of the vulnerability they supposedly fixed.

