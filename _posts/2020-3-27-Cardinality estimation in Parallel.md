---
layout: post
title: Cardinality estimation in parallel
---
### Introduction
First, what is cardinality? It's counting the unique elements of a field.  In SQL, something like `SELECT DISTINCT field FROM table`.  You can definitely count unique elements this way but the trouble begins when you want to do quick analyses on big data.  If you want to explore relationships between two variables, it may involve `GROUP BY`s and other operations for every pair of variables that you want to explore.  This is especially tedious and expensive if you were to explore every combination of fields.  I'm no expert in big O notation estimates, but this sounds like O(n²) to me.  With probalistic data structures like HyperLogLog and MinHash, we can compute this for every column, so then the cost is only around O(n).

### What I'm doing
I'm modifying a Python implementation of HyperLogLog to work with Dask.  So far, the modifications have included serialization and adding the ability to get cardinality for intersections (HyperLogLog proper calculates cardinality for unions only).  I wanted to document my adventure here.  Here's an example for applying a HyperLogLog (hll) function to every new element.

![](https://raw.githubusercontent.com/scottlittle/scottlittle.github.io/master/images/tree-reduce-graph.svg?sanitize=true)


### Exploring the data
Usually, we would like to count the number of distinct users who did x and also did y.  I looked for a fairly large dataset (a few GB) that was open and interesting and found the Chicago Divvy Bike Share dataset.  Instead of distinct users, this dataset's primary key is `trip_id` (although the raw version is a bit messed up and needs to be deduped).  

|    |   trip_id |   year |   month |   week |   day |   hour | usertype   | gender   | starttime           | stoptime            |   tripduration |   temperature | events   |   from_station_id | from_station_name           |   latitude_start |   longitude_start |   dpcapacity_start |   to_station_id | to_station_name          |   latitude_end |   longitude_end |   dpcapacity_end |
|----|-----------|--------|---------|--------|-------|--------|------------|----------|---------------------|---------------------|----------------|---------------|----------|-------------------|-----------------------------|------------------|-------------------|--------------------|-----------------|--------------------------|----------------|-----------------|------------------|
|  0 |   2355134 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Male     | 2014-06-30 23:57:00 | 2014-07-01 00:07:00 |       10.0667  |            68 | tstorms  |               131 | Lincoln Ave & Belmont Ave   |          41.9394 |          -87.6684 |                 15 |             303 | Broadway & Cornelia Ave  |        41.9455 |        -87.646  |               15 |
|  1 |   2355133 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Male     | 2014-06-30 23:56:00 | 2014-07-01 00:00:00 |        4.38333 |            68 | tstorms  |               282 | Halsted St & Maxwell St     |          41.8646 |          -87.6469 |                 15 |              22 | May St & Taylor St       |        41.8695 |        -87.6555 |               15 |
|  2 |   2355130 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Male     | 2014-06-30 23:33:00 | 2014-06-30 23:35:00 |        2.1     |            68 | tstorms  |               327 | Sheffield Ave & Webster Ave |          41.9217 |          -87.6537 |                 19 |             225 | Halsted St & Dickens Ave |        41.9199 |        -87.6488 |               15 |
|  3 |   2355129 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Female   | 2014-06-30 23:26:00 | 2014-07-01 00:24:00 |       58.0167  |            68 | tstorms  |               134 | Peoria St & Jackson Blvd    |          41.8777 |          -87.6496 |                 19 |             194 | State St & Wacker Dr     |        41.8872 |        -87.6278 |               11 |
|  4 |   2355128 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Female   | 2014-06-30 23:16:00 | 2014-06-30 23:26:00 |       10.6333  |            68 | tstorms  |               320 | Loomis St & Lexington St    |          41.8722 |          -87.6615 |                 15 |             134 | Peoria St & Jackson Blvd |        41.8777 |        -87.6496 |               19 |

Something that I wanted to explore here is something that I knew would overlap, so I picked `gender` and `month` as my pair of variables that I wanted to explore.

![](https://raw.githubusercontent.com/scottlittle/scottlittle.github.io/master/images/venn_gender_month.svg?sanitize=true)

This is the groupby count for every combination:

|    |   month | gender   |   trip_id |
|----|---------|----------|-----------|
|  0 |       1 | Female   |     50003 |
|  1 |       1 | Male     |    221712 |
|  2 |       2 | Female   |     60649 |
|  3 |       2 | Male     |    245326 |
|  4 |       3 | Female   |     92346 |
|  5 |       3 | Male     |    345583 |
|  6 |       4 | Female   |    144702 |
|  7 |       4 | Male     |    476986 |
|  8 |       5 | Female   |    216973 |
|  9 |       5 | Male     |    652611 |
| 10 |       6 | Female   |    319679 |
| 11 |       6 | Male     |    858546 |
| 12 |       7 | Female   |    355774 |
| 13 |       7 | Male     |    923731 |
| 14 |       8 | Female   |    356374 |
| 15 |       8 | Male     |    948329 |
| 16 |       9 | Female   |    314032 |
| 17 |       9 | Male     |    875939 |
| 18 |      10 | Female   |    244807 |
| 19 |      10 | Male     |    752905 |
| 20 |      11 | Female   |    145102 |
| 21 |      11 | Male     |    498454 |
| 22 |      12 | Female   |     78234 |
| 23 |      12 | Male     |    316438 |

We can use this to verify is our probabilistic data structures are doing the right thing.

### Great! But there's a problem...
When I first starting using and modifying the HyperLogLog Python package, I thought this was all that I would need to get cardinality estimates for combinations of variables.  At this point I didn't really care how the package worked, as long as it did.  To skip to the chase, what I found out is that HyperLogLog only estimates unions, not intersections.  My first instinct was to play with the code enough to tease the information out of the functions that I already had.  It turns out that one can do multiple cardinality estimates to do arbitrary combinations of unions and intersections, i.e. `(A U B) ∩ C`, using something called the inclusion-exclusion principle.  This is just a fancy way of saying that we can calculate intersections with only unions.  For example, if we wanted to get the intersection of `A ∩ B`, we'd "include" or add the individual cardinalities for A and B, then "exclude" or subtract the cardinality of the union `A U B`.  Needless to say this could get a little complicated to make a function for, and I did make one, but found out that this method is wildly inaccurate when the original cardinalities of the variables being compared are very different.  That is, the error of the inclusion-exclusion principle might be more than the result itself! Bummer.
