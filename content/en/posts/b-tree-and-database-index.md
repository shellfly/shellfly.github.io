---
title: "B-tree and database index"
date: 2020-02-22T15:14:45+08:00
draft: false
---

 I know database index was implemented by B-tree for a long time, but actually I don’t understand how it works in detail and what stored in the tree node. Recently, I reread the book *Algorithms, 4th Edition* and there is a B-Tree implementation in the last chapter as an extend topic. I start to understand a bit after read the book, and then I did some search in Youtube and found an *excellent tutorial* which explains the B-tree and index very well. I’m writing this help myself understand the topic by reorganized the content in the video, and I highly recommend everyone who is interested in this topic to have a watch as well（link will be provided below）.

There are four parts in this post:
1. Latency of disk access
2. Index structure
3. B-tree
4. Difference of index in MySQL and PostgreSQL

## Lagency of disk access

Disk access is far slower compare to main memory. Assume that we don't have index, the data stored in the disk is just the tables(usually database stores one table in one file). When want to access one of some of the records, the whole file has to be read from disk because we don't know the address of the subset data we want in advance. and you can image that how it costs if we just want to update one row in a table. Even though SSD is common today, it still takes 0.25 to 1ms latency to access it, and the best speed is around 500MB/s to 1GB/s

So we need to a mechanism to get the exactly address pointer of the data, which is sector in HDD or page in SDD, so that we can minimize disk access, and that is index's job.

## Index structure

As a developer, you must already have a high level understand of the index concept, it's kind like the book index, you can know the exact page number of the content you are interested in. It also called page in database, but it's the page of data.

Index is constructed like a tree, so we can start from the simplest form(Binary Search Tree) first, and then see how to improve it for index.
Binary search tree is a tree that every node has two children, and the key is greater or equal to left and less then right. the time to access any data in the tree is **logarithmic time** with factor of two in ideal case.

There are two things need to be optimized if we want to use a tree for index, one is the **tree height**, as mentioned above, we want to minimize disk seek times, we can do this by change a binary search tree to a M-way search tree which means every node can have at most M-1 keys. And another thing is we have to maintain the **tree balance** so that every node has at least M/2 keys and all leaf nodes are in the same level.

## B-tree

B-tree implements two features mentioned above and is used for index in RDMS. Index itself is a kind of data in database which also need to be persisted in disk, and of course it's also splited to pages. PostgreSQL uses 8 kB page size for table and index while MySQL use 16 kB page size. Depends on server disk block size setting which may need one or two times to read one page. To minimize disk access, database usually use one page as a tree node. 

There is also a variant of B-tree called B+tree, the difference is that the internal node in B+tree doesn't store key-value pairs but all keys so that in the limit of page size, the internal node can hold more keys. All internal node will be duplicated in leaf nodes and data only stored in leaf nodes. the leaf nodes are also maintained in a linked list for fast traversal.

Both MySQL and PostgreSQL use this variant, although PostgreSQL doesn't called it B+tree, because they regard B+tree as store data in the tree but they don't store table data in a tree directly like InnoDB does.

## Difference of index in MySQL and PostgreSQL

MySQL has a special index called **clustered index** which is used for primary key, table data is organized in disk based on the value of primary key. And other index are called secondary index, the value in secondary index is primary key. 

With clustered index, it's very fast to retrive data by primary key because there is only a single I/O operation. And this is very often when you order by auto incremented ID or joined other table by foreign key.

And the downside of clustered index is that it requires twice tree nodes traversal when using secondary index, first scan over secondary index to get the primary key and then walk though clustered index to get data.

PostgreSQL split table and index, table is stored in regulary table structure called heap. the value in index is the row postion in disk. you don't get better performance by using primary key and you don't pay extra to use seconday key either.

Another benefit to use clustered index is that when you update/datele a record in table, to support MVCC, there will be different versions of data exists at the same time. if you don't have clustered index, you have to update every index to pointer to the correct positions. Although PostgreSQL has the HOT feature which eliminates redundant index entries, it has the limits that the new tuple has to be stored on the same page and the update doesn't update an index column. This is also the main reason why Uber migrated from PostgreSQL to MySQL.


Reference

###### https://github.com/kevin-wayne/algs4

###### https://www.youtube.com/watch?v=aZjYr87r1b8

###### http://codecapsule.com/2014/02/12/coding-for-ssds-part-2-architecture-of-an-ssd-and-benchmarking/

###### https://use-the-index-luke.com/sql/anatomy

###### https://www.postgresql.org/docs/12/storage-page-layout.html

###### https://dev.mysql.com/doc/refman/8.0/en/innodb-physical-structure.html

###### https://en.wikipedia.org/wiki/Talk%3AB%2B_tree#PostgreSQL's_use_of_B+_trees

###### https://github.com/postgres/postgres/blob/master/src/backend/access/heap/README.HOT

###### https://eng.uber.com/mysql-migration/