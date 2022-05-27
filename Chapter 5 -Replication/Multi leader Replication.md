---
aliases: replication, multi-leader, multi-leader replication
---
# Multi-leader Replication

## use cases 
### multi data center operation 
- allows us to tolerate failure of an **entire** datacenter 
- also allows each data center to process writes independently of other datacenters 
- aids performance - don't need to wait on network if each node can process writes 
- can also tolerance datacenter outages - if single leader fails, need to wait to promote; multi leader don't care, can just promote within data center and wait 
- single leader sensitive to *inter* datacenter network issues as we need to wait for replication 
- **but** data might have conflict so we need to resolve 

### clients with offline operation 
- calender app requiring sync is essentially a "multi-leader" scenario, where the local calendar is leader and other devices are aso leaders.
- each device is a "datacenter" that has extremely unreliable connection  
	- we need to be able to edit even without connection and for these edits to be synced properly when network comes back on 

### collaborative editing 
- essentially also a multi-leader problem. local edits are first shown on local leader before async replication onto other users + server
- 1 way to guarantee no editing conflict is to have a lock on doc before edit then unlock after but this is extremely slow, esp when large amount of users 

## handling write conflicts 
### sync vs async conflict detection 
- single leader (sync) can just wait for first write to complete before accepting second write.
- multi leader will just accept both writes and might lead to conflict. this conflict might not be detected even at second write - only at read. (eg: first write then second is logical order but network delays lead to second -> first)

### conflict avoidance 
- very hard to resolve conflicts - we can avoid them by making writes (deterministically) go through a specific leader
- this will fail if we need to reroute in the event of failure (eg: if we cannot tolerate delay)

### convergence
- multi leader no defined write order - if we have 2 write requests to 2 leaders for the same data, (A, B) === (B, A)
- can define a monotonically increasing ID for the write (eg: timestamp)
	- **this is prone to data loss**
- can also do ID for replica 
	- also prone to data loss 
- merge values
- CRDTs 

### custom conflict resolution logic 
**on write**: need logic to determine if conflict on write and resolve before writing to db 
	- this is sensitive to time delays 
**on read**: store all write then resolve on read. can return to user first then write through the correct result to db 

### Multi leader replication topologies 
- star (1 central then spread to rest)
- all to all 
- circular 

for circular/star, a write might need to pass through several nodes before all replicas are reached. hence, nodes need to forward data changes from other nodes. to prevent infinite loops (eg: a -> b -> c -> a), writes are tagged with *identifiers* of the nodes that they have passed through

circular/star have problem where if 1 node fails, communication between nodes can be interrupted. this is especially prominent for star if central node fails. denser topology allows messages to travel along diff paths, avoiding single mode of failure.

all to all networks might face issue of *differing* network speeds along node links. some links might be faster due to less congestion etc, which results in different ordering.
	- eg: client A inserts x = 2 and client B edits so that x = 3 (from 2).
	- these requests are served by different leaders, L(A) and L(B) and it might be that L(B) sees client B's request first instead of client A
	- bug where x didn't exist prior but now need to edit value
	- this is similar to causality issue seen in *consistent prefix reads*. timestamps are not sufficient -> clocks **cannot be trusted** to be sufficiently in sync to correctly order these events  ^0b9f0b

^ This might be solved using ***[[Leaderless Replication#version vectors|version vectors]]***

