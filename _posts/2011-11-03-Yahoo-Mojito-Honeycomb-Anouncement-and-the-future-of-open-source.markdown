---
layout: post
title: Yahoo Mojito ""Honeycomb"" Announcement (and the Future of Open Source)
---

# Yahoo Mojito ""Honeycomb"" Announcement (and the Future of Open Source)

Some thoughts on the recent release of [Yahoo's Mojito/Manhattan][1] announcement earlier yesterday.

I for one do not appreciate the way most companies do open-source and [the consequences that can have][2]. I've been annoyed at the way companies like google and 10gen promote their products as open source when in reality they are mostly maintained by a unique company and following the unique business interests of those companies.

But hey let's recognize the benefit here. MongoDB is use by thousands of happy developers, and I personally use both Chrome and Android. So it's easy to understand the value of these products and it's better to have some access to the source than none.

A word of appreciation goes for companies like Couchbase, Joyent, and nodejitsu that still manage to do big open source products but that are mostly "owned" by the community. Like node.js and CouchDB.

Now there was something about the Mojito announcement that unsettled me. First having the same code in the server and client is something that has been researched for many years know and everyone failed (to a degree or another, some in terms of tech, other in terms of no adoption).

This is a net win for me, I love it when people try to do something that everyone else failed at. Especially because it's fun to prove everyone's wrong. But from a developers perspective I don't know if this is a good or bad, but node.js looks like a pretty good fit to actually pull this off.

However I heavily disliked some things about the announcement:

* Made it look like this will be a giant monolithic system written on top yui. I absolutely do not think large monolithic systems are a good fit for node. And yui.
* This old model of doing open source and saying they will eventually release it. This is a framework, so why not just release it in github. And show developers they can trust you and that it is a community driven project? Somewhere in 2012? Common...

Personally I don't think a framework is enough value to justify my investment (as a developer) in something that is not "owned" by the community, so I won't use it. And I sure hope the future of node.js is not large monolithic frameworks.

From the business perspective my first impression was that yahoo was desperate to get some traction of their yahoo cloud platform. Or maybe they are just clueless? Maybe both?

[1]: http://developer.yahoo.com/blogs/ydn/posts/2011/11/yahoo-announces-cocktails-%E2%80%93-shaken-not-stirred/
[2]: 