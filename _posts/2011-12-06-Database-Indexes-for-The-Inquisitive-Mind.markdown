---
layout: post
title: Database Indexes for The Inquisitive Mind
---

I've used to be a developer advocate an awesome database product called [MarkLogic][dmc], a NoSQL Document Database for the Enterprise. Now it's pretty frequent that people ask me about database stuff.

In here I'm going to try to explain some fun stuff you can do with indexes. Not going to talk about implementing them but just about what they solve.

The point here is to help you reason about the choices you have when you are implementing stuff to speed up your applications. I'm sure if you think an idea is smart and fun you'll research what's the best algorithm to implement it.

If you are curious about MarkLogic you can always check the [Inside MarkLogic][insidemarklogic] white-paper.

## Range Indexes

The most frequent type of indexes in a database are `range indexes`. They allow you to do really fast order bys, count, aggregates, etc. Let's think about a `location` index. I can define a index that says if a document contains a `json property` called `local` then add that property to a range index called `location` treating that value as a string 

| Index     | Count | Documents |
| Algeria   | 2     | C,D       |
| Australia | 1     | A         |
| Canada    | 5     | A,B,C,D,E |
| Portugal  | 3     | A,B,C     |
| Togo      | 5     | B,C,D,E,F |

This means that document C and D have `local` Algeria and so on. So now I can ask the database to `give me the list of countries by first letter (including frequencies)`:

```
A (3)
C (5)
P (3)
T (5)
```

You can now display this to the user and they can use it to drill down in the content, even if visually it's impossible to display all the option that exist. You could also combine this with other visualizations that could, for example, say `choose locations in countries started with A and that are ruled by evil dictators`. You would just need to add another index to the `evil dictators json property`.

Now considering this a use can press the `A (3)` tab. In the index you can slice it up and get these two rows of documents. Now do a merge sort and you get:

```
A,C,D
```

Meaning document A, C, and D are located in countries that start with the letter A.

Same technique can be used for sorting documents, and executing fast aggregates, etc. You can normally keep a bunch of these in memory cause they are fairly lean indexes, and if you created them you probably need them! This is a technique that also allows you to do ranges in dates and express stuff like `in the last 3 days`, or `during the eighties (decade)`, etc. Super cool and super useful.

One thing this index does not give you is `what are the locations that belong to document A`. This would be an equivalent of a full table scan. So you can create the other way around, meaning associate documents with the locations they have. For this example that would be:

| Document  | Count | Locations |
| A         | 3     | Australia, Canada, Portugal | 
| B         | 3     | Canada, Portugal, Togo |
| C         | 4     | Algeria, Canada, Portugal, Togo |
| D         | 3     | Algeria, Canada, Togo |

Now it's fairly trivial to say the locations for document A isn't it? :) So just create this by omission when your use asks for a range index on location and he can have both. :)

The disadvantage with range indexes is you have to define them to use them, meaning if you forget to create an index and then do a ad-hoc query performance will suck. Or it will timeout. It will likely timeout if you are doing anything serious with the data. Full table scans take time.

## Inverted Index

Inverted indexes are what power search engines today, and for me one of the most revolutionary thing that happened to databases up until now. We all accept that full text search sucks in databases right? 

However search engines showed us the value behind this structure: Gives me any text in any form and ask any question and against words, I can answer quickly. ALL the frickin full internet. Yeah!

A inverted index answers questions like `find me document that contain the word blue but not the word black.` They are kind of like the index in the back of a book. You can just see what pages the word blue appears on (let's call it set A), and then what pages the word black appears (set B). What we are looking for is:

```
A EXCEPT B
```

And we can just go on adding constraints. The cool thing about it is that with a inverted index the more conditions you add you diminish the query granularity, which normally translate to less io and CPU, which means faster queries.

A inverted index looks a lot like a hash table. You hash the word and place it in a hash table. Then, like in the range index, you keep an array of the document that match that term. Unlike the range index the inverted index is hashed, thus not ordered. Unlike the range index the inverted index is not lean and indexes every single word it find in a document.

| Term     | Term List |
| red      | C         |
| blue     | A,B       |
| black    | A         |
| run      | A         |
| running  | B,C       |

This is a hash table, unordered and you don't have access to the keys. If you ask for words started with B this index is useless, you can only find things after running them thru the hash function. However this makes things like stemming super easy. When hashing you can coalesce words like run, running, ran to the same hash. This mean you can understand these words are the same for the purpose of the search directly out of the index. Actually if you stem terms before hashing them you loose the ability to distinguish if the word was run or running.

Every time you insert a document you need to go thru every word and add it to this index. So if you have a document with 80 thousand words that document triggers 80 thousand updates to the index. This takes time. 

Since you can't really control the complexity of the indexing algorithm (considering you are a smart guy and implemented the most efficient algorithm for your problem) all you can do is control the `n`. 

In other words you can have all indexes updating a single entry, the giant index, and then you cant give the user guarantees of when it will be ready (other than eventually). Or you can have the index partitioned so that you control the `n`, this way you can paralyze better and give real time results to your user. The problem with this approach is that query performance degrades with partitioning (e.g. the index for the word blue now exists in multiple partitions) and you need to compact your indexes eventually. The partitioning technique MarkLogic and other NoSQL databases use is the [LSM-Tree Cache][lsm], the versioning and compaction at the database level technique is called [Multi-Version Concurrency Control][mvcc].

In MarkLogic there's actually a fun twist to this: They only do writes in memory and keep the indexes and documents in a buffer (it's double buffered so when a flush happens you don't have to wait for a new buffer to be ready). Writes are journaled into disk so that if a computer crashes, MarkLogic can recreate the indexes and memory artifacts from index. So everything is written to memory and when it's full it gets flushed to disk. The actually compaction part of the LSM-Tree happens on the artifacts that are in disk and not in memory.

Inverted indexes are super fun and they should be in the core of any modern database systems. They will in some time :)


## Universal Index

So with the universal index I can ask any question that goes against words and get real fast responses on "ad-hoc" queries without doing a full table scan. Sweet. But if I need anything that relates to json I'm in trouble, the inverted index only indexes words.

This is where the guys at MarkLogic invented something super cool called the Universal Index. The idea is when you are indexing words you also index the structure of the document. First let me tell you a story so you understand why the universal index works on parent child associations to store structure.

How would you create an entry in the inverted index to find a phrase? 

Imagine I'm looking for the phrase "something wrong with" in document A

```
There's something wrong with me, I'm a cuckoo
```

If you use a normal inverted index you can find document that have the word "something", documents that have the word "wrong", and documents that have the word "with". But loads of those documents that do have all those terms won't have the sentence. In an ideal world, for this search, you would be grouping terms 3 by 3 to power this search:

| Term | Term List |
| There's something wrong | A |
| something wrong with | A |
| wrong with me | A |
| with me im | A |
| me im a | A |
| im a cuckoo | A |

Now if I did that search I wouldn't find any false positives in the index, which means smaller query granularity, which normally translates to a faster query. However with this index you can't search for two word phrases. So the default in most search indexes is to index phrases by grouping words two by two.

| Term | Term List |
| There's something | A |
| something wrong | A |
| wrong with | A |
| with me | A |
| me Im | A |
| Im a | A |
| a cuckoo | A |

If you find for sentences that are more than two words you can still have false positives but the query granularity is probably much better and will likely work.

So why all this now? In the universal index you augment the inverted index with structure about documents. So things like parent child relationships are stored. Things like the value of a property is stored. This augments the inverted index and makes it super useful. e.g. for the following document A:

``` js
{ "site":
  { "name": "github"
  , "description": "social coding done right"
  }
, "owner": "Pedro"
}
```

would produce the following index:

| Term | Term List | 
| Word:github | A |
| Word:Social | A |
| Word:Coding | A |
| Word:Done   | A |
| Word:Right  | A |
| Property:name->github | A |
| Property:descrition->social coding done right | A |
| Property:owner->Pedro | A |
| Hierarchical:root.site1 | A |
| Hierarchical:root.site1.name | A |
| Hierarchical:root.site1.description | A |
| Hierarchical:root.owner | A |

And now we can answer super complicated questions like "give me documents that have the word github but not the term red that are owned by pedro and are named github".

MarkLogic took this to another level by adding security, collections (kind of gmail labels) and even structuring directories using a inverted index. The beauty of it is the more complicated you make it the faster the query returns.

e.g. If I have 1 billion of documents but only 10 are mine the security can see, right from the index, that I can only see those 10 documents. So the maximum IO I can do is 10, even in such a large dataset.

## Conclusion

There's some more fun stuff I could right about but maybe in another article.

Fun stuff like how to store what your user likes and have indexes that help you alert in scale, or register queries in your system, or even using map reduce to queries views in CouchDB.

Feel free to check the [Inside MarkLogic Paper][insidemarklogic], it goes into infinite more detail than this text.

[dmc]: http://developer.marklogic.com
[insidemarklogic]: http://www.odbms.org/download/inside-marklogic-server.pdf
[lsm]: http://nosqlsummer.org/paper/lsm-tree
[mvcc]: http://en.wikipedia.org/wiki/Multiversion_concurrency_control