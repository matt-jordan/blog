---
title: 'Asterisk 12: Now Available (with PJSIP)'
date: 2013-12-23 19:31:30
tags:
    - Asterisk
    - Asterisk 12
    - SIP
    - PJSIP
---
We released [Asterisk 12](http://lists.digium.com/pipermail/asterisk-announce/2013-December/000507.html) on Friday. So, that's a thing.

There's a lot to talk about in Asterisk 12 - as [mentioned when we released the alpha](https://www.matthewjordan.net/2013/09/02/asterisk-12-alpha/), there's a lot in Asterisk 12 that makes it unlike just about any prior release of Asterisk. A new SIP channel driver, all new bridging architecture (which is pretty much the whole "make a call" part of Asterisk), redone CDR engine (borne out of necessity, not choice), CEL refactoring, a new internal publish/subscribe message bus (Stasis), AMI v2.0.0 (build on Stasis), the Asterisk REST Interface (ARI, also built on Stasis), not to mention the death of `chan_agent` - replaced by `AppAgentPool`, completely refactored Parking, adhoc multi-party bridging, rebuilt features, the list goes on and on. Phew. It's a lot of new code.

Lots of blog posts to write on all that content.

I've been thinking a lot about how we got here. The SIP channel driver is easiest: I'll start there.

Ever since I was fortunate enough to find myself working on Asterisk - now two years, five months ago (where does the time go!) - it was clear `chan_sip` was a problem. Some people would probably have called it "the problem". You can see why - output of `sloccount` below:

* 1.4 - 15,596

* 1.6.2 - 19,804

* 1.8 - 23,730

* 11 - 25,823

* 12 - 25,674

Now, I'm not one for measuring much by SLOC. Say what you will about Bill Gates, but he got it right when he compared measuring software progress by SLOC to measuring aircraft building progress by weight. That aside, no matter who you are, I think you can make two statements from those numbers:

1. The numbers go up - which means we just kept on piling more crap into `chan_sip`

2. The SLOC is too damned high

{% asset_img sloc_is_too_high.jpg The SLOC is too damned high %}

I don't care who you are, 15000 lines of code in a single file is hard to wrap your head around. 25000 is just silly. To be honest, it's laughable. (As an aside: when I first started working on Asterisk and saw `chan_sip` (and `app_voicemail`, but that's another story), I thought - "oh hell, what did I get myself into". And that wasn't a good "oh hell".)

It isn't like people haven't tried to kill it off. Numerous folks in the Asterisk community have taken a shot at it - sometimes starting from scratch, sometimes trying to refactor it. For various reasons, the efforts failed. That isn't an insult to anyone who attempted it: writing a SIP channel driver from scratch or refactoring `chan_sip` is an enormous task. It's flat out daunting. It isn't just writing a SIP stack - it's integrating it all into Asterisk. And, giving credit to `chan_sip`, there's a lot of functionality bundled into that 15000/25000 lines of code. Forget calling - everything from a SIP registrar, SIP presence agent (kinda, anyway), events, T.38 fax state, a bajillion configuration parameters is implemented in there. The 10% rule applies especially to this situation - you need to implement 90% of the functionality in `chan_sip` to have a useful SIP stack, and the 10% that you don't implement is going to be the 10% everyone cares about. And none of them will agree on what that 10% is.

As early as Asterisk 11, a number of folks that I worked with had starting thinking about what it would take to rewrite `chan_sip`. At the time, I was a sceptic. I'm a firm believer in not writing things from scratch: [90% of new software projects fail](http://www.it-cortex.com/Stat_Failure_Rate.htm) (thanks [Richard Kulisz](http://richardkulisz.blogspot.com/2011/07/90-of-software-projects-fail.html) for the link to that - I knew the statistic from somewhere, but couldn't find it.) (And there's a lot of these 90/10 rules, aren't there? I wonder if there's a reason for that?). Since so many new software projects fail - and writing a new SIP channel driver would certainly be one - I figured refactoring the existing `chan_sip` would be a better course. But I was wrong.

Refactoring `chan_sip` would be the better course if it had more structure, but the truth is: it doesn't. Messages come in and get roughly routed to SIP message type handlers; that's about it. There's code that gets re-used for different types of messages, and change their behaviour based on the type of message. But a lot of that is just bad implementation and bad organizational design; worse is the bad end user design. You can't fix `users.conf`; you can't fix peers/friends/users in `sip.conf`. (And no, the rules for peers versus friends isn't consistent.) Those are things that even if you had a perfect implementation you'd still have to live with, and that's not something I think any one wants to support forever.

But, I don't believe that not liking something is justification for replacing it. After all, `chan_sip` makes a lot of phone calls. And that's not nothing. So, why write a new SIP channel driver?

As someone who has to act as the Scrum master during sprint planning and our daily scrums, I know we spend a lot of time on `chan_sip`. Running the statistics, about 25% of the bugs in Asterisk are against `chan_sip`; anecdotally, half of the time we spend is on `chan_sip`. When we're in full out bug fix mode - which is a lot of the time - that's half of an eight man development team doing nothing but fixing bugs in one file. Given its structure, even with a lot of [testing](http://svn.asterisk.org/svn/testsuite/asterisk/trunk/tests/channels/SIP/), we still find ourselves introducing bugs as we fix new ones. Each regression means we spent more than twice the cost on the original bug: the cost to fix it, the cost to deal with triaging and diagnosing the resulting regression report, the cost to actually fix the regression, the impact to whatever issue doesn't get fixed because we're now fixing the regression, the cost to write the test to ensure we don't regress again, etc. What's more, all of the time spent patching those bugs is time that isn't spent on new features, new patches from the community, improving the Asterisk architecture, and generally moving the project forward.

Trying to maintain `chan_sip` is like running in place: you're doing a lot, but you aren't going anywhere.

A few weeks before AstriDevCon in 2012, we were convinced that we should do something. There wasn't any one meeting, but over the months leading up to it, the thought of rewriting `chan_sip` crossed a number of people's minds. There were a few factors converging that motivated us prior to the developer conference:

* In my mind, the knowledge that we were spending half of the team's energy on merely maintaining a thing was motivation enough to do something about it.

* The shelving of Asterisk SCF. Regardless of the how and the why that occurred, the result was that we had learned a lot about how to write a SIP stack that was not `chan_sip`. Knowing that it could be done was a powerful motivator to do it in Asterisk.

* Asterisk 11. We had spent a lot of time and energy making that as good of an LTS as we thought we could: so if we were going to do something major in Asterisk, the time was definitely now.

* As we went into AstriDevCon, the foremost question was, "would the developer community be behind it? Would they want to go along with us?"

As it turned out: yes. In fact, I wasn't the first one to bring it up at AstriDevCon - Jared Smith (of [BlueHost](http://www.bluehost.com/)) was. And once the ball got rolling, there wasn't any stopping it.

The rest, as they say, is [history](https://wiki.asterisk.org/wiki/display/AST/AstriDevCon+2012#AstriDevCon2012-ChannelDrivers). And so, on Friday, the successor to `chan_sip` was released to the world.

There's a long ways to go still. Say what you will about `chan_sip` (and you can say a lot), it is interoperable with a lot of devices, equipment, ITSPs, and all sorts of other random SIP stacks. It does "just" work. And `chan_pjsip` - despite all of our testing and the knowledge that it is built on a proven SIP stack, PJSIP - has not been deployed. It will take time to make it as interoperable as `chan_sip` is. But we'll get there, and the first biggest step has been taken.

The best part: all of the above was but one project in Asterisk 12! Two equally large projects were undertaken at the same time - the core refactoring and Stasis/ARI - because if you're going to re-architect your project, you might as well do as much as you can. But more on that another time.
