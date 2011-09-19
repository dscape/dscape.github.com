---
layout: post
title: So you think you can build a document database?
---

# So you think you can build a document database?

We all know how relational databases work. We all know very well how to solve the problem of squeezing data into tables and getting answers out of it using the old SQL dialect.

But what about when we have a document database? How can we allow our document to remain in their original shape and still get any answer we want using newer database dialects like XQuery or JavaScript? How would you engineer a database for unstructured data?

Many have tried.  Search engines do it by... not being a database! They give away query time flexibility so you can index massive amounts of textual documents. If you want to do a text search, they're great, but if you want to treat documents like a database - issuing ad hoc queries that understand the document structure - they can't.

Other document databases like CouchDB create something like serialized views of the data that give you query performance at the cost of ad-hoc queries. Others like MongoDB allow you to create relational-like indexes on top documents in a somehow flexible way by giving up on transactional guarantees.  If you want ad-hoc queries and transaction guarantees, you need something else.  If you want full-text search you also need something else.

In MarkLogic we pride ourselves in having a high throughput, ACID compliant, fast ad-hoc query engine supported by both inverted indexes (that make MarkLogic look like a search engine) as well as range indexes (which are more common in relational-land).

MarkLogic doesn't make compromises.  You can issue ad-hoc queries that understand the document structure.  You can have transactional guarantees.  You can run full-text queries, or database-style value or scalar queries, all in one and with ACID guarantees.

In Berlin I got the chance to introduce our architecture in a session named "ACID Transactions at the PB Scale with MarkLogic Server". I invite you all to watch it and challenge me with your questions.

<iframe src="http://player.vimeo.com/video/26777627?byline=0&amp;portrait=0&amp;color=c9ff23" width="601" height="338" frameborder="0" webkitAllowFullScreen allowFullScreen>
  
</iframe>
<a href="http://vimeo.com/26777627">
Nuno Job ACID TRANSACTIONS AT THE PB SCALE WITH MARKLOGIC SERVER
</a>

If afterwards you feel like you wish you knew (even more) about how MarkLogic works feel free to check the [Inside MarkLogic Server][1] white-paper!

Happy monday guys!

[1]: http://www.odbms.org/download/inside-marklogic-server.pdf