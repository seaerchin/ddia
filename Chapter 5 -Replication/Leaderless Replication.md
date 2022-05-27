---
aliases: replication, leaderless, leaderless replication
---
# Leaderless Replication 
- in leaderless replication, **any** replica can accept writes from client. 
	-  some implementations uses a parallel approach, where a read request is serviced by many different clients 
	- others use a co-ordinator node. however, this node does **not** impose any write-ordering  ^3ac860
		- this is not constrained to any specific node -> when node *n* is co-ordinating at *t* and it crashes at *t + 1*, some other node *m* can take over

## Writing to DB when a node is down 
- in single leader approach, you will wait for leader to come back up (through promoting perhaps)
- in leaderless config, no such thing as failover.
	- if client fires off a write request in parallel to diff replicas and it is deemed a success if majority ACKs, we have some minority of replicas without that write. 
	- if node fails then comes back online, it might service clients with stale data due to missing the write
	- hence, read requets are also fired **in parallel**

### Read repair and anti-entropy
- *Read repair* ^d802a5
	- when a client makes a read from several nodes in parallel, it can detect any stale responses (**how?** - attaching version values would work but what if a write misses and you end up with a version that contains outdated data in reality?) 
		- attach version numbers, then resolve using some merge strategy (LWW/union etc)

- *Anti-entropy process* ^a9633c
	- background process that looks for differences in data between replicas and does a copy of missing data between replicas.
	- writes are **not** copied in any order 
	- there may be significant delay before data is copied  

### Quorums for reading/writing 
- number of writers (w) + number of readers (r) > number of nodes (n)
	- referred to as having *quorum*
	- read heavy workloads want to have higher number of writers to confirm (decreases number of machines required for read); vice versa for writes ^fab0c6

- having *w + r > n* allows the system to process unavailable nodes as follows: 
	 - if *w < n*: we can still process writes if a node is unavailable
	 - if *r < n*, we can still process reads if a node is unavailable
	 - *(w | r) - n* is # of nodes that can fail for either write/read

usually, reads/writes are sent to **all** nodes in parallel; *w/r* refers to the number of nodes that need ACK.

### Limitations of Quorum Consistency
if [[#^fab0c6|quorum]] is obeyed, every read should return latest value as most recent value would have been stored in at least 1 node

- usually *r* and *w* are chosen to be a majority (> n // 2) so that *w + r > n* while still tolering up to n // 2 failures. 
	- quorums don't have to be majorities -> as long as r && n intersect in 1 node that is sufficient

- conversely, if *w + r <= n*, quorum is **not** satisfied and there might be stale reads. however, you get increased read/write throughput (less nodes need to ACK to consider success) 
	- tradeoff b/w correctness vs availability 

- However, even given the quorum condition, there are still edge cases where stale values are returned
	 - *[[#^26c125|sloppy quorum]]* -> writes end up on different nodes than reads, hence no longer overlap between read/writes
	 - concurrent writes -> unclear of ordering. safest solution is to merge but writes can be lost due to [[Multi leader Replication#^0b9f0b|clock skew]]
	 - concurrent read/write -> unclear ordering -> reads may return old/new value 
	 - if a write is *partially successful* (writes on less than *w* nodes), it might not be rollbacked on replicas where it succeeded. 
	 - if a write is reported as failed, subsequent reads may or may not reflect the value of that particular write. 
		 - for example, we write to server 1 -> client reads from 1 of (1, 2), (1, 3), (2, 3) -> for the first 2, it's reading a failed write.
	 - if a node w/ correct value fails, but data is restored from some replica carrying old value, the # of replicas storing new value might fall below *w*, thus breaking quorum
	 - timing inconsistencies 

quorums don't offer an iron-clad guarantee of reads giving the latest write value and hence, such leaderless (dynamo styled) databases are best with use cases that can tolerate *eventual consistency*

- *w* and *r* allows you to adjust probability of stale values but this is not an absolute guarantee.
	- don't get the same level of guarantees as *Problems with Replication Lag* so anomalies can still occur -> stronger guarantees only come with *transactions* or *consensus*

### Monitoring staleness
driven by need to tell if values returned are *stale*. even if app can tolerate stale values, it might be indicative of bad app health.

- for leader/[[Multi leader Replication|multi-leader]] replication, db can expose metrics for replication lag. 
	- possible because writes are **always** applied to leader then to followers in same order and each node has a position in the replication log (# of writes it has applied locally)
	- hence, by subtracting follower position from leader position, you can tell replication lag (# of entries that a node needs to write to be considered up to date)

in leaderless replication, there is **no fixed order** of writes -> monitoring is difficult + cannot do the replication log trick.

- if the database only uses [[#^d802a5| read repair]] and not [[#^a9633c | anti-entropy]], there is no limit to how old a value might be. 
	- if a value is infrequently read, the value returned by stale replica can be ancient

## Sloppy quorums and Hinted Handoff

^26c125

- databases w/ correctly configured quorums can tolerate individual node failures w/o failover. 
	- this also means that they can tolerate individual nodes becoming slow as they don't have to wait for all *n* nodes - either *w* or *r* is sufficient
- **but** a big network interruption can bisect the network into 2 disjoint sets -> any client connecting to minority set is as good as dead as their reads/writes won't be seen by majority.

### Sloppy quorum
*definition:* accept writes anyway during a network interruption and these writes will be on some nodes (*m*) that are not part of *n* ^b308e9

reads and writes still require *r* and *w* respectively but may include additional nodes that are not in *n*

### Hinted Handoff
*definition*: once network recovers, writes to *[[#^b308e9|m]]* will then be propagated back to *n*.

### Uses 
- *Sloppy quorums* are hence useful for increasing *write availability* -> as long as *any w* nodes are available, the db can accept writes
	- quorums require *w*, *sloppy quorums* only require 1
	- you are **not guaranteed** freshness of value as the latest value might be in *m*
- not a quorum, technically. more a guarantee of *durability*, that the writes will be committed somewhere safe. no guarantee of fresh reads until [[#Hinted Handoff]] occurs.

#### multi-datacenter operations
- cassandra/voldemort
	- *n* is all nodes in system
	- client writes are broadcasted to all nodes but only quorum needs to be reached for the write to be ACKed.
- riak 
	- *n* refers to replicas within a single datacenter 
	- client communcation between db is restricted to 1 datacenter. 
	- cross-datacenter replication is async and a background process

## Detecting Concurrent Writes
- writes may arrive in different orders at different nodes 
- nodes cannot simply overwrite whenever new write requests come in as data can become permanently inconsistent
- we need some notion of *eventual consistency*

### last write wins (discard concurrent writes)
- declare that each replica only stores the **most recent** values and to discard older values
- define most recent values to be the timestamp attached to each write 
- this eventually converges but at the cost of **durability** -> if there are *concurrent* writes to same key, even if they are reported successful, only the last write (by timestamp) would be preserved.

safety with lww is to ensure that a key is only written **once** and treated as **immutable** thereafter -> this avoids any further mutation and hence, concurrent updates.

[Cassandra](https://cassandra.apache.org/_/index.html) uses a UUID as the key, which makes write ops unique.

### happens-before and concurrency 
*definition*: an operation A *happens before* some other operation B if B knows about A (**what does this mean?**) or depends on A or builds upon A in some way.  ^658ce0

*definition*: two operations are *concurrent* if neither *[[#^658ce0|happens before]]* the other -> this also means that these two operations know nothing about the other

hence, with any 2 operations (A, B), there are 3 possibilities 
 - A -> B 
 - B -> A 
 - A is concurrent with B

### capturing the happens-before relation
- attach a version number to key
- increment version everytime the key is written and store version w/ value 
- when client reads key, server returns latest values + version 
	- a client **must** read a key before writing to it
- when client writes, it needs to include version from prior reads and merge values that it received in prior reads
- when server receives a write request w/ a particular version number, it can overwrite all writes with that version number of below 
	- this is because the write will supercede all earlier key versions 

### merging concurrently written values
- if several operations are concurrent, clients have to clean up by merging concurrently written values.
	- this is similar to *[[Multi leader Replication#handling write conflicts|multi-leader replication]]* in multi-leader replication
- however, if deletion is allowed, then merging by taking union may not yield the correct result
	- a *tombstone* needs to be placed at the key to indicate that it's been removed as of some version

### version vectors 
extend the versioning to include replicas -> now each replica increments it's own version number when processing a write and also tracks *seen* version numbers from other replicas.

*definition:* a collection of version numbers from all replicas is called a *version vector*. 

this version vector allows database to distinguish between *overwrites* and *concurrent writes*. 

similar to single replica, when merging *siblings* (concurrent writes), the version vector helps to ensure that it's safe to read from replica A but write to replica B