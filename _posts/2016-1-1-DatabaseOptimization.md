---
layout: post
title:  "Database Optimization"
date:   2016-03-14 15:14:00
categories: Technology
tag: cloud computing,java, MySQL, Hbase
comments: true

---

In the phase1, we are required to 

> 1. Implement Extract Transform and Load (ETL) on a large data set (~ 1 TB) and load into MySQL and HBase systems.
> 2. Design schema as well as configure and optimize MySQL and HBase databases to deal with scale and improve throughput.

### ETL Stage

#### JSON

The raw tweet data are JSON files stored in Amazon S3, to have a clearer view about the structure, you can you [Jason Online Parser](http://json.parser.online.fr) to parse the file.

```json
{ 
"created_at":"Thu May 15 09:02:25 +0000 2014",
"id":466866178913083400,
"id_str":"466866178913083392",
"text":"あれ、イベ垢定期流れてない？('A`)",
"user":{
"id":304930190,
"id_str":"304930190",
"name":"Haruka♂"}
}
```

The JSON format could be further parsed by [GSON](https://sites.google.com/site/gson/gson-user-guide), we use ```gson.fromJson(String json, Class<T> classOf T)``` parse Json into special designed class `Tweet`, the function could use reflection to inspect the class variable of `Tweet` and put the value of JSON field into the variable with the same name, if fields are wrapped in JSON, a class with the same structure should be used to contain the wrapped values.

#### Clean
After Extracting from JSON file, we should clean the data based on following rules:

1. **Remove Duplicated tweets.**There may be duplicate tweets (tweets with the same id) in the raw data. Remove all duplicates. You should return one tweet only ONCE.
2. **Remove malformed tweets.**Any tweets that satisfy any one of the following conditions should be regarded as malformed tweets:
	- Both id and id_str are missing or empty
	- Any one of 'created_at', 'text' and 'entities' fields is missing or empty.
Cannot be parsed as a JSON object.
	- You should filter them out of the data set. data. 
	
Moreover, as the project requires, we should also generate *sentimental density* and *censor forbidden words* before loading into databases. This article is focus on ETL and Database optimization, so I will explain more about these methods in another article.

#### Transform
Queries in this question are of the same format, which allows us to further optimized the data based on the query. The request come in the same format:

> GET /q2?userid=[123456]&hashtag=[tag]

and we are asked to response all tweets from required userid with required hashtag. Note that a user may post multiple tweets, a tweet may contain multiple hashtag, we could not represent such a 2 dimension information with only one row in database without hard-merging hashtags. 

So we decided to split the hashtags within one tweet, this may double the total size of dataset, but save much more time because otherwise mySQL has to traverse the whole database using `like` to figure out whose tweet has the hashtag.
For example, we convert tweet:

|id|tweet_id|user_id|density|content|hashtags|
|--|:-------:|:--------:|:-----:|:---------|:--------:|
|1|123454332|1|1.000|I love CC #CC #CMU| CC,CMU|

into

|id|tweet_id|user_id|density|content|hashtag|
|--|--------|--------|-----|---------|--------|
|1|123454332|1|1.000|I love CC #CC #CMU| CC|
|1|123454332|1|1.000|I love CC #CC #CMU| CMU|

So we can easily create index on hashtag to make search faster.

We also eliminate all rows without hashtag, because the queries are assumed to be complete.

After eliminating all duplicate rows, malformed rows, rows without hashtags, and splitting the hashtags into separated rows, we finally got a cleaned dataset with **92861130** rows by using MapReduce.

> Notice: Some tweets many contains duplicate hashtags, which mains you may get duplicate rows after splitting hashtags.

### MySQL Optimization

#### Index

It is not hard to represent the request using this SQL query:

```sql
SELECT * FROM twitter WHERE user_id=123 AND hashtag='tagA';
```

Originally, it requires to traverse the whole database to find out all rows match this condition, this `O(n)` search may take up to 10 minutes in worst case. By creating index on both user_id and hashtag, MySQL will apply Btree search on both column, which could improve the searching time complexity from `O(n)` to `O(logn)`. Further more, we could create multiple index on user_id and hashtag, which concatenate this too key together as a string, and then build index on this combined string, which could further improve the worst case of search performance from `O(2logn)` to `O(logn)`. According to the *MySQL Reference Manual* 

> As an alternative to a composite index, you can introduce a column that is “hashed” based on information from other columns. If this column is short, reasonably unique, and indexed, it might be faster than a “wide” index on many columns.

So here we use **MD5** to encrypt the concatenated string of `user_id` + `hashtag`, MySQl originally support MD5 by providing `MD5()`function. MD5 produces 128-bit hash. So for 2^128+1 number there would be at least one collision, but this is good enough for this context to represent a tweet from certain user with certain hashtag(actually in our database, it is tested unique).

Another advantage of using MD5 is that encrypted string is **32 character fixed-length** string, which make it faster to search. On the other hand, hashtag in tweet could be arbitrary encoding, of arbitrary length.

So instead of hashtag, we recalculate the MD5 of each `user_id`-`hashtag` pair and create index on `MD5` column.

The optimized SQL query is like:

```sql
SELECT * FROM twitter WHERE user_id=123 AND md5=MD5(CONCAT([userid],[hashtag]));
```

To ensure the correctness of response, as the size of database grows, we could also double check user id and hashtag in the above query.


#### Partition

When the database grows larger, it is ridiculous to traverse the whole database for one key, especially when the request fall into the worst case(the bottom of the Btree). To improve the performance, we minimize the hight of index tree by create partitions on the only big table. We use `user_id` as the key, and use Hash partitioning as the key is hard to determine by range or enumeration. 

The hash partitioning simply partition the original table into given number(in our case we set it as 100), so user id `x` goes to partition `x mod 100`. In this way, we elevate the average time complexity from `O(logn)` to `O(log(n/k))`. Even though the improvement is still within the `O(logn)` level, it doubles the search performance in worst case.

#### Caching

Finally, on the frontend, we achieved temporal locality using `PriorityQueue`, which could keep records of certain amount of recent visited data in local RAM. Caching could largely reduce the latency from database connection and improve the total throughput by 50%.

#### Alternatives

Besides our solution, an alternative solution is to build a cluster of backends, which could either use duplication or sharing to split the request load. Note that involving load balancer may also involves addition network latency. Based on our experiments, both the CPU and frontend networking could not be the bottle neck of system overall throughput because the average cpu utilization of m4.large is constantly below 1%, and the frontend throughput without querying database is over 25000 rps with a less than 2 ms latency, accordingly, the latency between frontend and backend connection is the primary bottleneck. So far we only use connection pool to open multi-thread database connections for better performance, it could be more efficient if we could balance the database connection by deploying several backends together.

Reference
---
1. [The importance of tuning your thread pools](http://www.javaadvent.com/2015/12/the-importance-of-tuning-your-thread-pools.html)
2. [Partitioning Types, MySQL Documentation](http://dev.mysql.com/doc/refman/5.7/en/partitioning-types.html)