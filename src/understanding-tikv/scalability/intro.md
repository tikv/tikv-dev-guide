# Scalability

In the database field, `Scalability` is the term we use to describe the capability of a system to handle a growing amount of work. Even if a system is working reliably and fast today, it doesn't mean it will necessarily work well in the future, especially when the increased load exceeds what the system can process. In modern systems, the amount of data we handle can far outgrow our original expectations, so scalability is a critical consideration for the design of a database.

There are two main types of scalability:

1. Vertical scaling, which is also known as scaling up, means adding resources to a single node in a system, typically involving the addition of CPUs or memory to a more powerful single computer. Vertical scaling is limited by the technology of semiconductor industry, and the cost per hertz of CPU or byte of memory will increase dramatically when near the technical limit.

2. Horizontal scaling, which is also known as scaling out, means adding more machines to a system and distributing the load across multiple smaller machines. As computer prices have dropped and performance continues to increase, high-performance computing applications have adopted low-cost commodity systems for tasks. System architects may configure hundreds of small computers in a cluster to obtain aggregate computing power that often exceeds that of computers based on a single traditional processor. Moreover, with widespread use of [Cloud computing](https://en.wikipedia.org/wiki/Cloud_computing) technology, horizontal scalable is necessary for a system to adapt to the resiliency of Cloud.

A system whose performance improves after adding hardware, proportionally to the capacity added, is said to be a scalable system. It's obvious that a scalable system depends on the ability of horizontal scaling.

TiKV is a highly scalable key-value store, especially comparing with other stand-alone key-value stores like [RocksDB](https://rocksdb.org/) and [LevelDB](https://github.com/google/leveldb).

To be scalable, TiKV needs to solve the following problem:

1. Partitioning, how to break data up into partitions, also known as _sharding_, to fully utilize resources of nodes in cluster. In TiKV, a partition is called a `Region`.

2. Scheduling, how to distribute the `Region` in cluster, and balance the load among all nodes, for eliminating hot spots or other bottle necks.

In the rest of this chapter, we will talk about `Region` and `Scheduling` of TiKV.
