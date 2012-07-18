---
layout: post
title: How To Update A Document With Nano (The CouchDB Client for Node.js)
---

# How To Update A Document With Nano (The CouchDB Client for Node.js)

CouchDB is [MVCC] so updates are not done in place. When you insert a document into CouchDB a pointer will say "for this URI this is the current version of the document".

<img src="/images/nano-diagram1.png" alt="CouchDB MVCC"></img>

So in a sense there's no updating in a MVCC database, updates mean changing a pointer.

In `nano` I deliberately tried not to have parts of the api called "update", or "connect" since those things are not things that you do in CouchDB. In CouchDB, you insert:

      // insert {foo: "baz"} into the "foobaz" document
      db.insert({"foo": "baz"}, "foobaz", function (error, foo) {   
        if(!err) {
          console.log("it worked");
        } else {
          console.log("sad panda");
        }
      });

If you need to `update` a document then you should just insert again (but specifying the revision you are updating):

      db.insert({"foo": "bar"}, "foobar", function (error, foo) {
        if(err) {
          return console.log("I failed");
        }
        db.insert({foo: "bar", "_rev": foo.rev}, "foobar", 
        function (error, response) {
          if(!error) {
            console.log("it worked");
          } else {
            console.log("sad panda");
          }
        });
      });

You need to specify the revision so that CouchDB can make sure for you that no one did conflicting updates while you where editing the document. If the `rev` you send to CouchDB is not the latest `rev` you will get a conflict.

You can also use design documents to perform updates in CouchDB. Read more on how you can do that with nano at [JackRussell](http://jackhq.tumblr.com/post/16035106690/nano-v1-2-x-document-update-handler-support-v1-2-x).

[MVCC]: http://en.wikipedia.org/wiki/Multiversion_concurrency_control