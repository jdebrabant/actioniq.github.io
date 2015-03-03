---
layout: post
title: "Adaptive Range Filters"
published: true
date: 2015-03-02
---

At ActionIQ, we love data. With our company being founded on a core of databases and distributed systems,
its no wonder our water cooler chats are more likely to be about query optimization, shuffle algorithms,
and partitioning techniques than weather. We also actively follow popular conferences and research coming
from academia, and like to highlight particularly interesting work. In this post, we’ll cover a
[paper](http://www.vldb.org/pvldb/vol6/p1714-kossmann.pdf) from VLDB 2014 about a new data structure called
an Adaptive Range Filter (ARF) that extends the basic idea of Bloom Filters (BF) to range queries. We'll give
a summary and overview of this paper as well as any requisite knowledge regarding BFs.

### Bloom Filter Basics

To start, let's define a BF and understand what it is and isn't good at. A BF is a compact data structure used
to determine if something is contained in a given set in constant time O(k), where k is fixed and independent
of the number of elements in the set. BFs guarantee that there will never be a false negative (i.e.,  the BF
will never return false for some value when really the value is actually in the set). However, there is no guarantee
about false positives (i.e., the BF might return true when the value is not present in the set). These properties,
specifically the relaxation on false positives, are what allow BFs to be so compact relative to other fixed constant
time lookup data structures, such as hash tables. At their core, BFs are a probabilistic data structure that try to
reduce the false positive rate as much as they can while also staying compact.

BFs are often used in databases because there is usually a large data store (e.g., a very large table stored on disk)
for which it is expensive to query, especially if it is only to find out that the item is not present in the table.
Alternatively, data can be encoded into a BF, where the BF is queried first, and the table is only queried if the BF
returns true.  Doing a BF lookup first adds negligible overhead in the positive case. For the negative case, because
a BF lookup is orders of magnitude faster than a disk read (or, more generally, a read from any large data store on
persistent storage),  a BF dramatically reduces query time since the larger data store no longer needs to be queried.
A prime example of this is the new memory-optimized engine in SQL Server, called
[Hekaton](http://research-srv.microsoft.com/pubs/218305/p1016-eldawy.pdf), that uses BFs to determine whether a tuple
is contained in memory or on-disk, since the two sets of tuples are mutually exclusive. 

### Problem with Ranges

While BFs are designed for point queries (e.g., "is this user in the table?"), database queries often contain range
filters (e.g., "users whose salary is between $20K-$50K"). To use BFs on a range query, the system must individually
query the BF for each possible value in the range predicate. This can be very inefficient (in our example it's 30K queries)
and can defeat the purpose of using BFs in the first place. In this paper, the authors describe a new data structure
called Adaptive Range Filters  to handle range queries in an efficient and compact way. As with BFs, ARFs provide the same
guarantee that there will never be a false negative, while also aiming to reduce the false positive rate. 

### The ARF Solution

ARFs work by encoding the entire domain of a range into a binary tree where each node represents a sub-range (lower, upper).
The root node encompasses the entire domain of the ARF tree. The children of a node split that node's range in half. Because
each node splits its range exactly in half, the actual ranges do not need to be stored in memory, but rather can be computed
as the tree is traversed. In fact, all that needs to be stored is information about the shape of the tree. For a given node,
there are four possible scenarios: 

1. Both of its children are leaf nodes
2. The left child is a leaf node, and the right is an intermediate node
3. The right child is a leaf node, and the left is an intermediate node
4. Both of its children are intermediate nodes

Because there are only four options, this information only takes two bits per node. Thus, it is incredibly compact and efficient
for traversal, and because you don’t need to store any pointers, the entire tree can be encoded as a sequential bit vector. The
leaf nodes of the tree contain an occupied bit specifying whether their range is in the set or not. These two pieces of information
(two "shape" bits for intermediate nodes, one "occupied" for leaf nodes) are all that is needed to encode the ARF.

### Querying with ARF

Using ARFs for range queries means doing a single lookup for the "users with salary within 20-50K" query above instead of the 30K
lookups needed for a BF. Furthermore, ARFs are also designed to return which ranges possibly contain data as opposed to a single
true/false for the entire range. In our example query, if the ARF responds with the ranges 12-15K and 18-22K, the system knows that
there are no users outside this range, so values outside of these ranges can be safely ignored in the underlying data store, thereby
potentially significantly reducing the amount of data that must be accessed to answer the original query. Because for many queries
the system is bottlenecked by reading data from disk, this can be a significant optimization. 

### Adaption to Query Workload

Another key feature of ARFs is that they learn dynamically from queries (hence the "adaptive" portion of their name). With traditional
BFs, when the data store is updated you also update the BF in order to maintain its correctness. BFs remain true to the underlying
dataset, but they are not tuned to a particular query workload. ARFs, on the other hand, adaptively restructure as queries are given
(even without any changes in the data) in order to focus more tree bits to cover the queries in the current workload, since more bits
for a given range reduces the amount of false positives of queries in that range. The adaptive algorithm works by learning from false
positives. Below is an example of the sequence of operations for adapting the ARF to the current query workload: 

1. Issue query to ARF for range R
2. ARF returns true for subranges R1, R2, and R3
3. Query database with R1|R2|R3
4. Find out that R2 is not in the set
5. Tell ARF that R2 is a false positive
6. ARF adjusts its tree to prevent false positive from happening again
7. Issue query to ARF for range R2
8. ARF returns false

Over time, the adaptation means the ARF learns the query workload and optimizes to the minimum false positive rate while maintaining space constraints.

When data is added to the underlying set, the ARF must be updated to prevent false negatives. The same is true of BFs. However, when data
is deleted from the data store, no action needs to be taken on the ARF. The next time that data is queried the ARF will return a false
positive, and then adapt to that change as described above. In fact, it is often more efficient not to update the ARF because the item that
was deleted may never actually be queried, in which case those bits in the ARF can be saved for more actively queried regions.

### Conclusion

In summary, the ARF provides a BF-like interface for range queries, while also adapting to changing workload conditions in order to minimize
false positives within the given space constraints. BFs are a key element of many current distributed systems that offer key-based lookups,
including NoSQL systems like [Apache Cassandra](http://cassandra.apache.org) as well as distributed storage systems such as [Google BigTable](http://static.googleusercontent.com/media/research.google.com/en/us/archive/bigtable-osdi06.pdf). As new and current systems expand to support richer query features, in particular the support for range queries, ARFs will likely be a key element in making such queries efficient. 
