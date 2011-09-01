---
layout: post
title: nano - minimalistic CouchDB client for nodejs
---

# nano - minimalistic CouchDB client for nodejs

## Motivation

In some of my [nodejs][10] projects I was using [request][1] to connect to [CouchDB][11]. As [Mikeal Rogers][3], the author of request, would have said [CouchDB and nodejs are a perfect fit][2]. I would argue that request is the perfect glue to binds nodejs and CouchDB together. Request is easy to use, and easy to reason with when you hit a problem.

One of the coolest things about request is that you can even proxy request from CouchDB directly to your end user using nodejs `stream#pipe` functionality.

After doing development like this for a while some obvious patterns started to emerge, as well as some code duplication. So the idea of `nano` was born: Build the minimal abstraction possible that allows you to use CouchDB from nodejs while preserving `stream#pipe` capabilities.

The result is a [very clean code base][4] based entirely on request.

## Show me the code

For your convenience I added all the code snippets to a [gist][9].

You can install nano using [npm][8]:

      mkdir nano_sample && cd nano_sample
      npm install nano

If you don't have CouchDB installed I would recommend using [Iris Couch][5]. You can sign up in less than a minute and you will have your CouchDB up and running. 

Now we can give nano a try:

     node
     var nano = require('nano')('http://localhost:5984');
     nano;
     // { db: 
     //  { create: [Function: create_db],
     //    get: [Function: get_db],
     //    destroy: [Function: destroy_db],
     //    list: [Function: list_dbs],
     //    use: [Function: document_module],
     //    scope: [Function: document_module],
     //    compact: [Function: compact_db],
     //    replicate: [Function: replicate_db],
     //    changes: [Function: changes_db] },
     // use: [Function: document_module],
     // scope: [Function: document_module],
     // request: [Function: relax],
     // config: { url: 'http://localhost:5984' },
     // relax: [Function: relax],
     // dinosaur: [Function: relax] }

One cool thing about nano is that you don't have to learn about errors: they are proxied directly from CouchDB. So if you knew them in CouchDB, you know them in nano. The only error nano introduces is a socket error, meaning the connection to CouchDB failed.

This makes it super easy for someone that knows CouchDB to use nano.

One common pattern I see in people developing CouchDB centric applications is lazy creation of databases. In other words you try to create a document, if the database doesn't exist then you create a database and retry. Let's see how that would work in nano:

      // don't forget to add your credentials if you are not in admin party mode!
      var nano = require('nano')('http://localhost:5984');
      var db_name = "test";
      var db = nano.use(db_name);
      
      function insert_doc(doc, tried) {
        db.insert(doc,
          function (error,http_body,http_headers) {
            if(error) {
              if(error.message === 'no_db_file'  && tried < 1) {
                // create database and retry
                return nano.db.create(db_name, function () {
                  insert_doc(doc, tried+1);
                });
              }
              else { return console.log(error); }
            }
            console.log(http_body);
        });
      }
      
      insert_doc({nano: true}, 0);

We use `nano.use(db_name)` to instruct nano to operate on that database. In nano all callback return three arguments: 1) errors, 2) http headers returned from couch, 3) the http body. That's why we can say `if(error.message === 'no_db_file'  && tried < 1)`: because we get the error message that was proxied from CouchDB. Here's a [gist with some verbose output from the execution of this code][6].

If you are an absolute beginner in nodejs there's two things here that might confuse you:

* The `callback` argument: nodejs is evented so that is the function that will be executed when the http request is complete.
* The `error` argument: In nodejs it's a best practice to return error as the first argument of your callback. Keep this is mind when developing nodejs applications as this is a very important (and helpful) pattern.

Because nano is minimalistic it doesn't try to support every single thing you can do in CouchDB. The way nano allows you to extend that functionality is by using the `request` method:

      var nano = require('nano')('http://localhost:5984');
      nano.request({db: "_uuids"}, function(_,uuids){ console.log(uuids); });

## Hello Pipe!

Let's try to use nano to pipe something from CouchDB using the [express][7].

      npm install express
      npm install request

We need something we can pipe out, so let's pipe the nodejs logo into couchdb:

      node
      // alias for require('nano')('http://localhost:5984').use('test');
      var db      = require('nano')('http://localhost:5984/test');
      var request = require('request');

      // {} for empty body as parameter is required but will be piped in
      request.get("http://nodejs.org/logo.png").pipe(
        db.attachment.insert("new", "logo.png", {}, "image/png")
      );

If you visit futon (i.e. `localhost:5984/_utils/`) you should be able to see the nodejs logo inside the `test` database, in document `new`, in an attachment called `logo.png`.

What if instead we want to pipe the attachment from CouchDB to the end user?

      vi index.js
      var express = require('express')
        , nano    = require('nano')('http://localhost:5984')
        , app     = module.exports = express.createServer()
        , db_name = "test"
        , db      = nano.use(db_name);

      app.get("/", function(request,response) {
        db.attachment.get("new", "logo.png").pipe(response);
      });

      app.listen(3333);
      

Now go to your browser and visit `localhost:3333`. You should be able to see the nodejs logo!

Hope you had fun following this little experiment -- feel free to ask questions in the comments.

[1]: https://github.com/mikeal/request
[2]: http://jsconf.eu/2010/speaker/nodejs_couchdb_crazy_delicious.html
[3]: http://www.mikealrogers.com
[4]: https://github.com/dscape/nano/blob/master/nano.js
[5]: http://www.iriscouch.com/
[6]: https://gist.github.com/6b2c6f5cc0711131feea
[7]: http://expressjs.com/
[8]: http://npmjs.org/
[9]: https://gist.github.com/e20560c9879efdcbc28c
[10]: http://nodejs.org/
[11]: http://couchdb.apache.org/