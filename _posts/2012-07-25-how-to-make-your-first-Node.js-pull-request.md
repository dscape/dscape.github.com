---
layout: post
title: How to make your first Node.js pull request
---

# How to make your first Node.js pull request

If you use open source and github you are probably used to creating issues and having them magically solved for you. You just `npm install` a new version and the issue is gone. Great right?

Well module maintainers can solve the majority of issues for developers (end users) but this takes time.  We need some kind of policy to determine how we react to a new issue on github. This is what I currently do:

* **critical bug**: Fix asap 
* **non-critical bug**: Leave it open and work on it when possible
* **minor-bug**: Close the issue and ask for a PR
* **feature request**: Close the issue and ask for a PR/tests/documentation and blog post announcing the functionality

The first thing you should do is to talk to the module owner (irc, email) and see if this is a issue.

If it is create an issue about what you found. You should include:

* A standalone .js source file that exemplifies the bug (i.e. should run on the node repl)
* Try to minimize dependencies as much as possible (i.e. use only node core and the module you are creating the issue for)

Now wait for the module owner to respond to you. He will either:

* Fix it, if he can/is willing to
* Classify as invalid (wontfix) and explain why
* Not reply
* Ask for the standalone isolated test talked about above if you didn't produce it to start with
* Ask for a PR

If you are asked for a pull request, you need to fork the repo into your own user. 

Imagining your repo is `dscape/foobar` and the bug is `issue63` this is what you could do:

      git clone git@github.com:dscape/foobar.git
      cd foobar
      git branch issue63
      git checkout issue63

Now you need to makes the changes that fix your bug. This will likely involve some investigation. After you do so run:

      git diff

This should have a small output, and should be limited to the minimum amount of lines of code necessary to fix the issue. If you have fixed other stuff that is unrelated, please undo, create a new issue, and do that in a new branch.

If it all looks ok for you, you can do a `git status` and check that you didn't introduce any files by mistake and finally `git add .` (or simple the files you changed one by one). Ok, time to commit:

      git commit -m "[fix minor] fixes #63
      
      * Solves foo
      * Modifies Behavior in bar
      "

If you are curious about why I wrote `fixes #63` in the commit message check the blog post about [issues 2.0 on github](https://github.com/blog/831-issues-2-0-the-next-generation).

Now you need to add tests. Read the read me section of the project and see if there's instructions on adding and running tests. 

Being very generic you should start of by running the existing test suite and make sure you didn't break anything:

      npm install
      npm test

You should also inspect the `package.json` file and make sure there's nothing else you should be running like un-mocked tests for instance.

Now add your tests, and repeat the `git diff`, `git status`, `git commit` workflow.

If the tests all run you are ready test wise. Add the docs, same workflow again.

Now:

      git push

Now go to github and open a pull request while selecting the appropriate branch.

## The extra mile

If you want to develop your own module you might be interested on what gets done then from the module maintainer perspective.

When you see a pull request you navigate and review the code, tests, documentation. 
You might go back and forward asking while some code is done in a certain way (e.g. code review)

If all is right you are going to do something like (in your module directory):

      git remote add dscape git@github.com:dscape/foobar.git
      git pull
      git pull dscape issue63

Now you need to run tests. When maintaining `nano` I normally do:

      npm test
      npm run nock_off

If there's bugs (which happens about 80% of the time cause people forget to run tests) the module maintained will fix them. He will add fixtures, adds mocks (whatever it takes) and commit them. After all is working, docs are fixed, etc the module maintainer will add you to the contributors list and go on to publish a new version. Let's say this is version `0.0.2`

      git push

Now we wait for travis tests to run and make sure [travisci](http://travis-ci.org/) tests pass in multiple node versions etc. If they don't, the module maintainer will go on fixing bugs again. 

When the tests finally pass:

      git tag 0.0.2
      git push --tags
      npm publish

Now the module maintainer will go back to the issue, close it, and warn the user the fix is available in version `0.0.2`.
