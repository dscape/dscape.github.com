---
layout: post
title: Mock Testing CouchDB in node.js with Nock and TAP
---

# Mock Testing CouchDB in node.js with Nock and TAP

One of my first node.js libraries was [nano]: A no fuss CouchDB client based on the super pervasive [request]. In foresight that was a good idea, even though there's a ton of clients for CouchDB none of them is as simple as `nano`, and any http client that is not based on `request` is not something I would even consider.

When you are writing a HTTP client you need to test with one (or several) HTTP endpoints. I was lazy about it so I choose to point nano to [iriscouch] and run the tests on real HTTP requests (even found a [bug] in node.js in the way, now fixed in 0.6+). This was a problematic but overall ok approach.

Then some weeks ago I started automating the tests using [travis]. And builds started to fail. To make this work and fix all the shortcomings of the connect to `iriscouch` I needed a HTTP Mocking module.

By the way Travis is super cool. You should test all your node.js libraries with it. All you need to do is go to the site, sign in with github and place a `.travis.yml` file like this one in the root of your lib:

```
language: "node_js"
node_js:
  - 0.4
  - 0.6
```

## Enter Nock

Pedro Teixeira's [nock] allows you do HTTP Mock Testing while preserving the possibility to run the tests against a real http endpoint.

Let's start on this small [tap] test `sudo npm install tap nano nock`:

```
var nano = require('nano')('http://nodejsbug.iriscouch.com') 
var test = require('tap').test;
var db   = nano.use('testing_nock');

test('Insert a Document Into CouchDB', function(t) {
  t.plan(4);
  nano.db.create('testing_nock', function () {
    db.insert({foo: "bar"},
      function ensure_insert_worked_cb(err, doc) {
        t.notOk(err, 'No errors');
        t.ok(doc.ok, 'Contains ok');
        t.ok(doc.rev, 'Rev exists');
        t.ok(doc.id, 'Id exists');
      });
  });
});
```

If we save this in a file `test.js` we can run the tests and see they all work. We can even invoke the script with debugging turned on and inspect the HTTP requests/response flow:

```
$ NANO_ENV=testing node test.js 
{ url: 'http://nodejsbug.iriscouch.com' }
>>
{ method: 'PUT',
  headers: 
   { 'content-type': 'application/json',
     accept: 'application/json' },
  uri: 'http://nodejsbug.iriscouch.com/testing_nock' }
<<
{ err: null,
  body: { ok: true },
  headers: 
   { location: 'http://nodejsbug.iriscouch.com/testing_nock',
     date: 'Thu, 01 Dec 2011 16:42:21 GMT',
     'content-type': 'application/json',
     'cache-control': 'must-revalidate',
     'status-code': 201 } }
>>
{ method: 'POST',
  headers: 
   { 'content-type': 'application/json',
     accept: 'application/json' },
  uri: 'http://nodejsbug.iriscouch.com/testing_nock',
  body: '{"foo":"bar"}' }
<<
{ err: null,
  body: 
   { ok: true,
     id: 'f191a858a66828d8de66b3c974005346',
     rev: '1-4c6114c65e295552ab1019e2b046b10e' },
  headers: 
   { location: 'http://nodejsbug.iriscouch.com/testing_nock/f191a858a66828d8de66b3c974005346',
     date: 'Thu, 01 Dec 2011 16:42:22 GMT',
     'content-type': 'application/json',
     'cache-control': 'must-revalidate',
     'status-code': 201 } }
# Insert a Document Into CouchDB
ok 1 No errors
ok 2 Contains ok
ok 3 Rev exists
ok 4 Id exists

1..4
# tests 4
# pass  4

# ok
```

So `nano` gives you a way to actually see all the HTTP traffic that it creates and receives. This is great but I still need to write code to support these interactions.

With `nock` this is super simple:

```
var nano = require('nano')('http://nodejsbug.iriscouch.com') 
var nock = require('nock'); // we require nock
var test = require('tap').test;
var db   = nano.use('testing_nock');

nock.recorder.rec();

test('Insert a Document Into CouchDB', function(t) {
  t.plan(4);
  nano.db.create('testing_nock', function () {
    db.insert({foo: "bar"},
      function ensure_insert_worked_cb(err, doc) {
        t.notOk(err, 'No errors');
        t.ok(doc.ok, 'Contains ok');
        t.ok(doc.rev, 'Rev exists');
        t.ok(doc.id, 'Id exists');
      });
  });
});
```

Running the tests returns:

```
$ node test.js 

<<<<<<-- cut here -->>>>>>

nock('nodejsbug.iriscouch.com')
  .put('/testing_nock')
  .reply(412, "{\"error\":\"file_exists\",\"reason\":\"The database could not be created, the file already exists.\"}\n", { server: 'CouchDB/1.1.1 (Erlang OTP/R14B04)',
  date: 'Thu, 01 Dec 2011 17:43:30 GMT',
  'content-type': 'application/json',
  'content-length': '95',
  'cache-control': 'must-revalidate' });

<<<<<<-- cut here -->>>>>>

<<<<<<-- cut here -->>>>>>

nock('nodejsbug.iriscouch.com')
  .post('/testing_nock', "{\"foo\":\"bar\"}")
  .reply(201, "{\"ok\":true,\"id\":\"8b787a6a1c2476ef9a2eed069e000ff0\",\"rev\":\"1-4c6114c65e295552ab1019e2b046b10e\"}\n", { server: 'CouchDB/1.1.1 (Erlang OTP/R14B04)',
  location: 'http://nodejsbug.iriscouch.com/testing_nock/8b787a6a1c2476ef9a2eed069e000ff0',
  date: 'Thu, 01 Dec 2011 17:43:31 GMT',
  'content-type': 'application/json',
  'content-length': '95',
  'cache-control': 'must-revalidate' });

<<<<<<-- cut here -->>>>>>

# Insert a Document Into CouchDB
ok 1 No errors
ok 2 Contains ok
ok 3 Rev exists
ok 4 Id exists

1..4
# tests 4
# pass  4

# ok
```

So now all we need to do is add these nock http mocks and we are done:

```
var nano = require('nano')('http://nodejsbug.iriscouch.com') 
var nock = require('nock'); // we require nock
var test = require('tap').test;
var db   = nano.use('testing_nock');

var couch = nock('nodejsbug.iriscouch.com')
  .put('/testing_nock')
  .reply( 412
   , "{ \"error\":\"file_exists\""+
      ", \"reason\":\"The database could not be created, the file" +
      " already exists.\"}\n"
   , { server: 'CouchDB/1.1.1 (Erlang OTP/R14B04)'
   , date: 'Thu, 01 Dec 2011 17:43:30 GMT'
   , 'content-type': 'application/json'
   , 'content-length': '95'
   , 'cache-control': 'must-revalidate' })
  .post('/testing_nock', "{\"foo\":\"bar\"}")
  .reply(201
   , "{ \"ok\":true" +
     ", \"id\":\"8b787a6a1c2476ef9a2eed069e000ff0\"" +
     ", \"rev\":\"1-4c6114c65e295552ab1019e2b046b10e\"}\n"
   , { server: 'CouchDB/1.1.1 (Erlang OTP/R14B04)'
   , location: 'http://nodejsbug.iriscouch.com/testing_nock/'
     + '8b787a6a1c2476ef9a2eed069e000ff0'
   , date: 'Thu, 01 Dec 2011 17:43:31 GMT'
   , 'content-type': 'application/json'
   , 'content-length': '95'
   , 'cache-control': 'must-revalidate' });

test('Insert a Document Into CouchDB', function(t) {
  t.plan(4);
  nano.db.create('testing_nock', function () {
    db.insert({foo: "bar"},
      function ensure_insert_worked_cb(err, doc) {
        t.notOk(err, 'No errors');
        t.ok(doc.ok, 'Contains ok');
        t.ok(doc.rev, 'Rev exists');
        t.ok(doc.id, 'Id exists');
      });
  });
});
```

All working, happy nocking! :)

```
$ node test.js 
# Insert a Document Into CouchDB
ok 1 No errors
ok 2 Contains ok
ok 3 Rev exists
ok 4 Id exists

1..4
# tests 4
# pass  4

# ok
```

[nano]: https://github.com/dscape/nano
[request]: https://github.com/mikeal/request
[iriscouch]: http://iriscouch.com
[bug]: https://github.com/joyent/node/issues/1569
[travis]: http://travis-ci.org/#!/dscape/nano
[nock]: https://github.com/pgte/nock
[tap]: https://github.com/isaacs/node-tap