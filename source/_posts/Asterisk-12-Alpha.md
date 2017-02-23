---
title: Asterisk 12 Alpha
date: 2013-09-02 06:31:15
tags:
    - Asterisk
    - Asterisk 12
    - Open Source
---

Last Friday (August 30th), we released [Asterisk 12.0.0-alpha1](http://downloads.asterisk.org/pub/telephony/asterisk/releases/asterisk-12.0.0-alpha1.tar.gz).

This was an incredibly ambitious project. We started work on it directly after [AstriDevCon](https://wiki.asterisk.org/wiki/display/AST/AstriDevCon+2012) last year (October 2012), so we've been hard at work on it for about ten months. Really, I would say that it hasn't even really been that long - the project probably didn't take its full shape until January of this year. So, in reality, over only the past eight months we've:

* Completely [re-architected bridging in Asterisk](https://wiki.asterisk.org/wiki/display/AST/Asterisk+12+Bridging+Project). Every phone call you make through Asterisk - where you actually talk to someone else - is probably going through the new Bridging API. The old bridging code was easily some of the most complex code in Asterisk as well - and while the new API is certainly very complex, it actually provides an abstraction of the bridging operations that the old code (which wasn't really didn't provide any abstraction, or an API) couldn't ever do. Also, it has a completely different threading model that let us nearly completely annihilate masquerades.

* Wrote a completely [new SIP stack](https://wiki.asterisk.org/wiki/display/AST/Configuring+res_pjsip). The old SIP channel driver is 35000 lines of code. The new SIP stack and channel driver has nearly the same feature parity. That's pretty crazy.

* Wrote a new API - [ARI](https://wiki.asterisk.org/wiki/pages/viewpage.action?pageId=29395573) (REST! JSON! WebSockets!) that exposes the communications objects in Asterisk to the outside world. You can, for really the first time, write your own communications applications in the language of your choosing, without having to mix and match AMI/AGI. You can also do things through ARI that you could never really do through AMI/AGI (asynchronous control of media, fine grained bridging operations, etc.)

And that doesn't take into account the ancillary changes: re-writing all core bridging features, CDRs, CEL, every AMI event. It doesn't take into account the infrastructure to make it all possible - the Stasis message bus, threadpools, and the Sorcery data access layer. And the list goes on.

How we got here wasn't straight forward. At AstriDevCon last year, we took away only two objectives: write a new SIP channel driver, and stop exposing internal Asterisk implementation details through the APIs. We didn't know we were going to have to gut Asterisk's bridging code. We didn't know we would end up having to re-write Parking, all Transfers (core and externally initiated), CDRs or CEL. We did not know we would need a more flexible publish/subscribe message bus in Asterisk, one that would end up obsoleting the existing event subsystem.

The moral of the story is: when you start a project, you have to have a goal. But the path to that goal is not a straight line. I'll have to think for awhile how that line curved to where we find ourselves today.

I won't say I'm not a little proud of this release. This was an incredibly ambitious project, and the team at Digium really rocked it out. We aren't a very large team either - besides myself, there are only seven other developers. For us to get this done, with shipped code, is easily the greatest success of my professional career. That isn't to say there isn't a ton left to do: it is, after all, only in an alpha state. But I'm damned proud of what we've accomplished so far.
