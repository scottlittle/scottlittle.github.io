---
layout: post
title: Cardinality estimation in parallel
---
### Introduction 🧮
First, what is cardinality? It's counting the unique elements of a field.  In SQL, something like `SELECT DISTINCT field FROM table`.  You can definitely count unique elements this way but the trouble begins when you want to do quick analyses on big data.  If you want to explore relationships between two variables, it may involve multiple `GROUP BY`s and other operations for every pair of variables that you want to explore.  This is especially tedious and expensive if you were to explore every combination of fields.  I'm no expert in big O notation estimates, but this sounds like _O(n²)_ to me.  And _O(n²)_ is considering pairs, it could grow to _O(nᵐ)_ if you wanted to compare m columns.  With probalistic data structures like HyperLogLog and MinHash, we can compute this for every column, so then the cost is only around _O(n)_.  See the references listed at the bottom for in-depth discussions on probalistic data structures, the class of data structures that HyperLogLog and MinHash belong to.

### What I'm doing ⚙️
I'm modifying a Python implementation of HyperLogLog to work with Dask.  So far, the modifications have included serialization and adding the ability to get cardinality for intersections (HyperLogLog proper calculates cardinality for unions only).  I wanted to document my adventure here.  Here's an example for applying a HyperLogLog (hll) function to every new element. I'll come back to this graph in a bit.

![](https://raw.githubusercontent.com/scottlittle/scottlittle.github.io/master/images/tree-reduce-graph.svg?sanitize=true)

Code used to generate the dask task graph above:
~~~python
def get_tasks( res ):
    L = res.to_delayed()
    while len(L) > 1: # tree reduction
        new_L = []
        for i in range(0, len(L), 2):
            lazy = dask.delayed( hll_reduce )(L[i-1], L[i])  # add neighbors  
            new_L.append( lazy )
        L = new_L                       # swap old list for new
    return L
~~~

### Exploring the data 📊
Usually, we would like to count the number of distinct users who did x and also did y.  I looked for a fairly large dataset (a few GB) that was open and interesting and found the Chicago Divvy Bike Share dataset.  Instead of distinct users, this dataset's primary key is `trip_id` (although the raw version is a bit messed up and needs to be deduped).  

<div class="table-wrapper" markdown="block">
  
|    |   trip_id |   year |   month |   week |   day |   hour | usertype   | gender   | starttime           | stoptime            |   tripduration |   temperature | events   |   from_station_id | from_station_name           |   latitude_start |   longitude_start |   dpcapacity_start |   to_station_id | to_station_name          |   latitude_end |   longitude_end |   dpcapacity_end |
|----|-----------|--------|---------|--------|-------|--------|------------|----------|---------------------|---------------------|----------------|---------------|----------|-------------------|-----------------------------|------------------|-------------------|--------------------|-----------------|--------------------------|----------------|-----------------|------------------|
|  0 |   2355134 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Male     | 2014-06-30 23:57:00 | 2014-07-01 00:07:00 |       10.0667  |            68 | tstorms  |               131 | Lincoln Ave & Belmont Ave   |          41.9394 |          -87.6684 |                 15 |             303 | Broadway & Cornelia Ave  |        41.9455 |        -87.646  |               15 |
|  1 |   2355133 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Male     | 2014-06-30 23:56:00 | 2014-07-01 00:00:00 |        4.38333 |            68 | tstorms  |               282 | Halsted St & Maxwell St     |          41.8646 |          -87.6469 |                 15 |              22 | May St & Taylor St       |        41.8695 |        -87.6555 |               15 |
|  2 |   2355130 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Male     | 2014-06-30 23:33:00 | 2014-06-30 23:35:00 |        2.1     |            68 | tstorms  |               327 | Sheffield Ave & Webster Ave |          41.9217 |          -87.6537 |                 19 |             225 | Halsted St & Dickens Ave |        41.9199 |        -87.6488 |               15 |
|  3 |   2355129 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Female   | 2014-06-30 23:26:00 | 2014-07-01 00:24:00 |       58.0167  |            68 | tstorms  |               134 | Peoria St & Jackson Blvd    |          41.8777 |          -87.6496 |                 19 |             194 | State St & Wacker Dr     |        41.8872 |        -87.6278 |               11 |
|  4 |   2355128 |   2014 |       6 |     27 |     0 |     23 | Subscriber | Female   | 2014-06-30 23:16:00 | 2014-06-30 23:26:00 |       10.6333  |            68 | tstorms  |               320 | Loomis St & Lexington St    |          41.8722 |          -87.6615 |                 15 |             134 | Peoria St & Jackson Blvd |        41.8777 |        -87.6496 |               19 |

</div>

Something that I wanted to explore here is something that I knew would overlap, so I picked `gender` and `month` as my pair of variables that I wanted to explore.

![](https://raw.githubusercontent.com/scottlittle/scottlittle.github.io/master/images/venn_gender_month.svg?sanitize=true)

This is the groupby count for `trip_id` for every combination of the values in month and gender:

|    |   month | gender   |     count |
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

### Great! But there's a problem...👹
When I first started using and modifying the HyperLogLog Python package, I thought this was all that I would need to get cardinality estimates for combinations of variables.  At this point I didn't really care how the package worked, as long as it did.  To skip to the chase, what I found out is that HyperLogLog only estimates unions, not intersections.  My first instinct was to play with the code enough to tease the information out of the functions that I already had.  It turns out that one can perform multiple cardinality estimates to calculate arbitrary combinations of unions and intersections, i.e. `(A ∪ B) ∩ C`, using something called the inclusion-exclusion principle.  This is just a fancy way of saying that we can calculate intersections with only unions.  For example, if we wanted to get the intersection of `A ∩ B`, we'd "include" or add the individual cardinalities for A and B, then "exclude" or subtract the cardinality of the union `A ∪ B`.  Needless to say this could get a little complicated to make a function for, and I did make one, but I found out that this method is wildly inaccurate when the original cardinalities of the variables being compared are very different.  That is, the error of the inclusion-exclusion principle might be more than the result itself! Bummer. 🙁 

### Not to fear, MinHash is here! 😁
So, the way to solve this was to tack on another probabilistic data structure to solve for intersections.  I can get into the details of HyperLogLog later (or not, and just refer you to some papers or blogs), but we have to hash every unique element of the field that we are considering the cardinality for (in the above example, it's `trip_id`).  FYI- without saying too much about the details- HyperLogLog is a fancy way of counting zeros to estimate cardinality.  HyperLogLog uses "registers" to store the result of adding a new element.  The main engine is **elements that were added previously, will not be added again** because they will be hashed in exactly the same way and added to the same registers.  There are more elements than number of registers, so there will be collisions (mistakes), that's why HyperLogLog is an estimate and not an exact amount.  Unions of elements are as easy as taking the max of there registers.  We can serialize the result as a small string, saving space on the millions or billions of elements that we would otherwise have to keep track of.  At the record level, it doesn't make any sense to have hll data structures, but the magic comes in when you combine them and for, say, all of the unique values for each of your variables.

So where does MinHash come into play? It turns out that our hll registers aren't good enough to easily create cardinality estimates for intersections, as mentioned above.  We need to add a the MinHash, or k number of `MinHash`es to each element.  I did this by creating a seperate number of registers, calculating an element's MinHash using a single hash function, then adding that value to the registers if it was low enough.  This is all just an elaborate way of saying that I "sampled" the variable of interest (in this case `trip_id`).  This sampling would look like this:

![](https://raw.githubusercontent.com/scottlittle/scottlittle.github.io/master/images/venn_gender_month_dots.svg?sanitize=true)

The result of this sampling is that we can find the proportion of the intersection to the union, which is called the Jaccard Index.  Mathematically, 
```J = ( A ∩ B ) / ( A ∪ B )```, where J is the Jaccard Index.  So, we can use this to get our intersection by 
```A ∩ B = J ( A ∪ B )```.  So we can get the intersection by calculating the Jaccard Index using MinHash and multiplying it by the union, calculted by the HyperLogLog, our old friend.

### On to how I did it, or, wait? ⏳
So, in short I modified an existing HyperLogLog Python package to accomplish this.  But first let me explain a bit of motivation (maybe I should have done this first?).  I noticed that in data science and machine learning, it's really helpful to calculate the correlation of multiple variables. In pandas you can do this with `df.corr()` and produce a beautiful grid matrix of all the different relationships of the variables.  But what if your data is mostly categorical, or what if you have big data, or both.  It's not as obvious how to see if two variables are related to each other.  One intuitive way we might do this is by something to think of our data like dna sequencing but instead of matching RNA, we're matching users.  But this breaks down when you consider that users don't necessarily have a particular order that they appear in for the variables to be correlated.  So we are limited to how many users are shared across two variables.  But variables aren't the ultimate unit.  Each variable has several values.  We can call this more fundamental unit a Key-Value Pair, or KVP.  The overlap of KVPs of one variable with the overlap of the KVPs of another variable are really what we want to get at to determine if the variables are related in any way.  If two KVPs have exactly the same number of users, then we can be pretty sure that there must be some sort of relation.  For example, we might find that users of a magazine variable correspond to the same users of a book variable.  Then upon further inpection, the value of the magazine variable is for science fiction and the value of the book variable is astronomy.  So, it makes sense why these two KVPs would have the same users.

### Dask stuff
To do...but let's summarize:

#### Simple function
1. Make function to apply HLL to each element in a series.
2. Map function to each element in parallel.
3. Reduce to combine HLLs over desired range (unioning these elements).
4. Get cardinality from combined function.

#### Group by functionality
We want to further this process to use with multiple combinations of variables and values, but this time we terminate the functionality with serialization of the HLL object so that we can instantiate it on the fly, that is, when we want to do analysis or quick computation, like in a dashboard.
1. Make function that encapsulates previous "simple function" and then applies it to every variable and value of interest.
2. Terminate with serialization of HLL object.
3. Re-instantiate and run analysis like intersection.

#### Parallelized estimate results
The moment you've been waiting for...

### Some light reading 📚
#### My stuff
* [Github: My take on HyperLogLog with intersections](https://github.com/scottlittle/hyperloglog)<br>

#### Other people's stuff
* [Github: Original HyperLogLog Python package without intersections](https://github.com/scottlittle/hyperloglog)<br>
* [Really great blog post on HyperLogLog and MinHash by NextRoll (AdRoll)](http://tech.nextroll.com/blog/data/2013/07/10/hll-minhash.html)
* [AdRoll's complimentary paper to their blog post on combining HyperLogLog and MinHash](http://tech.nextroll.com/media/hllminhash.pdf)
* [Recent paper on HyperMinHash and some state of the art techniques](https://arxiv.org/pdf/1710.08436.pdf)
* [Paper on one permutation hashing for Minhash](https://papers.nips.cc/paper/4778-one-permutation-hashing.pdf)
