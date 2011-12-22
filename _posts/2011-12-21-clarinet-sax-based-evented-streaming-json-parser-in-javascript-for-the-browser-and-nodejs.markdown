---
layout: post
title: Clarinet - SAX based evented streaming JSON parser in JavaScript for the browser and nodejs
---

# Clarinet - SAX based evented streaming JSON parser in JavaScript for the browser and nodejs

I'm super happy to announce `clarinet`. It's currently running ~110 [tests] both in the `browser` and in `node.js` which include some of the most obtuse edge cases I could find in other JSON parser tests, and can currently parse all `npm` registry without blinking.

`clarinet` is not a replacement for `JSON.parse`. If you can `JSON.parse`, you should. It's super fast, comes bundled in V8, and that's all. Move along.

My motivation for `clarinet` was to stream large (or small) chunks of JSON data and be able to created indexes on the fly.

`clarinet` is more like `SAX`. It's a streaming parser and when you feed it json chunks it will emit events. 

Or in code:

      var chunks = ['{"foo":', ' "bar', '"}'];

You can't parse that. Even if you control the source that is emitting those chunks there's plenty of situations where you can't just emit a 10mb file in one chunk. Also if your JSON file is larger than the memory you have available on your computer, you need `clarinet`.

This is how you would implement SubStack's [npmtop], a tool that returns an list on npm modules authors ordered by the number of modules they publish, with `clarinet`:

      var fs             = require('fs')
        , clarinet       = require('clarinet')
        , parse_stream   = clarinet.createStream()
        , author         = false // was the previous key an author?
        , authors        = {}    // authors found so far
        ;
      
      // open object is emitted when we find '{'
      // the name is the first key of the json object
      // subsequent ones will emit key
      parse_stream.on('openobject', function(name) {
        if(name==='author') author=true;
      });
      
      // a key was found
      parse_stream.on('key', function(name) {
        if(name==='author') author=true;
      });
      
      // we got all of npm, lets aggregates results 
      // and sort them by repo count.
      parse_stream.on('end', function () {
        var sorted = []
          , i
          ;
        for (var a in authors)
          sorted.push([a, authors[a]]);
        sorted.sort(function(a, b) { return a[1] - b[1]; });
        i = sorted.length-1;
        while(i!==-1) {
          console.log(sorted.length-i, sorted[i]);
          i--;
        }
      });
      
      // value is emitted when we find a json value, just like in the
      // specification in json.org: strings, true, false, null, and number.
      //
      // you can find out the value type by running a typeof
      //
      // this could be faster if we emitted different events for each value.
      // e.g. .on('string'), .on('true'), etc..
      //
      // would be faster cause clarinet wouldn't have to parse it for you
      // but this api choice seemed easier for the developer 
      // that needs to have less events
      // to attend to
      parse_stream.on('value', function(value) {
        if(author) { 
          // get the current count for this author
          var current_count = authors[value];
          // if it exists increment it
          if (current_count) authors[value] +=1;
          // else it's the first one
          else authors[value] = 1;
          // this is not an author key
          author=false; 
        }
      });
      
      // create a read stream and pipe it to clarinet
      fs.createReadStream(__dirname + '/npm.json').pipe(parse_stream);


Feel free to browse the [docs] and [samples] for more goodies. Feedback is great, pull requests are even better.

## Performance

### tl;dr

* Clarinet is good and fast enough at doing what it is designed to do
* `JSON.parse` is way faster - you should use it if you can

Since having a streaming parser requires consistently understanding performance implications, I've done a preliminary [study] on how well `clarinet` performs. [Source code is open][bench] so you are welcome to replicate.

Because none of the other parsers tests was able to do streaming JSON parsing I had to create a test that uses `fs.readFileSync` so all of the parsers could be tested. This sucks, I really wanted to test async parsers but none existed. I tried [yajl]  (a c++ module that looks a lot like clarinet) but it's current  version does not build in node `0.6`. `jsonparse` should also be able run asynchronously but since it's not documented properly and was made in a previous version of node I was unable to make it work . If you are looking for differences between `jsonparse` and `clarinet`:

* `jsonparse` emits less events and is not sax like. this makes it faster on smaller json files and much slower on large json files. plus it makes it unsuited for the certain lower level application like the one i had in mind, building indexes. both `clarinet` and `jsonparse` would benefit from abstractions on top that allow people to do json streaming for common case scenarios, even if at this point it's not obvious what that means.

If you want your parser to be included, or refute any of my claims please send me an email and I'll fix this article provided you give me source code and results to go along with it.

I created an [async] version of the tests but only clarinet is included there for obvious reasons. In the process I also created a [profiling] page that can help you get profiling information about `clarinet` using Google Chrome Developer Tools.

### In detail

In the test we've compared `clarinet`, `JSON.parse` (referred as V8 in the tables and figures) and @[creationix] [jsonparse]. To avoid sample bias I've tested all modules against four different JSON documents:

* **[npm]**: The full npm registry (11M). Parsed once.
* **[twitter]**: A bunch of node.js tweets (13M). Parsed once.
* **[creationix][creationixs]**: One of `jsonparse` test cases (4.5K). Parsed 2000 times to have more significant times.
* **[wikipedia]**: The JSON sample from wikipedia.org/JSON (467B). Parsed 100000 times to have more significant times.

In order to test whether `clarinet`, `JSON.parse`, and the `jsonparse` modules differed in terms of execution time, I conducted analyses of variance (ANOVAs). To obtain the estimate data, I ran [scripts][bench] that created 10 runs for each JSON document, resulting in 40 measurements per parser.

Then, the three modules were at first compared between each other, regardless of the documents that generated their values (i.e. the execution times). This ANOVA showed that the differences in the execution times obtained with `clarinet`, `JSON.parse`, and `jsonparse`, were statistically significant (F(2,117) = 40.28, p = .000).

Post-Hoc Tests with Scheffe correction revealed that the execution times of `JSON.parse` module were statistically different from both `clarinet` and `jsonparse`, but these two did not differ from one another (see Table 1 and Fig. 1). Specifically, `JSON.parse` module demonstrated smaller execution times than both `clarinet` and `jsonparse`.

![Table 1](http://writings.nunojob.com/images/clarinet-table1.png "Table 1")
![Figure 1](http://writings.nunojob.com/images/clarinet-figure1.png "Figure 1 Overall")

Next, given that two of the documents were "big" (i.e. > 1MB), and two of them were "small" (i.e. < 1MB) and the expectation that the size of the documents would play an important role in the performance of the modules, I computed an ANOVA to compare the execution times of the modules for the "big" documents, and another ANOVA to compare them for the "small" documents.

The differences between the execution times of `clarinet`, `JSON.parse`, and `jsonparse` were statistically significant for the "big" documents (F(2,57) = 279.96, p = .000).

Post-Hoc Tests with Scheffe correction revealed that the execution times of the modules were statistically different between the three of them (see Table 2 and Fig. 2).

![Table 2](http://writings.nunojob.com/images/clarinet-table2.png "Table 2")
![Figure 2](http://writings.nunojob.com/images/clarinet-figure2.png "Figure 2 Big")

The differences between the execution times of `clarinet`, `JSON.parse`, and `jsonparse` were also statistically significant for the "small" documents (F(2,57) = 36.95, p = .000).

Post-Hoc Tests with Scheffe correction revealed that the execution times of the modules were once again statistically different between the three of them (see Table 3 and Fig. 3).

![Table 3](http://writings.nunojob.com/images/clarinet-table3.png "Table 3")
![Figure 3](http://writings.nunojob.com/images/clarinet-figure3.png "Figure 3 Small")

In conclusion, the execution times of the three modules under analysis were different in all the conditions tested (i.e. regardless of the document size, for big documents only, and for small documents only), but this difference was greater when considering the estimates made for dealing with big documents only (which can be seen by the F ratios), where the `JSON.parse` demonstrated clearly smaller execution times.

[npmtop]: https://github.com/substack/npmtop
[docs]: https://github.com/dscape/clarinet
[samples]: https://github.com/dscape/clarinet/tree/master/samples
[tests]: https://github.com/dscape/clarinet/blob/master/test/clarinet.js
[study]: https://github.com/dscape/clarinet/tree/master/bench/results/dscape-study
[bench]: https://github.com/dscape/clarinet/tree/master/bench
[async]: https://github.com/dscape/clarinet/blob/master/bench/async.js
[creationix]: https://github.com/creationix
[jsonparse]: https://github.com/creationix/jsonparse
[twitter]: https://github.com/dscape/clarinet/blob/master/samples/twitter.json
[npm]: https://github.com/dscape/clarinet/blob/master/samples/npm.json
[wikipedia]: https://github.com/dscape/clarinet/blob/master/samples/wikipedia.json
[creationixs]: https://github.com/dscape/clarinet/blob/master/samples/creationix.json
[profiling]: https://github.com/dscape/clarinet/blob/master/test/bench.html
[yajl]: https://github.com/lloyd/node-yajl