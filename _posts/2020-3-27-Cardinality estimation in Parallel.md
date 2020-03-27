---
layout: post
title: Cardinality estimation in parallel
---
### Introduction
First, what is cardinality? It's counting the unique elements of a field.  In SQL, something like `SELECT DISTINCT field FROM table`.  You can definitely count unique elements this way but the trouble begins when you want to do quick analyses on big data.  If you want to explore relationships between two variables, it may involve `GROUP BY`s and other operations for every pair of variables that you want to explore.  This is especially tedious and expensive if you were to explore every combination of fields.  I'm no expert in big O notation estimates, but this sounds like O(n²) to me.  With probalistic data structures like HyperLogLog and MinHash, we can compute this for every column, so then the cost is only around O(n).

### What I'm doing
I'm modifying a Python implementation of HyperLogLog to work with Dask.  So far, the modifications have included serialization and adding the ability to get cardinality for intersections (HyperLogLog proper calculates cardinality for unions only).  I wanted to document my adventure here.  Here's an example for applying a HyperLogLog (hll) function to every new element.

![](https://raw.githubusercontent.com/scottlittle/scottlittle.github.io/master/images/tree-reduce-graph.svg?sanitize=true)
