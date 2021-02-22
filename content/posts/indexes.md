---
title: "Indexes"
date: 2021-02-21T19:45:34-08:00
draft: true
---

Knowledge of database systems is essential to software engineers, and it's unfortunate that I don't have much exposure to databases (especially relational ones) at work. Luckily, there's this [highly rated database course](https://15445.courses.cs.cmu.edu/fall2020/) from CMU with quite a lot of learning materials, which provides me with a good chance to catch up on this area.

My current plan is to go over indexing and transactions first and this particular post will be grouping some index related notes. The major reason behind this this is that I have recently just gain some high-level understanding of these 2 topics by reading [Designing Data-Intensive Applications](https://www.amazon.com/Designing-Data-Intensive-Applications-Reliable-Maintainable/dp/1449373321) book, and going over them in more details again could facilitate these knowledges.

This post will go over the followings:

* structure of B+ tree
* comparison between B tree and B+ tree
* operations on B+ tree (insertion, deletion)
* handling of duplicated keys
* more specific B+ tree index types (composite index, clustered index, covering index, partial index)
* B+ tree optimizations
* concurrency control on B+ tree index

Images in this post are borrowed from course materials of the course linked above.

# Structure of B+ Tree

![alt text][structure]

[structure]: /images/indexes/b+_tree.jpg "B+ tree structure"

In B+ tree, each node fits into a fixed-size page. There are generally 2 types of B+ tree nodes: inner (or intra) nodes, and leaf nodes.

Inner nodes consist of key boundaries and identifiers of their children's pages (i.e. record id, or "pointers" to their children). Typically a node in modern RDBMS will have several hundreds of pointers to their children, and if the boundaries and pointers of a node can't fill up a entire page, that page will have some blank, ununsed space. Inner nodes of B+ tree need to be at least half-full to maintain the balancing feature of B+ tree, which means if a node is less than half full, there will be some re-balancing operations. This operation, along with the case where a insertion causes a node to be overfull, will be discussed in "operations on B+ tree" section.

On the other hand, a leaf nodes stores both the keys and the corresponding "values". Double quote here is added because different databases may have different implementations of the "values" (we'll get to this in "more specific B+ tree index types" section), but in general they will provide information for us to find the value in a tuple. It's worth pointing out that the sibling pointers were not included in the original concepts of B+ tree, but was introduced by a variation of B+ tree called B-link tree. These pointers are added so that range queries can directly move to the sibling pages when neede, instead of going back to the parent level and then down to leaf nodes again, which is more efficient.