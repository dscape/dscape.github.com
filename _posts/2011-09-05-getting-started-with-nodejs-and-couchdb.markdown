---
layout: post
title: Getting Started with nodejs and CouchDB
---

# Getting Started with nodejs and CouchDB

After seeing some questions on [stack-overflow][5] about getting started with CouchDB and nodejs decided to give it a go at answering one of them. Hopefully this will help other people with similar issues!

Let's start by creating a folder and installing some dependencies:

    mkdir test && cd test
    npm install nano
    npm install express

If you have CouchDB installed, great. If you don't you will either need to install it setup a instance online at [iriscouch.com][1]

Now create a new file called `index.js`. Inside place the following code:

    var express = require('express')
       , nano    = require('nano')('http://localhost:5984')
       , app     = module.exports = express.createServer()
       , db_name = "my_couch"
       , db      = nano.use(db_name);
    
    app.get("/", function(request,response) {
      nano.db.create(db_name, function (error, body, headers) {
        if(error) { return response.send(error.message, error['status-code']); }
        db.insert({foo: true}, "foo", function (error2, body2, headers2) {
          if(error2) { return response.send(error2.message, error2['status-code']); }
          response.send("Insert ok!", 200);
        });
      });
    });
    
    app.listen(3333);
    console.log("server is running. check expressjs.org for more cool tricks");

If you setup a `username` and `password` for your CouchDB you need to include it in the url. In the following line I added `admin:admin@` to the url to exemplify

    , nano    = require('nano')('http://admin:admin@localhost:5984')

The problem with this script is that it tries to create a database every time you do a request. This will fail as soon as you create it for the first time. Ideally you want to remove the create database from the script so it runs forever:

    var express = require('express')
       , db    = require('nano')('http://localhost:5984/my_couch')
       , app     = module.exports = express.createServer()
       ;
    
    app.get("/", function(request,response) {
        db.get("foo", function (error, body, headers) {
          if(error) { return response.send(error.message, error['status-code']); }
          response.send(body, 200);
        });
      });
    });
    
    app.listen(3333);
    console.log("server is running. check expressjs.org for more cool tricks");

You can now either manually create, or even do it programmatically. If you are curious on how you would achieve this you can read this article I wrote a while back [Nano - Minimalistic CouchDB for node.js][4].

For more info refer to [expressjs][2] and [nano][3]. Hope this helps!

[1]: http://iriscouch.com
[2]: http://expressjs.org
[3]: http://github.com/dscape/nano
[4]: http://writings.nunojob.com/2011/08/nano-minimalistic-couchdb-client-for-nodejs.html
[5]: http://stackoverflow.com/questions/tagged/couchdb+node.js