---
layout: post
title: Why SQL Sucks for NoSQL Unstructured Databases
---

# Why SQL Sucks for NoSQL Unstructured Databases

As some of my readers know I have now worked in two document databases: [IBM pureXML][1], a native XML database built on top of a relational engine _(pun intended)_ that offers both relational ([SQL/XML][2]) and unstructured ([XQuery][3]) query languages, and [MarkLogic][4], a database built from scratch on a new database paradigm (call it NoSQL if you like) that understands unstructured data and offers an unstructured query language ([XQuery][3]).

Another relevant tidbit of information is this emerging trend amongst NoSQL database vendors to implement SQL (or SQL-like interfaces). An example would be the recent push on [Cassandra][7] with [CQL][5], or even the more mature [Hadoop based SQL interfaces][6]. I see this as NoSQL trying to grow enterprise which overall is a good thing.

I'm not going to argue on whether these NoSQL vendors are doing the right choice with SQL, or even to talk about the fact that enterprise is about more than just bolting on a SQL interface. I'm also not going to discuss why some data models lend themselves better to SQL than others, e.g. [Cassandra][7] vs. [MongoDB][8] _(But if you want to discuss those topics just leave a comment)_.

In this post I'll focus on some lessons learned about mixing the worlds of relational and unstructured databases. 

## When the Two Worlds Collide

[NoSQL is about No SQL][9]. What this means to me is a shift of focus towards non-relational database alternatives that might even explore different interfaces to the database _(and not caring about being politically correct)_. That is a good thing! Blindly accepting the suckyness of SQL for the sake of enterprise? Well, even if SQL is the right choice for your product, you still need to reason about the consequences and make sure things are well aligned between the two worlds. In other words, it means removing the "blindly" part and reducing the "suckyness" to a bearable minimum for your developers. 

But be warned: things will get messy. SQL isn't pretty and it's about to collide with the awesome unstructured truck _(Slightly biased)_!

![Calvin and Hobbes](img/calvin-mess.gif)

## Data Model

In relational you have:

      RowSet -> SQL -> RowSet

A RowSet is something like:

     RowSet -> Item+
     Item   -> INT | VARCHAR n | ...

I'm unaware of a data model for JSON so I'll talk about data model I'm fairly familiar with: the XPath Data Model:

     XDM -> XPath/XQuery -> XDM

And the [XDM][10] is something like:

     XDM        -> Item+
     Item       -> AtomicType | Tree
     AtomicType -> integer | string | ...
     ...

_(Both these definitions are oversimplified but serve the purpose)_.

A thing that is different about a data model for document is that trees are not flat:

     {
       "namespace": "person-2.0",
       "comments": "This guy asked me for a dinosaur sticker. What a nutter!",
       "person": {
         "handle": "dscape",
         "comments": "Please do not send unsolicited mail."
       }
     }

So there's multiple interpretation to what this could mean:

     SELECT comments from PERSON where handle = "dscape"

What "comment" element is the query referring to? If you look at [SQL/XML][2] _(which is a terrible, terrible thing)_ your query would be something like:

     SELECT XMLQuery('$person/comments')
     FROM PERSON
     WHERE XMLExists('$person/person/handle')

Which brings me to this obvious conclusion: Trees need a way to navigate. In XML that is [XPath][11], in JSON maybe that will be [JSONSelect][12], maybe something else. But you still need a standard way to navigate first.

Something that makes this challenge even more interesting is schema versioning and evolution. While this has been ignored for ages in relational world _(with serious business implications due to downtime during those fun alter-table moments)_, it really, really, REALLY can't be ignored for documents. Think of Microsoft Word - how many different document versions do they support? Word 2003, 2005, etc.. 

Schema-less, Flexible, Unstructured: Pick your word but they all lend themselves to quick evolution of data formats. In this query we assume that handle is a child of person, and that the comments about me being an idiot are a direct descendent of the tree. This is bound to change. And SQL doesn't support versioning of documents, thus you will have to extend it so it does.

A true query language for unstructured data must be version aware. In XQuery we can express this query as something like:

     declare namespace p = "person-2.0" ;
     
     for $person in collection('person')
     let $comments-on-person := $person/p:comments
     where $person/p:handle = "dscape"
     return $comments-on-person

## Frankenqueries by Example

Someone once referred to me (talking about SQL/XML) as those Frankenqueries. The term stuck to my head up until now. Let's explore that analogy a little further and look for the places where the organic parts and bolts come together.

Let's imagine two shopping lists, one for Joe and one for Mary

      marys-shopping.json
      "fruit": {
        "apples": 2,
        "apples": 5
      }
      
      joes-shopping.json
      "fruit": {
        "apples": 6,
        "oranges": 1
      }

Now with my "make believe" SQL/JSON-ish extension I do:

      SELECT $lists.fruit.apples
      FROM LISTS

What does this return? Remember RowSet goes in, RowSet comes out?

      2, 5
      ---
      6

So, even though you are clearly asking for a list of quantities of apples, you get two RowSets instead of three, and one of the RowSets will have a list of quantities. If you instead decide to return three things, you had two RowSets come in and three RowSets come out. I'm no mathematician but that doesn't sound good.

Once again this is not a problem if you use something that can deal with unstructured information. You don't have this problem in javascript and certainly won't have it in XQuery. In both javascript and XQuery it's all organic. (or bolts if you prefer)

## Conclusion: The awesome languages for unstructured data, unicorns and pixie-dust!

While XQuery is a great language for unstructured information my point here is not advocating for it's use. The point I'm trying to make is the need for a real language for unstructured data, whatever you (read: the developers) choose it to be. 

But I do ask you (developers) not to accept the "suckyness of SQL" back. She's gone and you have this new hot date called NoSQL. Just give it some time and it will grow on you. Plus it's lots of fun writing javascript code that runs on databases: Don't let them take that away from you.

SQL for unstructured data will fail. Then PL-SQL for unstructured data will fail. So if a vendor is pushing something your direction don't settle for anything less than a full fledged programming language: you can write your full app in javascript and store it in a [CouchApp][14], or you can write your full app in XQuery and store it in MarkLogic. And it should remain like that!

Here's a checklist of things to look for on a query language for unstructured information  _(feel free to suggest some more)_:

* Navigation Language
* Data Model
* Regular expressions
* Lambdas
* High order functions
* Functional flavor
* Good string handling
* Modules so you can build your own libraries
* App Server aware: has functions that serve REST

You can choose to ignore this advice but you might end up feeling like a [frustrated silverlight developer][13]. And we, the guys that love to innovate in databases, will feel frustrated that the developers have chosen to accept the suckyness back!

## See you at Open Source Bridge

If you want to talk more about this topic I would like to invite you to join [me][16], [J Chris Anderson][15] ([CouchDB][14]) and [Roger Bodamer][17] ([MongoDB][8]) at [Open Source Bridge][20] in Portland this month. We will be hosting a panel-ish un-talk about data modeling in a session called [No More Joins][18]. So go on [register][19] and we will see you there!

[1]: http://www-01.ibm.com/software/data/db2/xml/
[2]: http://en.wikipedia.org/wiki/SQL/XML
[3]: http://www.w3.org/TR/xquery-30/
[4]: http://developer.marklogic.com
[5]: http://www.slideshare.net/shotaz/cql-cassandra-query-language
[6]: http://wiki.apache.org/hadoop/Hive
[7]: http://cassandra.apache.org/
[8]: http://www.mongodb.org/
[9]: http://kellblog.com/2010/09/29/nuno-jobs-nosql-frankfurt-presentation/
[10]: http://www.w3.org/TR/xpath-datamodel/
[11]: http://www.w3.org/TR/xpath-datamodel-30/
[12]: http://jsonselect.org/
[13]: http://forums.silverlight.net/forums/t/230502.aspx
[14]: http://couchdb.apache.org/
[15]: http://twitter.com/jchris
[16]: http://twitter.com/dscape
[17]: http://twitter.com/rogerb
[18]: http://opensourcebridge.org/sessions/524
[19]: http://osbridge.eventbrite.com/
[20]: http://opensourcebridge.org/