---
layout: post
title: So you think you can build a document database?
---

# So you think you can build a document database?

We all know how relational databases work. We all know very well how to solve the problem of squeezing data into tables and getting answers out of it using the old SQL dialect.

But what about when we have a document database? How can we allow our document to remain in their original shape and still get any answer we want using newer database dialects like XQuery or javascript? How would you engineer a database on unstructured data?

Many tried: Search engines do it by... not being a database! They give away query time flexibility so you can index massive amounts of textual documents. By doing that they loose out of the structure of the document and will only allow you to ask predefined questions, unlike SQL where you can ask a questions whether or not a specific index was created to help with that question (it will just be slower without the index)

Other document databases like CouchDB create something like serialized views of the data that allow you query performance at the cost of ad-hoc queries. Others like MongoDB allow you to create relational-like indexes on top documents in a somehow flexible way by giving up on any transactional guarantee whatsoever, and still in a quite unsatisfactory way in what search is concerned.

In MarkLogic we pride ourselves in having a high throughput, ACID compliant, fast ad-hoc query engine supported by both indexes that look like a search engine and in range indexes (which are more common in relational-land). 

With MarkLogic you can have transactions (indexes will be available right after you create them, plus consistency and durability), high throughput and clusters that scale up to PBs.

In Berlin I got the chance to introduce our architecture in a session named "ACID Transactions at the PB Scale with MarkLogic Server". I invite you all to watch it and challenge me with your questions.

<iframe src="http://player.vimeo.com/video/26777627?byline=0&amp;portrait=0&amp;color=c9ff23" width="601" height="338" frameborder="0" webkitAllowFullScreen allowFullScreen>
  
</iframe>
<a href="http://vimeo.com/26777627">
Nuno Job ACID TRANSACTIONS AT THE PB SCALE WITH MARKLOGIC SERVER
</a>

If afterwards you feel like you wish you knew (even more) about how MarkLogic works feel free to check the [Inside MarkLogic Server][1] white-paper!

Happy monday guys!

[1]: http://www.odbms.org/download/inside-marklogic-server.pdf