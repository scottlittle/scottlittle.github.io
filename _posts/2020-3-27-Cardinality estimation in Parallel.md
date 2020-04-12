---
layout: post
title: Cardinality estimation in parallel
---
### Introduction 🧮
First, what is cardinality? It's counting the unique elements of a field.  In SQL, something like `SELECT DISTINCT field FROM table`.  You can definitely count unique elements this way but the trouble begins when you want to do quick analyses on big data.  If you want to explore relationships between two variables, it may involve multiple `GROUP BY`s and other operations for every pair of variables that you want to explore.  This is especially tedious and expensive if you were to explore every combination of fields.  I'm no expert in big O notation estimates, but this sounds like _O(n²)_ to me.  And _O(n²)_ is considering pairs, it could grow to _O(nᵐ)_ if you wanted to compare m columns.  With probalistic data structures like HyperLogLog and MinHash, we can compute this for every column, so then the cost is only around _O(n)_.  See the references listed at the bottom for in-depth discussions on probalistic data structures, the class of data structures that HyperLogLog and MinHash belong to.

### What I'm doing ⚙️
I'm modifying a Python implementation of HyperLogLog to work with Dask.  So far, the modifications have included serialization and adding the ability to get cardinality for intersections (HyperLogLog proper calculates cardinality for unions only).  I wanted to document my adventure here.  

### Exploring the data 📊
Usually, we would like to count the number of distinct users who did x and also did y.  I looked for a fairly large dataset (a few GB) that was open and interesting and found the Chicago Divvy Bike Share dataset.  Instead of distinct users, this dataset's primary key is `trip_id`.  The data is 9495235 rows long and has 9495188 unique `trip_id`s (there are a small number of dupes present).  I broke this into 128 MiB chunks, which makes 16 partitions.  Partitions are what makes dask parallel. And in this case, if you had 16 cores, you could get a 16X speedup.  Plus, dask is out-of-core, meaning that you can process datasets larger than your ram.  Dask recommends just using pandas if your dataset fits in memory.

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
So, in short I modified an existing HyperLogLog Python package to accomplish this.  But first let me explain a bit of motivation (maybe I should have done this first?).  I noticed that in data science and machine learning, it's really helpful to calculate the correlation of multiple variables. In pandas you can do this with `df.corr()` and produce a beautiful grid matrix of all the different relationships of the variables.  But what if your data is mostly categorical, or what if you have big data, or both.  It's not as obvious how to see if two variables are related to each other.  One intuitive way we might do this is by something to think of our data like DNA matching, but instead of DNA, we're matching users.  

![](https://github.com/scottlittle/scottlittle.github.io/blob/master/images/CBP_chemist_reads_a_DNA_profile.jpg?raw=true)

But this breaks down when you consider that users don't necessarily have a particular order that they appear in for the variables to be correlated.  So we are limited to how many users are shared across two variables.  But variables aren't the ultimate unit.  Each variable has several values.  We can call this more fundamental unit a Key-Value Pair, or KVP.  The overlap of KVPs of one variable with the overlap of the KVPs of another variable are really what we want to get at to determine if the variables are related in any way.  If two KVPs have exactly the same number of users, then we can be pretty sure that there must be some sort of relation.  For example, we might find that users of a magazine variable correspond to the same users of a book variable.  Then upon further inpection, the value of the magazine variable is for science fiction and the value of the book variable is astronomy.  So, it makes sense why these two KVPs would have the same users.

### Dask stuff
Let's summarize and expand each step here. I'm just going to put the heart of the code here. [See this for the full Jupyter Notebook.](https://github.com/scottlittle/hyperloglog/blob/master/examples/hll%20intersection%20with%20dask.ipynb)

#### Map reduce functions
1. Make function to apply HLL to each element in a series.
2. Map function to each element in parallel.
3. Reduce to combine HLLs over desired range (unioning these elements).
4. (Optional) Get cardinality from combined function.

Map and reduce code used to generate the dask task graph:
~~~python
############ map steps #############

def hll_count_series(series):
    hll = hyperloglog.HyperLogLog(0.01)  # accept 1% counting error
    series.map( hll.add );
    return hll

res = ddf['trip_id'].map_partitions(hll_count_series, meta=('this_result','f8'))

res = res.to_delayed()

########## reduce steps ############

def hll_reduce(x, y):
    x.update(y)
    return x

# make tree reduction tasks
L = res
while len(L) > 1:
    new_L = []
    for i in range(0, len(L), 2):
        lazy = dask.delayed(hll_reduce)(L[i-1], L[i])  # add neighbors  
        new_L.append(lazy)
    L = new_L                       # swap old list for new

~~~
Here's what this looks like as a dask task graph:

![](https://raw.githubusercontent.com/scottlittle/scottlittle.github.io/master/images/tree-reduce-graph.svg?sanitize=true)


#### Group by functionality
We want to further this process to use with multiple combinations of variables and values, but this time we terminate the functionality with serialization of the HLL object so that we can instantiate it on the fly, that is, when we want to do analysis or quick computation, like in a dashboard.
1. Make function that encapsulates previous "simple function" and then applies it to every variable and value of interest.
2. Terminate with serialization of HLL object.
3. Re-instantiate and run analysis like intersection.

Here's the code-in-brief used for these steps:

~~~python
def hll_serialize(x):
    return x.serialize()

def get_serialized_hll( series ):
    '''returns serialized hll from dask series as dask delayed object'''
    res = series.map_partitions( hll_count_series , meta=('this_result','f8'))
    L = get_tasks(res)
    S = dask.delayed( hll_serialize )(L[0])
    return S
    
def groupby_approx_counts( ddf, x, countby='trip_id' ):
    '''
    input: takes in dask dataframe and item to group by, as well as item to count by
    output: tuple containing: list of unique values in the item to group by, a list of dask delayed objects with the serialized hll object
    '''
    results = []
    uniques = ddf[x].unique().compute().tolist()
    for value in uniques:
        results.append( get_serialized_hll( ddf[ ddf[x]==value ][countby] ) )
        
    serial_objs = dask.compute( *results )
    
    return x, uniques, serial_objs
~~~~

### Parallelized estimate results
The moment you've been waiting for...

<div class="table-wrapper" markdown="block">

|    |   month | gender   |   card estimate |   card actual | percent diff   |
|----|---------|----------|-----------------|---------------|----------------|
|  0 |       1 | Female   |           51188 |         50003 | -2.37%         |
|  1 |       1 | Male     |          222455 |        221712 | -0.34%         |
|  2 |       2 | Female   |           59210 |         60649 | 2.37%          |
|  3 |       2 | Male     |          246677 |        245326 | -0.55%         |
|  4 |       3 | Female   |           93464 |         92346 | -1.21%         |
|  5 |       3 | Male     |          340408 |        345583 | 1.5%           |
|  6 |       4 | Female   |          141403 |        144702 | 2.28%          |
|  7 |       4 | Male     |          476394 |        476986 | 0.12%          |
|  8 |       5 | Female   |          212724 |        216973 | 1.96%          |
|  9 |       5 | Male     |          668272 |        652611 | -2.4%          |
| 10 |       6 | Female   |          321649 |        319679 | -0.62%         |
| 11 |       6 | Male     |          871597 |        858546 | -1.52%         |
| 12 |       7 | Female   |          360552 |        355774 | -1.34%         |
| 13 |       7 | Male     |          930521 |        923731 | -0.74%         |
| 14 |       8 | Female   |          356013 |        356374 | 0.1%           |
| 15 |       8 | Male     |          952413 |        948329 | -0.43%         |
| 16 |       9 | Female   |          313715 |        314032 | 0.1%           |
| 17 |       9 | Male     |          879395 |        875939 | -0.39%         |
| 18 |      10 | Female   |          246669 |        244807 | -0.76%         |
| 19 |      10 | Male     |          759524 |        752905 | -0.88%         |
| 20 |      11 | Female   |          141407 |        145102 | 2.55%          |
| 21 |      11 | Male     |          509001 |        498454 | -2.12%         |
| 22 |      12 | Female   |           76642 |         78234 | 2.03%          |
| 23 |      12 | Male     |          321145 |        316438 | -1.49%         |

</div>

Not too bad, not too bad. Could be worse. `¯\_(ツ)_/¯` <br>
At least it's better than the intersections by the inclusion-exclusion rule:

<div class="table-wrapper" markdown="block">

|    |   month | gender   |   card_int |   card actual | perc diff   |
|----|---------|----------|------------|---------------|-------------|
|  0 |       1 | Female   |      44596 |         50003 | 10.81%      |
|  1 |       1 | Male     |     216517 |        221712 | 2.34%       |
|  2 |       2 | Female   |      77474 |         60649 | -27.74%     |
|  3 |       2 | Male     |     255826 |        245326 | -4.28%      |
|  4 |       3 | Female   |     107591 |         92346 | -16.51%     |
|  5 |       3 | Male     |     359199 |        345583 | -3.94%      |
|  6 |       4 | Female   |     159666 |        144702 | -10.34%     |
|  7 |       4 | Male     |     476835 |        476986 | 0.03%       |
|  8 |       5 | Female   |     233786 |        216973 | -7.75%      |
|  9 |       5 | Male     |     664212 |        652611 | -1.78%      |
| 10 |       6 | Female   |     320897 |        319679 | -0.38%      |
| 11 |       6 | Male     |     866065 |        858546 | -0.88%      |
| 12 |       7 | Female   |     358388 |        355774 | -0.73%      |
| 13 |       7 | Male     |     954115 |        923731 | -3.29%      |
| 14 |       8 | Female   |     371748 |        356374 | -4.31%      |
| 15 |       8 | Male     |     942378 |        948329 | 0.63%       |
| 16 |       9 | Female   |     322049 |        314032 | -2.55%      |
| 17 |       9 | Male     |     862643 |        875939 | 1.52%       |
| 18 |      10 | Female   |     246845 |        244807 | -0.83%      |
| 19 |      10 | Male     |     762295 |        752905 | -1.25%      |
| 20 |      11 | Female   |     155851 |        145102 | -7.41%      |
| 21 |      11 | Male     |     513240 |        498454 | -2.97%      |
| 22 |      12 | Female   |      86019 |         78234 | -9.95%      |
| 23 |      12 | Male     |     314132 |        316438 | 0.73%       |

</div>

### Happy App-y
To demostrate the power of this data structure, I made a ipywidgets app that can explore the cardinalities of different intersections.  The setup should be intuitive here, but behind the scenes, the app is calculating the unions between values of the same fields and the intersections of all of these results.  What you get is a nearly realtime exploratory tool of the data.  To do this with exact counts, you would need to do groupby sums on different cuts of the data, which would take a lot longer.

![](https://github.com/scottlittle/scottlittle.github.io/blob/master/images/Screen%20Shot%202020-04-09%20at%204.06.11%20PM.png?raw=true)

### Summary statistics
Another really powerful advantage of this method of computing cardinality is quickly getting summary statistics about our data.  This was actually my first motativation of implementing everything here.  I wanted to get the "correlation" between two categorical variables on a large dataset, but this took 4 days to compute the the groupby counts for every pair of fields in the dataset.  This method, in opposition, would take minutes.  [Here's a case](https://towardsdatascience.com/the-search-for-categorical-correlation-a1cf7f1888c9) where this statistic was calculated for a very small dataset and was able to find highly correlated coefficients.

Theil's U (uncertainty coefficient):

$$
U(X \mid Y) = \frac{H(X) - H(X \mid Y)}{H(X)}
$$

with

$$
H(X) = -\sum_{X} P_X \log{ \frac{P_X}{P_{X}} }
$$

$$
H(X \mid Y) = \sum_{X,Y} P_{XY} \log{ \frac{P_Y}{P_{X}} }
$$

or with N:

$$
H(X \mid Y) = \frac{1}{N} \sum_{X,Y} N_{XY} \log{ \frac{N_Y}{N_{XY}} }
$$

Using this on our data above, we get <br>
**U( gender | month ) = 0.003743** for gender given a month, and <br>
**U( month | gender ) = 0.000888** for month given a gender. <br>
So, neither of these are highly correlated. But we do get more information about gender from a given month, than month given a gender (it's 4.2 times more).

### Conclusion
In conclusion, I made a thing. Check out the github project, post a comment here, or an issue on the github page.

### Some light reading 📚
#### My stuff
* [Github: My take on HyperLogLog with intersections](https://github.com/scottlittle/hyperloglog)<br>

#### Other people's stuff
* [Github: Original HyperLogLog Python package without intersections](https://github.com/scottlittle/hyperloglog)<br>
* [Really great blog post on HyperLogLog and MinHash by NextRoll (AdRoll)](http://tech.nextroll.com/blog/data/2013/07/10/hll-minhash.html)
* [AdRoll's complimentary paper to their blog post on combining HyperLogLog and MinHash](http://tech.nextroll.com/media/hllminhash.pdf)
* [Recent paper on HyperMinHash and some state of the art techniques](https://arxiv.org/pdf/1710.08436.pdf)
* [Paper on one permutation hashing for Minhash](https://papers.nips.cc/paper/4778-one-permutation-hashing.pdf)
