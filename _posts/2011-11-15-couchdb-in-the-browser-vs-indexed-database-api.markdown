---
layout: post
title: CouchDB in the Browser vs. Indexed Database API
---

# CouchDB in the Browser vs. Indexed Database API

There's something fundamentally wrong with the way we do browser apps in javascript which can be described in a single sentence: You can't use a local  database. Or a local search engine.

Even if you try to build your own abstractions you will never be able to build something that can work minimally well as databases go. e.g. javascript doesn't allow you to create custom datatypes that are optimized for database processing, such as BTrees and stuff.

The [Indexed DB API][idb] tries to formulate the minimal common denominator, the things that browser vendors must provide to you so you build your own databases using javascript.

This is much better than what we have right now so you would expect a database geek such as myself to feel ecstatic about the [Indexed DB API][idb]. I am happy about it but still have some really big concerns about it:

Indexed DB API is not a finished product. Developers need something that can sync our data to another machine, that can store JSON, and where we can run some queries. Maybe something that can gives us push notifications in a flexible way? The Indexed DB API focus in none of these things.

The Indexed DB API specification is heavily relational geared in nature. One might say I'm being untrue, but if they are they probably never worked in a document database. You can read about cursors, transactions, and all sort of things that I would not expect to be in early drafts of a database API for the web. At least I wouldn't.

The Indexed DB API is years from mainstream usage when we need it now! This is my biggest pain point with it: We need to wait for a recommendation, then people will build products (hopefully). Then we need to wait for browsers to catch up and then we need to wait for users to upgrade their browsers. Not easy considering how many people still use IE 6 today.

It looks like we are in the HTML5 vs. XHTML standoff again. This is taking way to long for thing we needed yesterday. I for once think HTML5 was a great thing and broke free from endless boring discussion about making the perfect markup language.

I for once would just vote for putting CouchDB in the browser and standardize it's HTTP API and replication engine. CouchDB is already an Apache project and is extremely successful in doing the stuff we need to do in web applications. Why reinvent the wheel? Take it and build your standard around it, not some relational biased IDB that people will then implement CouchDB on top of.

I would love to hear why this is such a terrible idea. Base the standard on something that works today and modify it accordingly to what the browser needs?

*PS* I have no interest in contributing to confusion or instigate anyone. My solo intention is to have this shit running before I have grey hair and look like Jack Nicholson. If you are looking for trolling I suggest you look somewhere else.

[idb]: http://www.w3.org/TR/IndexedDB/