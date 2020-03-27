---
layout: post
title: Hashing in parallel
---

Working on hashing in parallel.  I'm modifying a Python implementation of HyperLogLog to work with Dask.  So far, the modifications have included serialization and adding the ability to get cardinality for intersections (HyperLogLog proper calculates cardinality for unions only).  I wanted to document my adventure here.

![](https://raw.githubusercontent.com/scottlittle/scottlittle.github.io/master/images/tree-reduce-graph.svg)
