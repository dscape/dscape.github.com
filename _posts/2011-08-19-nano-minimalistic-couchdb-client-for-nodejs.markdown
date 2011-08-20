---
layout: post
title: nano - minimalistic CouchDB client for nodejs
---

# nano - minimalistic CouchDB client for nodejs

## Motivation

In some of my nodejs projects I was using [request][1] to connect to CouchDB. As [Mikeal Rogers][3], the author of request, would have said [CouchDB and nodejs are a perfect fit][2]. I would argue that request is the perfect glue to binds nodejs and CouchDB together. Request is easy to use, and easy to reason with when you hit a problem.

One of the coolest things about request is that you can even proxy request from CouchDB directly to your end user using nodejs `stream#pipe` functionality.

After doing development like this for a while some obvious patterns started to emerge, as well as some code duplication. So the idea of `nano` was born: Build the minimal abstraction possible that allows you to use CouchDB from nodejs while preserving `stream#pipe` capabilities.

The result is a [very clean code base][4] based entirely on request.

## Show me the code

You can install nano using npm:

      mkdir nano_sample && cd nano_sample
      npm install nano

If you don't have CouchDB installed I would recommend using [Iris Couch][5]. You can sign up in less than a minute and you will have your CouchDB up and running. 

     vi couch.cfg.js
     module.exports = { url: "http://localhost:5984" };

Now we can give nano a try:

     node
     var cfg = require('./couch.cfg');
     var nano = nano = require('nano')(cfg);

One thing that is cool in nano is that you don't have to learn about errors: they are proxied directly from CouchDB. So if you knew them in CouchDB, you know them in nano. The only error nano introduces is a socket error, meaning the connection to CouchDB failed.

This makes it super easy for someone that knows CouchDB to use nano.

One common pattern I see in people developing CouchDB centric applications is lazy creation of databases. In other words you try to create a document, if the database doesn't exist then you create a database and retry. Let's see how that would work in nano:

      var db_name = "test";
      var db = nano.use(db_name);
      
      function insert_doc(file_name, doc, tried,callback) {
        db.insert(doc, file_name,
          function (error,http_headers,http_body) {
            if(error) {
              if(error.message === 'no_db_file'  && tried < 1) {
                // create database and retry
                return nano.db.create(db_name, function () {
                  insert_doc(file_name, doc, tried+1, callback);
                });
              }
              else { return callback(error,http_headers); }
            }
            callback(null,http_headers,http_body);
        });
      }
      
      insert_doc("testfile", {nano: true}, 0,
        function(error,http_headers,http_body) {
          return http_body;
        }
      );

We use `nano.use(db_name)` to instruct nano to operate on that database. In nano all callback return three arguments: 1) errors, 2) http headers returned from couch, 3) the http body. That's why we can say `if(error.message === 'no_db_file'  && tried < 1)` and get the error message that was proxied from CouchDB. Here's a [gist with the output I got in node][6].

If you are an absolute beginner in nodejs there's two things here that might confuse you:

* The `callback` argument. nodejs works with a event loop, so that is the function that will be executed when the http request is completed.
* The `error` argument. In nodejs it's a best practice to return error as the first argument of your callback. Keep this is mind when developing nodejs applications, this is a very important (and helpful) pattern.

Because nano is minimalistic it doesn't try to support every single thing you can do in couchdb. instead it gives you a framework to do advanced requests by using the `request` method. 

For instance if you want to call `_uuids` you can:

      nano.request({db: "_uuids"}, function(_,_,uuids){ console.log(uuids); });

Don't forget to quit node (control+c) to continue.

## Hello Pipe!

Let's try to use nano to pipe something from CouchDB using the [express][7].

      npm install express
      npm install request

We need something we can pipe out, so let's pipe the nodejs logo into couchdb:

      node
      var cfg = require('./couch.cfg');
      var nano = require('nano')(cfg);
      var request = require('request');

      var db_name = "test";
      var db = nano.use(db_name);

      // {} for empty body as parameter is required but will be piped in
      request.get("http://nodejs.org/logo.png").pipe(
        db.attachment.insert("new", "logo.png", {}, "image/png")
      )

If you visit futon (i.e. `localhost:5984/_utils/`) you should be able to see the nodejs logo inside the `test` database, in document `new`, in an attachment called `logo.png`.

Let's do the other way around and pipe the attachment from CouchDB to the end user:

      npm install express
      vi index.js
      var express = require('express')
        , cfg     = require('./couch.cfg')
        , nano    = require('nano')(cfg)
        , app     = module.exports = express.createServer()
        , db_name = "test"
        , db      = nano.use(db_name);

      app.get("/", function(request,response) {
        db.attachment.get("new", "logo.png").pipe(response);
      });

      app.listen(3333);
      

Now go to your browser and visit `localhost:3333`. You should be able to see the nodejs logo!

Hope you had fun following this little tutorial, feel free to ask questions in the comments!

[1]: https://github.com/mikeal/request
[2]: http://jsconf.eu/2010/speaker/nodejs_couchdb_crazy_delicious.html
[3]: http://www.mikealrogers.com
[4]: https://github.com/dscape/nano/blob/master/nano.js
[5]: http://www.iriscouch.com/
[6]: https://gist.github.com/6b2c6f5cc0711131feea
[7]: http://expressjs.com/