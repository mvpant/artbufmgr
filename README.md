# ARTful Buffer manager in PostgreSQL

This is a main page of the project that took part in the GSoC 2019 event as a participant of PostgreSQL organization. The main goal of this project was to implement and evaluate an adaptive radix tree (abbreviated as ART) as an alternative to the current underlying data structure of PostgreSQL buffer manager.

Project's proposal can be found in the [google document](https://docs.google.com/document/d/1HmhOs07zE8Q1TX1pOdtjxHSjjAUjaO2tp9NSmry8muY/edit?usp=sharing) or this [repository](docs/proposal.pdf).

Origin of the project's idea comes from several presentations and community discussions about inefficiencies of the hash table as a mapping data structure:

* [Improving Postgres' Buffer Manager](https://www.postgresql.eu/events/fosdem2016/schedule/session/1184-improving-postgres-buffer-manager/)
(https://wiki.postgresql.org/wiki/FOSDEM_2016)
* [PostgreSQL's buffer manager - Problems & Improvements](https://www.pgcon.org/2016/schedule/track/Hacking/958.en.html)
* [drop/truncate table sucks for large values of shared buffers](https://www.postgresql.org/message-id/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR%2BAw3atNfj9WkAGg%40mail.gmail.com)
* [Reducing the size of BufferTag & remodeling forks](https://www.postgresql.org/message-id/20150702133619.GB16267%40alap3.anarazel.de)
* [Speedup of relation deletes during recovery](https://www.postgresql.org/message-id/CAHGQGwHVQkdfDqtvGVkty%2B19cQakAydXn1etGND3X0PHbZ3%2B6w%40mail.gmail.com)
* [pgsql: Checkpoint sorting and balancing](https://www.postgresql.org/message-id/E1aeBt7-0005jP-Il%40gemulon.postgresql.org)

## Technical details
If you are interested in all details, then there is solid chapter about [the internals of PostgreSQL buffer manager](http://www.interdb.jp/pg/pgsql08.html) that describes the components of the subsystem and what role does the hash table (also named as buffer table) play in it.

In the essense, the hash table simply maps a BufferTag structure into an integer index of the buffer pool array. A buffer pool is a fixed-length array and by default consists of 8Kb slots/pages. It essentially does the same job as the Linux page cache subsystem — improves the overall perfomance by minimizing the disk I/O.

The BufferTag structure looks the following way:
```C
/*
 * Object ID is a fundamental type in Postgres.
 */
typedef unsigned int Oid;

typedef struct RelFileNode
{
	Oid			spcNode;		/* tablespace */
	Oid			dbNode;			/* database */
	Oid			relNode;		/* relation */
} RelFileNode;

typedef enum ForkNumber
{
	InvalidForkNumber = -1,
	MAIN_FORKNUM = 0,
	FSM_FORKNUM,
	VISIBILITYMAP_FORKNUM,
	INIT_FORKNUM
} ForkNumber;

typedef struct buftag
{
	RelFileNode rnode;			/* physical relation identifier */
	ForkNumber	forkNum;
	BlockNumber blockNum;		/* blknum relative to begin of reln */
} BufferTag;
```
and identifies which disk block the buffer contains. To find out if a requested block is present in the buffer pool array,
a 20 bytes BufferTag instance goes through a hash function that produces a hash code. The hash code is then used in two operations:
* To calculate and lock a specific buffer partition — a logical entity that improves access to the shared hash table in a highly concurrent environment. Currently, there are 128 partitions/locks that let one to simultaneously read and modify different parts of a hash table in the shared memory.
* To calculate a bucket number — a cell in a hash table array that contains the head of a linked list. The latter is one of the ways to resolve hash collisions.

Once a partition lock is obtained, thus preventing the range from concurrent modifications, a search is performed in a bucket's linked list. In case of success the caller gets the index of the buffer pool cell and, therefore, can read it from memory without disk I/O. Otherwise, process attempts to read the block from a disk and inserts corresponding entry into the hash table.

This straightforward data structure works fine in main scenario, but lacks additional properties that can be useful in other cases. Mainly, the hash table does not have any data locality, so neighbour blocks of the same relation can be placed in completely different spots. Ignoring the penalty on the CPU level, lack of this property forces one to iterate over the whole buffer array to find the blocks associated with certain relation. The latter can be required in the case of table or index removal.

---

In this project we make an attempt to provide the missing properties by replacing the existing hash table data structure with a radix tree that: a) has data locality; b) preserves order; c) doesn't require balancing. For this purpose an [ART](https://db.in.tum.de/~leis/papers/ART.pdf) was chosen, as it is a space efficient data structure comparing to the straightforward [Linux implementation](https://lwn.net/Articles/175432/) with equal-sized nodes.

The main reason why one shouldn't use radix tree with equal-sized nodes and no compression for such long keys as the BufferTag (20 bytes) structure is that the height of it will grow linearly. With a span of 1 byte — a chunk of the key that is used for navigation on each level, the height of the tree will be 20. This is `8(pointer size) * 256(array of children) * 20 = 40960` bytes to save a single key.

To be an space efficient data structure an adaptive radix tree has three features:
* different types of nodes with 4, 16, 48 and 256 children;

ART changes its structure depending on how many children the node actually has, rather than how many it might have. As the number of children in a node changes, ART transparantely swaps out one implementation of the node structure for another.
* lazy expansion;

Inner nodes are only created if they are required to distinguish at least two leaf nodes.
* path compression.

All inner nodes that have only a single child are removed.

All details of these techniques are thoroughly described in the [paper](https://db.in.tum.de/~leis/papers/ART.pdf) and can be seen in the [open-source algorithm implementation](https://github.com/armon/libart), that was also used & adapted in this project.

---

Primarily we have started with a single-lock ART, using the same locking strategy as an existing hashtable.
It was obvious that single-lock tree, by definition, can't stand
128-locks hashtable in all concurrent cases, so the main idea was to split a single tree into a tree of trees.
So, currently, the main tree operates on the first 16 bytes of BufferTag and contains subtrees as leaves. Therefore, each subtree works only with 4 byte BlockNumber and have a separate lock.

Such separation had additional benefits,
besides throughput improvement:
* At a certain point in time the main tree is almost unchanged, as it gets saturated by "most used" relation's keys. So it is mostly used as a read-only structure.
* A "path" to specific subtree of the main tree can be cached in SMgrRelation data structure, therefore reducing the cost of lookup operation from 20 to 4 bytes.

## Installation
There are two options to use ART as an alternative data structure:
- a) to clone this [branch](https://github.com/mvpant/postgres/tree/artbufmgr)
- b) to apply the patch from this repository to the [clean 11.3 branch](https://github.com/postgres/postgres/tree/REL_11_3) (preferable)



## Results
All the numerical evaluations are presented in the following [google spreadsheet](https://docs.google.com/spreadsheets/d/1VfVY0NUnPQYqgxMEXkpxhHvspbT9uZPRV9mflu8UhLQ/edit?usp=sharing).

For testing purposes TPC-H benchmark and pgbench utility were used.

## TODO
I am planning to continue work on this project and here is an ordered list of what should be done in the first place:
* There is a const size list of pre-allocated subtrees (10000), which aren't recycled even when they become completely empty. Easy way to see this is to run `install makecheck` in PostgreSQL directory, that creates many new tables and indexes in a regression database, then inspect the output of `select * from pg_buffertree_common;` query. To overcome this problem 2 steps should be done: 1) cut off ForkNumber from the key of the main tree and replace leaf structure with array[MAX_FORKNUM] of tree pointers, one for each fork. This change will a) decrease the height of main tree even further b) simplify subtree cache eviction in backends, so a similar mechanism to the DROP operation can be applied.
* The freeLists of inner ART nodes are pre-allocated with certain size — the initial sizes of freeLists should be reduced and dynamic additional allocation (the same way as it done in hash table) should be added.
* There are 4 freeLists of inner ART nodes for each type — that should be increased to decrease possible contention.

## Useful references
1. [PostgreSQL documentation: Installation from Source Code](https://www.postgresql.org/docs/current/installation.html)

