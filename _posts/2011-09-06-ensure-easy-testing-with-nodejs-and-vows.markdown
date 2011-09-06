---
layout: post
title: Ensure - Easy testing with node.js and vows
---

# Ensure - Easy testing with node.js and vows

I've been using [vows][1] (and enjoying it) lately. However I never did any behavior driven development, so all that building a JSON document with phrase thing didn't really mean much to me.

All I needed from a test framework was to have my tests described in a way that contributors and maintainers can understand the test as easily as possible. I came up with something like this:

      var ensure = require('ensure')
        , assert = require('assert')
        , tests = exports
        ;
      
      // your test code, callback is sent to testname_ok
      tests.foo = function (callback) {
        callback(true);
      };
      
      // the callback you will eventually fire
      tests.foo_ok = function (t) {
        assert.ok(t);
      };
      
      ensure('foo', tests, module);

This was the idea for [ensure][3] - vows without all the other stuff I didn't care about. [ensure][3] is simply [twelve lines of code][4] that will translate something like this code snippet into something that vows can run. You can install `ensure` with [npm][5] `npm install ensure`. 

You define a test on a property and give it a `name`. Then the convention is the callback will be in `name_ok`. e.g. If you have test `foo` the callback will be `foo_ok`. That's it.

One thing I noticed in all test frameworks I tried was that I was constantly commenting other tests to isolate and run a single test. This is cumbersome and causes some [faulty commits][2]. I had the same problem in `ensure` so decided to tackle it. When calling `ensure` you can pass a comma separated argument which says which tests you want to run (by default `ensure` runs all tests). So if I want to execute tests in `foo.js` named `bar` and `baz` I would:

      # this will only run tests bar and baz
      node foo.js bar,baz

This is super useful, e.g. in `nanocouch` I frequently want to see verbose output of a single test. I can achieve that by simply:

      NANO_ENV=testing node tests/att/get.js att_get

Which would run the `att_get` test with verbose output. If you are curious about how real life tests would look like you can look at [nuvem's][6] (MarkLogic node.js client) tests.

[1]: http://vowsjs.org/
[2]: https://github.com/dscape/nuvem/commit/55adcd418c03d0b25308ca2a3b31a7f7677de27d#L1R11
[3]: https://github.com/dscape/ensure
[4]: https://github.com/dscape/ensure/blob/master/ensure.js
[6]: https://github.com/dscape/nuvem
[5]: http://npmjs.org