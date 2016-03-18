Thinking about operation and optimization of MySQL 
----
### Expensive Operation
In this phase, all the requests are **uniform**, **readonly** operations, which means we are not required to consider ACID, writing performance and reliability,   so how to locate the row(s) match certain requirement is the biggest challenge. This problem could be further split into two subproblems:

- divide
- search

which mean we could both optimize the search algorithm time complicity and reduce the total amount of data the algorithm has to traverse.

In MySQL, column with Index would be traversed along the Btree, otherwise, by primary key(record id), if the corresponding column of query item does not have an index, the performance could be a disaster(≥ 600000 ms), by implement index the first-time query could be improved largely(≥ 1000 ms), but the worst performance is still far away from requirement. So we further partition the big table into 100 separated partitions using Hash, this time, when a query comes in, say `user_id` = 123 and `hash_tag` = 'love', hash function would first determine that all records of `user_id` 12345 fall into partition `p3`, the total number of raws is as much as 90,000,000, however in `p3` there are only 600,000. Within a certain partition, the performance could be further improved to 100ms. Finally ,which the built-in cache of mysql, in the second time, the latency would fall below 10 ms.

However, 10ms latency only secure a 4000+ rps, which is far less than the required 10,000 rps. According to our calculation, to achieve that goal, our application have to reduce the latency to less than 3 ms, that is cruel for MySQL! To overcome this problem, we add a addition cache on frontend with an optimized capacity. By using temporal locality, the front end could directly return the cached value without querying the database, which dramatically reduce the latency to required 3 ms.






