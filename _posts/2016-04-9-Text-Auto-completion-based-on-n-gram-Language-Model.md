---
layout: post
title:  "Text Auto-completion based on n-gram Language Model"
date:   2016-04-9 19:56:00
categories: Technology
tag: cloud computing,language model, MapReduce, Hbase
comments: true

---
The individual project of 15619 this week is of great fun, it is about building an Input Text Predictor based on language model.

Generally, the text predictor is based on a huge n-gram semantic model, which could be achieved from wikipedia corpus, and we are required using MapReduce to calculate the frequency of each n-gram.


The task could be broken down into the following steps:

1. **Preprocess source text**
2. **Generate n-grams**
3. **Generate text prediction candidates**
4. **Store the result**
5. **Frontend testing**

### Preprocess source text

As the write up requires, we need to take care of the following cases during data preprocessing:

> 1. Lowercase 
> 2. Remove URLs
> 3. Remove `<ref>` tags
> 4. Remove all non-char symbols, aka remain only `[a-z]`
> 5. **One exception**: Retain single `'` in the middle of a word, which means:
> 		* `I'm` should be taken as a word;
> 		*  but `'before`, `after'` should be trimmed to `before`, `after`;
> 		* 	moreover, multiple `'` like `'''` in `''manybefore` `many''mid` or 
> 		`manyafter'''''` should be taken as delimiters,so `many''mid` turns to be 
> 		`many` and `mid`

Apparently, all the cleaning tasks could be done with regular expression, at beginning, I use the following regex:

* ref tags: `</?ref[^>]*>`
* urls: `(https?|ftp):\\/\\/[^ ]*`
* Replace `[^a-z']|'{2,}` by `space`, than by `[\\s+]` into String array
* Foreach word, remove `^'+|'+$`
* finally examine if word's length is greater than 0

This process is painful, and it forced me to learn a new skill named **lookarount** to solve this problem gracefully[^1].[^2].

#### Lookaround

To be completed...

### Generate n-gram

In the fields of computational linguistics and probability, an n-gram is a contiguous sequence of n items from a given sequence of text or speech. The items can be phonemes, syllables, letters, words or base pairs according to the application.[^3]

N-gram could be a representative pattern of human nature language, by collecting the frequency of each phase(a specific contiguous sequence of words), we can easily tell which one is more popular. In another word, n-gram ensure that a phase is make sense, and the n-gram with higher possibility ensure that the phase is common and widely used. These two criteria are accordance to what we expect to the auto-completed words: **it should make sense, and should be the word first come to your mind.**

The implement of n-gram is fairly easy:

* In the **mapper phase**, take a cleaned sentence as input value(a auto generate LongWritable as key), using a DFS recursion with n as a constraint to generate all possible n-gram combinations of the sentence as output key, and assign '1' as value(refer [WordCountv1 example](https://hadoop.apache.org/docs/r1.2.1/mapred_tutorial.html)).
* In the **reducer phase**, sum the count number of a key up.

### Store & Extract
In the previous stage, we take raw data from HDFS and write MR output back to HDFS, here we want to see the top 100 n-grams in the result.

The first way to do that is by: **concatenate all MR output files -> sort -> retrieve the first 100 lines** right in the base. But I failed because the sorting requires a lot of storage for intermediate results, and it is rather slow!

A improved solution is using **Hive**, it can take files on HDFS as a database and perform SQL-like query on these data using MapReduce jobs. First, we create table using:
```sql	
CREATE EXTERNAL TABLE ngram (key STRING, value INT) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t' LOCATION 'hdfs://52.91.198.12:9000/output';
```
	
Then, select the top 100 n-gram phases using:
```sql
	select * from ngram order by `value` desc limit 100;
```

### Generate text prediction candidates

A simple example of text prediction:

When you enter phase `Today`, the system should provide several candidate words after the given phase based on probability. For example in our database, we have three 2-gram start with `Today`,namely

phase|probability
----|:-----------:
today is|0.7
today are|0.2
today would|0.1

so the candidate of `Today` should be `is`,`are`and`you`.

Based on this, I use the following Hbase schema to store the data for prediction candidate:


ROWKEY| COLUMN|VALUE
--|--|--
phase| candidate|Probability

In this way, I can use simple a `Get`method to retrieve all candidates and their probabilities of a given phase.

To achieve this, during the MapReduce job, instead of using writing output directly to the HDFS, we should use HBase as data sink. Some modifications should be done[^4].[^5]:

1. The Reduce class should extend `TableReducer` class;
2. The output key value type should be `ImmutableBytesWritable`, which could be written directly into hbase, specifically you should write a `org.apache.hadoop.hbase.client.Put` instance as the output value.
3. In the Driver, the `config` should be a `HBaseConfiguration`;
4. Set the `OutputKeyClass` as `ImmutableBytesWritable`;
5. Set the `OutputFormatClass` as `TableOutputFormat.class`;
6. Configurate the reducer class, MR job, and target data sink table using
		
		TableMapReduceUtil.initTableReducerJob(TABLE, LMReduce.class, job);
	this will guide the MapReduce job to execute assigned Reducer class and put the data into suggested table.
	
After loading the data, the data regarding to phase `this` in Hbase should be like:

> ROWKEY:'this' COLUMNFAMILY:'is' TIMESTAMP:XXXXXX VALUE:0.7  
> ROWKEY:'this' COLUMNFAMILY:'are' TIMESTAMP:XXXXXX VALUE:0.2  
> ROWKEY:'this' COLUMNFAMILY:'would' TIMESTAMP:XXXXXX VALUE:0.1

### Frontend Testing

Then we build a simple HTML with PHP, JQuery and MySQL to achieve the text auto-prediction function.[^6]

The HTML part is really simple. Simply paste this into your file index.html:

	<!DOCTYPE html>
	<html>
	 <head>
  	 <title>Auto-complete tutorial</title>
  	 <script src="//ajax.googleapis.com/ajax/libs/jquery/2.1.1/jquery.min.js"></script>
  	 <script src="auto-complete.js"></script>
	 </head>
	 <body>
  	 	<input type="text" value="" placeholder="Search" id="keyword">
	 </body>
	</html>
	
JavaScript Part:

	var MIN_LENGTH = 3;
	$( document ).ready(function() {
	 $("#keyword").keyup(function() {
	 var keyword = $("#keyword").val();
	 if (keyword.length >= MIN_LENGTH) {
	 	$.get( "auto-complete.php", { keyword: keyword } )
		.done(function( data ) {
		 	console.log(data);
		 });
		}
	 });
	});
The get method take a php page as parameter and expect an array as return value, the javascript will simply display the returned array as a list view below the input textbox.
So PHP part will be like:

```php
	<?php
		// get the search term
		$searchTerm = strtolower(trim( $_REQUEST['term'] ));
		// Prepare the query
		$url = "http://ec2-54-84-198-195.compute-1.amazonaws.com:8080/".		$tableName."/".urlencode($searchTerm)."?v=1";
		//Retrive the Hbase Response 
		$response = \Httpful\Request::get($url)->addHeader('Accept','application/json')->send();
		//... parse the response into a return array:
		...
		...		
	    $data[] = array(
			'label' => $first[$key],
			'probability' => $second[$key]
		// return JSON encoded array to the JQuery autocomplete function
		echo json_encode($data);
		flush();>
	?>
```
Notice that here the web service use Hbase REST server to get data form hbase, as default, the hbase server is running on port 8080, you should first start HBase rest server by

	./home/hadoop/hbase/bin/
		
and then, restart Apache

	hbase-daemon.sh start rest

Then you are supposed to get response from that REST API by sending GET request with header Key:`Accept`, Value:`application/json`, and format like:
	
	http://dns:8080/tablename/searchterm?v=1

Notice that the response is in JSON format and need to be parsed by`json_decode()`, and the data is encoded by base64, so you should also decode using `base64_decode()`.

### Appendix: Install Tmux

Mapreduce task is long and fragile, I have suffered from broken pipe for more then three times due to different reasons, however, AWS EMR instance with hadoop does not preinstalled tmux, which bother me a lot, so I strongly recommend to install tmux to get avoid of broken pipeline. 

	sudo yum install tmux
	
attach
	
	tmux a

detach 
	
	ctrl + b, d

Split horizontally

	ctrl + b, "
	
Split Vertically

	ctrl + b + %
	

#### Reference
[^1]: [Basis of regular expression -- Lookaround(Chinese)](http://blog.csdn.net/lxcnn/article/details/4304754)
[^2]:[Lookaround](http://blog.sciencenet.cn/blog-656335-814400.htm)
[^3]:Broder, Andrei Z.; Glassman, Steven C.; Manasse, Mark S.; Zweig, Geoffrey (1997). "Syntactic clustering of the web". Computer Networks and ISDN Systems 29 (8): 1157–1166. [doi:10.1016/s0169-7552(97)00031-7.](http://www.sciencedirect.com/science/article/pii/S0169755297000317)
[^4]:[HBase MapReduce Examples](http://hbase.apache.org/0.94/book/mapreduce.example.html)
[^5]:[HBase and Hadoop](https://bighadoop.wordpress.com/2012/07/06/hbase-and-hadoop/)
[^6]:[PHP Autocomplete](http://markonphp.com/autocomplete-php-jquery-mysql-part1/)



