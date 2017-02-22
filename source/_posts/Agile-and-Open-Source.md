---
title: Agile and Open Source
date: 2013-11-12 07:32:46
tags:
    - Agile
    - Open Source
    - Project Management
---
Googling any combination of the keywords Agile and Open Source yields dismal results. In general, what you'll find are a plethora of Open Source Agile tools (most of which are mediocre at best - whatever happened to "Individuals and interactions over processes and tools"?) and almost no source on marrying Agile and Open Source software development. To Twitter to complain!

> It's hard to find good literature on [#Agile](https://twitter.com/search?q=%23Agile&src=hash) processes used with [#FOSS](https://twitter.com/search?q=%23FOSS&src=hash) projects. I don't think that FOSS implies Agile; others apparently do.
> 
> — Matt Jordan (@mattcjordan) [November 11, 2013](https://twitter.com/mattcjordan/statuses/400012000236822529)

And thus enters the power of the internet. Not surprisingly, both Kevin and Russell commented back - and Russell linked to another [blog post](http://fnords.wordpress.com/2011/01/21/agile-vs-open/) and a [mailing list thread](http://lists.openstack.org/pipermail/openstack-dev/2013-April/007872.html) he was involved in that cast some cold water on marrying Agile - or at least Scrum - and FOSS.

*Disclaimer*: I'm a big fan of Agile development.

### Background

I came to Agile development painfully. I'd say prior to doing Agile development - in some flavor - I did one of two forms of development:

* *Heroic Development*: Also called "no process". This is where planning is minimal, requirements are barely known, and you only release software through the blood, sweat, and tears of you and your compatriots. The highs are high; the lows are abysmal. Projects typically succeed once (we released 1.0!) and then fail miserably as projects gain complexity. Things drift. I'd guess that many FOSS projects fall into the Heroic Development category, which may account for why so many fail (and many do fail - I don't think 95% is far off).

* *Mythical Man-Month Development*: This goes by many names. Waterfall. Spiral. Iterative (which is often confused with being Agile). It's an approach that marries well with 1970's programming. It sometimes makes sense with mission critical embedded software development, where environments are well known, requirements are set in stone, and releases are few (if not just one). In general, however, it is a terrible, terrible approach to software development for any modern project.

Working in one of those two development models for years taught me a lot. I watched a lot of projects fail despite my best efforts; I watched a lot of good code get thrown away because the customer (who is that? Write more code!) never knew what was being written for them. It made me yearn for some way of doing my job that didn't suck the life out of me. So I have a hard time thinking that Agile - which, while not perfect, is the closest I've seen to a sustainable, working model of software development - can't mesh with what is essentially a way of organizing a project. Agile doesn't say "close your source code!". Nor does being Open Source imply chaos.

So... wherein lies the conflict?

### What is Agile?

Agile is all the rage. There's [Agile](http://aspe-sdlc.com/adwords/scrum-master-training/?gclid=CInc1Mak3roCFU8V7AodIFkA9g) [Certification](http://www.pmi.org/Certification/New-PMI-Agile-Certification.aspx) [Courses](http://www.agilealliance.org/news/agile-certification-a-position-statement/) all over the internet; I'm not sure what a certificate in being Agile is supposed to tell me, or anyone else. There's [Agile](www.versionone.com/agile-tools/) [development](https://www.atlassian.com/agile/) [tools](http://www.rallydev.com/) everywhere you look ("eBook: Agile Portfolio Management Drives Innovation". I just threw up in my mouth a little). While I appreciate the usefulness of good tools, so much of this is obviously snake oil that it is hard to tell what is genuinely good from what is crap. When in doubt, I find it's always good to go back to the [source](http://agilemanifesto.org/).

* **Individuals and interactions over processes and tools**
* **Working software over comprehensive documentation**
* **Customer collaboration over contract negotiation**
* **Responding to change over following a plan**

Does anything advertised in the pages I linked to emphasize the concepts on the left side of those sentences? Not really... and I have a suspicion that people view Agile development through the prism that has been foisted on them by said salesman. So since the internet isn't much help on this one, how can we apply Agile methodologies to Open Source?

At first glance, nothing in the manifesto contradicts the principles underlying FOSS projects. In fact, every aspect of it feels directly in line with how FOSS projects are managed and run. FOSS thrives on individual interactions, although it tends to focus on IRC and mailing lists over personal interaction. FOSS is all about working software (and could usually use more documentation - after all, we still value the things on the right side of those sentences). FOSS is usually driven by customers: it is through deployments and customer needs that many developers on FOSS make their livelihood, and you'd be hard pressed to find any contracts in any official capacity in a FOSS project. FOSS projects are all about change: plans are usually loose and non-binding, if they exist at all.

So what's the problem?

### Implementation Concerns

As is so often the case, the devil is in the details. Implementations of Agile development tend to be hard to implement in open source projects. Extreme Programming emphasizes pair programming; that is obviously nearly impossible in a FOSS project. Demos are difficult - who do you demo to, and who is your product owner? Scrum has the daily standups, which are difficult to organize across a plethora of time zones (although that issue is not unique to FOSS: any company with distributed teams runs into that problem.) How do you organize a backlog? Or obtain consensus on said organization? How do you coordinate participation in user story development? The vary roles in said participation (scrum master/product owner/etc.) get fuzzy quickly.

Rather than get stuck in the muck on one particular Agile implementation, I think it's a good idea to look at the principles behind the manifesto. These tend to feed more directly into these implementations, and can show the root of some of these conflicts:

> *Our highest priority is to satisfy the customer through early and continuous delivery of valuable software.*

No problems here. Write software!

> *Welcome changing requirements, even late in development. Agile processes harness change for the customer's competitive advantage.*

Change is usually welcome in FOSS projects; I can't think of an instance where this wasn't the case. It's usually hard to think of cases where requirements were defined in stone in the first place.

> *Deliver working software frequently, from a couple of weeks to a couple of months, with a preference to the shorter timescale.*

This can be more difficult in FOSS. Everyone has a different time scale that they work on, and coordinating that across many organizations/individuals with different constraints is impossible. I think the important point here is to release as much as possible for the project, with a reasonably predictable schedule. We balance this in Asterisk - we have bug fixes every six months; new major versions every year. In general, this seems to be striking the right balance - but it's also always a contentious discussion every time we bring up potential changes. I'm not sure what the answer is here, but I can say that delaying releases has always backfired. The longer the gap in releases, the more difficult the recovery period is as we iron out things missed during the test cycles. Regression releases are common in such scenarios.

> *Business people and developers must work together daily throughout the project.*

No problems here - businesses often drive the features that go into FOSS and sponsor development.

> *Build projects around motivated individuals. Give them the environment and support they need, and trust them to get the job done.*

This is more of a dig against centralized command and control management than anything else. Companies that view two "Engineer 3" developers as being equivalent and interchangeable don't understand this concept.

> *The most efficient and effective method of conveying information to and within a development team is face-to-face conversation.*

And... here we are. This is the one that sticks in the craw of FOSS. But even if face-to-face conversation isn't always possible in a FOSS project, there are some points that I think that FOSS projects have to admit.

Face-to-face conversation is the most efficient and effective mode of conveying information. Nuance and subtleties don't translate well in IRC or mailing lists. Witness the power of FOSS conferences and many open source contributors desire to meet up with fellow contributors.

Even when face-to-face conversation isn't possible, open source projects do live and die by their communications. Negative communication, ill will, and malcontents are poison to the lifeblood of FOSS. Emphasizing positive, productive communication has to be a cornerstone of any FOSS project.

> *Working software is the primary measure of progress.*

Yup!

> *Agile processes promote sustainable development. The sponsors, developers, and users should be able to maintain a constant pace indefinitely.*

This is something FOSS could learn from. I think many newcomers to a FOSS project fall into the trap of taking on too much and getting involved in too many places. Participating at a constant, steady pace is often more valuable than someone who is involved for a short period of time and wanders off to another project when they burn themselves out.

> *Continuous attention to technical excellence and good design enhances agility.*

Again, this is something that many FOSS projects have learned at their own peril. Early in many FOSS projects, the tendency is to accept any code that "works" - for some definition of work (it worked on my machine! My customers love it!) This can lead to compromised design, which eventually constrains the project and makes it difficult to adapt to changing requirements. The chaotic nature of such projects can make it difficult to address long standing design flaws that creep in over time; a lack of adherence to technical excellence and good design can quickly exacerbate this situation. Witness the effort of unscrewing the damage of masquerades in Asterisk.

> *Simplicity--the art of maximizing the amount of work not done--is essential.*

This is rarely a problem in FOSS. In general, contributions are made because someone, somewhere, found them useful. Rarely is code contributed that has no practical purpose.

> *The best architectures, requirements, and designs emerge from self-organizing teams.*

This marries well with FOSS: the people participating in the project want to be there. Respect in a project is earned, not bestowed.

> *At regular intervals, the team reflects on how to become more effective, then tunes and adjusts its behavior accordingly.*

Again, this can be difficult for FOSS projects to do, but it can be done. Periodic retrospectives are possible for any project, even if they don't align with a "sprint".

### Conclusions?

After going through all of them, there aren't many conflicts. The difficultly seems to lie directly with the implementations of Agile, and not with Open Source. What is needed is an Agile Development methodology directed at Open Source projects, but one that still meets the principles that guide Agile methodology.

This has been some rambling thoughts, none of which resolved the reason I started thinking about this; namely, how do I better coordinate Agile development with running an Open Source project. There are still real world concerns to wrestle with, some of which still need some more thought.

I do think [Thierry Carrez](https://fnords.wordpress.com/2011/01/21/agile-vs-open/) nails the essential differences between Agile and Open Source:

> *The goals are also different. The main goal of Agile in my opinion is to maximize a development team productivity. Optimizing the team velocity so that the most can be achieved by a given-size team. The main goal of Open source project management is not to maximize productivity. It’s to maximize contributions. Produce the most, with the largest group of people possible.*

He's right; however, I have to quibble a bit. The tone makes it sound as if maximizing contributions will trump maximizing team productivity; while that can be true, it isn't always. This is a bitter pill for that most advocates of Open Source projects don't want to swallow. Not all contributions are created equal. A large number of trivial contributions are useful, but are not equivalent to a few good major, meaningful contributions. It's perfectly fine to maximize contributions in an open source project, but not at the expense of the larger strategic goals for the project. Agile feels like the answer to this; the question is how to implement it effectively while still maintaining the large collaborate nature of an Open Source project.

**UPDATE**

I published this post back in 2013. I ended up giving a talk on this topic in 2015 at [LeanDUS](https://www.leandus.de/2015/06/matt-jordan/), graciously hosted by [sipgate](https://www.sipgate.de/). I think it shows that while my thoughts on this topic have evolved some, it's still worth thinking through how you can apply an Agile philosophy to an Open Source project.
