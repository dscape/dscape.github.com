---
layout: post
title: Point-free Calculator in Haskell
---

# Point-free Calculator in Haskell

Point-free is a style of programing in Haskell where variables can be omitted. It's pretty useless except for the fact that it makes the code look real cool.

We were given the task of:

* Given a set of point-free simplification rules
* Given a set of point-free expressions

We had to create an algorithm that would analyze those expressions and try to simplify them to a minimum number of expressions using the rules. This turns out to be NPH so we had to develop some heuristics to solve it in real time.

Because this is not hard enough we had to define our own recursion schemes on recursive data types we created specifically for the expressions.

This is analog to saying: Define mathematic expressions in it's own datatype, create recursion methods for it (like reduce, map, etc) and then based on mathematic axioms use a computer to solve those expressions.

Describing the work in one word? Kick Ass! (Ok that's two words)

Github: [pointfreeexprsimplication](https://github.com/dscape/pointfreeexprsimplication)