---
layout: post
title: Mock HTTP Integration Testing in Node.js using Nock and Specify
---

# Mock HTTP Integration Testing in Node.js using Nock and Specify

One of the big things in releasing [nano] 3 was updating the tests. I really wanted this release out before [LXJS] but I had a lot of requirements for the tests:

* It should be possible to run all the tests with mocks on and off by simply switching a environment variable
* Tests should be as simple to read as any file in the `samples` directory
* Tests shouldn't be cluttered with mock code
* Mocks are not code - as such they should be specified in the `fixtures`
* It should be possible to write setup and teardown stages for all tests, and they should always be executed regardless of other tests (especially if other tests throw)
* Tests throwing shouldn't prevent other tests from running if they are in a separate file or unit
* You should run tests by doing `node tests/test-name.js` and not by executing an external binary
* Whatever runs the tests shouldn't hijack `stdin`, `stderr`, or `stdout`
* Descriptions of what the tests do are optional and no specific testing paradigm should be enforced. Tests are tests
* I should be able to specify what tests to run when executing the tests from the command line. e.g. `node my-test.js setup test1 test2` would run setup, then test1, then test2, but nothing else (including teardown)
* Tests should support streaming
* Tests should support timeouts for long running asynchronous functions

I decided to use [nock] and [specify] for this. `nock` because it's the only tool that can do this job. And `specify` because despite being written in 100 LOC it is the only tool that can fill all of these requirements. (disclaimer: I'm only familiar with `node-tap`, `mocha`, & `vows` so other testing things might be able to do all of these requirements)

In this article we are going to `npm install nano` and try to write some tests for it. This is strange but serves for demonstration purposes only. You can check the [nano] tree for more tests if you feel like checking more stuff out after you are done here.

      mkdir nano_mock_testing
      cd nano_mock_testing
      npm install nano specify nock

These are the versions that got installed in my pc

       nock@0.13.0 ./node_modules/nock 
       specify@0.4.0 ./node_modules/specify 
       ├── cycle@1.0.0
       ├── colors@0.6.0-1
       └── difflet@0.2.1
       nano@3.0.1 ./node_modules/nano 
       ├── errs@0.2.0
       ├── request@2.9.202
       └── follow@0.8.0

And my node versions

       $ node -e "console.log(process.versions)"
       { node: '0.6.7',
         v8: '3.6.6.15',
         ares: '1.7.5-DEV',
         uv: '0.6',
         openssl: '0.9.8r' }

Now we can test the insert functionality of nano. Let's write two tests, on testing a simple insert and another inserting functions inside documents. The test has a `setup` stage where we create a database and a `teardown` stage where we destroy the database we created. Because of specify can handle uncaught exceptions we know that teardown will always execute, so this won't affect your future tests even if you run in un-mocked mode.

      var specify  = require('specify')
        , helpers  = require('./helpers')
        , timeout  = helpers.timeout
        , nano     = helpers.nano
        , nock     = helpers.nock
        ;
      
      // this will work with nocks when you set NOCK=on
      // and without nocks when you don't set the environment variable
      // NOCK
      var mock = nock(helpers.couch, "doc/insert")
        , db   = nano.use("doc_insert")
        ;
      
      specify("setup", timeout, function (assert) {
        nano.db.create("doc_insert", function (err) {
          assert.equal(err, undefined, "Failed to create database");
        });
      });
      
      specify("simple", timeout, function (assert) {
        db.insert({"foo": "baz"}, "foobaz", function (error, foo) {   
          assert.equal(error, undefined, "Should have stored foo");
          assert.equal(foo.ok, true, "Response should be ok");
          assert.ok(foo.rev, "Response should have rev");
        });
      });
      
      specify("functions", timeout, function (assert) {
        db.insert({fn: function () { return true; },
        fn2: "function () { return true; }"}, function (error, fns) {   
          assert.equal(error, undefined, "Should have stored foo");
          assert.equal(fns.ok, true, "Response should be ok");
          assert.ok(fns.rev, "Response should have rev");
          db.get(fns.id, function (error, fns) {
            assert.equal(fns.fn, fns.fn2, "fn matches fn2");
            assert.equal(error, undefined, "Should get foo");
          });
        });
      });
      
      specify("teardown", timeout, function (assert) {
        nano.db.destroy("doc_insert", function (err) {
          assert.equal(err, undefined, "Failed to destroy database");
          assert.ok(mock.isDone(), "Some mocks didn't run");
        });
      });
      
      specify.run(process.argv.slice(2));



This won't run. We reference a `helpers.js` that is not yet there. Let's create it. It needs to add functionality to run tests in mocked and un-mocked mode, and expose things like the CouchDB configuration and default timeouts.

      var path    = require('path')
        , fs      = require('fs')
        , cfg     = {couch: "http://localhost:5984", timeout: 50000}
        , nano    = require('nano')
        , helpers = exports
        ;
      
      function endsWith (string, ending) {
        return string.length >= ending.length && 
          string.substr(string.length - ending.length) == ending;
      }
      
      function noop(){}
      
      function fake_chain() {
        return {
            "get"                  : fake_chain
          , "post"                 : fake_chain
          , "delete"               : fake_chain
          , "put"                  : fake_chain
          , "intercept"            : fake_chain
          , "done"                 : fake_chain
          , "isDone"               : function () { return true; }
          , "filteringPath"        : fake_chain
          , "filteringRequestBody" : fake_chain
          , "matchHeader"          : fake_chain
          , "defaultReplyHeaders"  : fake_chain
          , "log"                  : fake_chain
        };
      }
      
      helpers.timeout = cfg.timeout;
      helpers.nano = nano(cfg.couch);
      helpers.Nano = nano;
      helpers.couch = cfg.couch;
      helpers.pixel = "Qk06AAAAAAAAADYAAAAoAAAAAQAAAP////8BABgAAAAA" + 
                      "AAAAAAATCwAAEwsAAAAAAAAAAAAAWm2CAA==";
      
      helpers.loadFixture = function helpersLoadFixture(filename, json) {
        var contents = fs.readFileSync(
          path.join(__dirname, 'fixtures', filename), 'ascii');
        return json ? JSON.parse(contents): contents;
      };
      
      helpers.nock = function helpersNock(url, fixture) {
        if(process.env.NOCK) {
          var nock    = require('nock')
            , nocks   = helpers.loadFixture(fixture + '.json', true)
            ;
          nocks.forEach(function(n) {
            var path     = n.path
              , method   = n.method   || "get"
              , status   = n.status   || 200
              , response = n.buffer
                         ? new Buffer(n.buffer, 'base64') 
                         : n.response || ""
              , headers  = n.headers  || {}
              , body     = n.base64
                         ? new Buffer(n.base64, 'base64').toString()
                         : n.body
              ;

            if(typeof response === "string" && endsWith(response, '.json')) {
              response = helpers.loadFixture(path.join(fixture, response));
            }
            if(typeof headers === "string" && endsWith(headers, '.json')) {
              headers = helpers.loadFixture(path.join(fixture, headers));
            }
            if(body==="*") {
              nock(url).filteringRequestBody(function(path) {
                return "*";
              })[method](path, "*").reply(status, response, headers);
            } else {
              nock(url)[method](path, body).reply(status, response, headers);
            }
          });
          nock(url).log(console.log);
          return nock(url);
        } else {
          return fake_chain();
        }
      };

Some things to note here. `response` and `body` refer to an http response and the body of an http request (e.g. POST). `base64` means you have a `base64` http request body in the fixture, and buffer means you have a base64 http response that should be converted into a buffer. 

This is not the perfect solution. It's just a solution that fits testing nano, and also the reason why I believe this boilerplate code is a good thing vs. having it in a library: Having this in `nock` would mean making these decisions for you, and I personally think you should have as little decisions made for your users as possible.

Another thing to notice is that you can use `*` to accept any http request body, and that if a header or response ends with `.json` the helper will try to load a fixture with that name. We aren't going to use this, but it can be handy.

We should now be able to run this unmocked (we haven't create the mocks yet remember):

      $ node insert.js 
      
        /insert.js
      
      ✔ 1/1 setup 
      ✔ 3/3 simple 
      ✔ 5/5 functions 
      ✔ 2/2 teardown 
      ✔ 11/11 summary

And the database got destroyed (as it should) by the teardown phase:

      $ curl localhost:5984/doc_insert
      {"error":"not_found","reason":"no_db_file"}

To run the mocked tests we need to add the fixture at `fixtures/doc/insert.json`. This isn't required by `nock`, it's something we defined in our `helpers.js` file:

      [
        { "method"   : "put"
        , "path"     : "/doc_insert"
        , "status"   : 201
        , "response" : "{ \"ok\": true }" 
        }
      , { "method"   : "put"
        , "status"   : 201
        , "path"     : "/doc_insert/foobaz"
        , "body"     : "{\"foo\":\"baz\"}"
        , "response" : "{\"ok\":true,\"id\":\"foobaz\",\"rev\":\"1-611488\"}"
        }
      , { "method"   : "post"
        , "status"   : 201
        , "path"     : "/doc_insert"
        , "body"     : "{\"fn\":\"function () { return true; }\",\"fn2\":\"function () { return true; }\"}"
        , "response" : "{\"ok\":true,\"id\":\"123\",\"rev\":\"1-611488\"}"
        }
      , { "path"     : "/doc_insert/123"
        , "response" : "{\"fn\":\"function () { return true; }\",\"fn2\":\"function () { return true; }\",\"id\":\"123\",\"rev\":\"1-611488\"}"
        }
      , { "method"   : "delete"
        , "path"     : "/doc_insert"
        , "response" : "{ \"ok\": true }" 
        }
      ]

Exit CouchDB and try it out:

      $ NOCK=on node insert.js 
      
        /insert.js
      
      ✔ 1/1 setup 
      ✔ 3/3 simple 
      ✔ 5/5 functions 
      ✔ 2/2 teardown
      ✔ 11/11 summary 

If you try running the tests un-mocked it will fail, since you switched CouchDB off!

## Testing binaries streams

Wow you made it this far! Let's just add a simple streaming test where we insert a pixel in CouchDB. Let's call it `pipe.js`:

      var fs       = require('fs')
        , path     = require('path') 
        , specify  = require('specify')
        , helpers  = require('./helpers')
        , timeout  = helpers.timeout
        , nano     = helpers.nano
        , nock     = helpers.nock
        , pixel    = helpers.pixel
        ;
      
      var mock = nock(helpers.couch, "att/pipe")
        , db   = nano.use("att_pipe")
        ;
      
      specify("setup", timeout, function (assert) {
        nano.db.create("att_pipe", function (err) {
          assert.equal(err, undefined, "Failed to create database");
        });
      });
      
      specify("test", timeout, function (assert) {
        var buffer   = new Buffer(pixel, 'base64')
          , filename = path.join(__dirname, '.temp.bmp')
          , ws       = fs.createWriteStream(filename)
          ;
          ws.on('close', function () {
            assert.equal(fs.readFileSync(filename).toString('base64'), pixel);
            fs.unlinkSync(filename);
          });
          db.attachment.insert("new", "att", buffer, "image/bmp", 
          function (error, bmp) {
            assert.equal(error, undefined, "Should store the pixel");
            db.attachment.get("new", "att", {rev: bmp.rev}).pipe(ws);
          });
      
      });
      
      specify("teardown", timeout, function (assert) {
        nano.db.destroy("att_pipe", function (err) {
          assert.equal(err, undefined, "Failed to destroy database");
          assert.ok(mock.isDone(), "Some mocks didn't run");
        });
      });
      
      specify.run(process.argv.slice(2));

And create the fixture in `fixtures/att/pipe.js`:

      [
        { "method"   : "put"
        , "path"     : "/att_pipe"
        , "status"   : 201
        , "response" : "{ \"ok\": true }" 
        }
      , { "method"   : "put"
        , "path"     : "/att_pipe/new/att"
        , "base64"   : "Qk06AAAAAAAAADYAAAAoAAAAAQAAAP////8BABgAAAAAAAAAAAATCwAAEwsAAAAAAAAAAAAAWm2CAA=="
        , "status"   : 201
        , "response" : "{\"ok\":true,\"id\":\"new\",\"rev\":\"1-3b1f88\"}\n"
        }
      , { "path"     : "/att_pipe/new/att?rev=1-3b1f88"
        , "status"   : 200
        , "buffer"   : "Qk06AAAAAAAAADYAAAAoAAAAAQAAAP////8BABgAAAAAAAAAAAATCwAAEwsAAAAAAAAAAAAAWm2CAA=="
        }
      , { "method"   : "delete"
        , "path"     : "/att_pipe"
        , "status"   : 200
        , "response" : "{ \"ok\": true }" 
        }
      ]

## Updated Nock

Nock now supports NOCK_OFF setting. Check the documentation for details. This means that in your `helpers.js` file `fake_chain` and the if should no longer be necessary

[nano]: http://github.com/dscape/nano
[nock]: http://github.com/flatiron/nock
[specify]: http://github.com/dscape/specify
[LXJS]: http://lxjs.org