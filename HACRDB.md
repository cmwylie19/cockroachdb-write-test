# CockroachDB - High Available by default 
Before we dive into the nuts and bolts of the extremely resilient Cockroach database, lets explore some interesting parallels between CockroachDB and its insect counterparts.    

**Question:** How long can a roach live without its head?   
**Answer**: _Up to a week_

**Question:** How long can a roach live without water?  
**Answer:** _Up to a week_

**Question:** How long can a roach go without food?   
**Answer:** _Up to a month!_

Cockroaches can hold their breath for forty minutes! They can even survive being submerged underwater for up to thirty minutes.

Like the insect, CockroachDB is designed survive and thrive through failure. 

## What does is mean to be HA?
High availability lets an application continue running even if a system hosting one of its services fails. This is achieved by scaling the application's services horizontally, i.e., replicating the service across many machines or systems. If any one of them fails, the others can simply step in and perform the same service.

## Availability Models
CockroachDB's availability model is described as **Multi-Active Availability.** In essence, multi-active availability provides benefits similar to traditional notions of high availability, but also _lets you read and write from every node in your cluster without generating any conflicts_.
    
This availability model has several advantages other models. For instance, in an Active-Passive model, all traffic is routed to a single, "active" replica. Changes to the replica's state are then copied to a backup "passive" replica, in an attempt to always mirror the active replica as closely as possible.

In an Active-Passive model, if you use asynchronous replication, you cannot guarantee that any data is ever successfully replicated to passive followers––meaning you can easily lose data.

If you use synchronous replication and any passive replicas fail, you have to either sacrifice availability for the entire application or risk inconsistencies.

In an Active-Active model, multiple replicas run identical services, and traffic is routed to all of them. If any replica fails, the others simply handle the traffic that would've been routed to it. For databases, though, active-active replication is incredibly difficult to instrument for most workloads. For example, if you let multiple replicas handle writes for the same keys, how do you keep them consistent?

This means that application will continue running even after a system hosting one of its services fails. This is achieved bu scaling the application's services horizontally, i.e., replicating the service across many machines or systems. If one fails, the others can simply step in and perform the same service.

**Currently in our ACM setup, we are using an mutli-active availability model, where multiple replicas run the same service, and traffic is routed to all of them. If any replica fails, the other handle the traffic that would have been routed to it.**

---
## Cockroaches are hard to kill (Demo)
This demo walks you through how CockroachDB remains available during, and recovers after, failure. Starting with a 6-node local cluster with the default 3-way replication, you'll run a sample workload, terminate a node to simulate failure, and see how the cluster continues uninterrupted. You'll then leave that node offline for long enough to watch the cluster repair itself by re-replicating missing data to other nodes. You'll then prepare the cluster for 2 simultaneous node failures by increasing to 5-way replication, then take two nodes offline at the same time, and again see how the cluster continues uninterrupted.

[Demo Link](https://www.cockroachlabs.com/docs/v21.2/demo-fault-tolerance-and-recovery)


# References
[Cockroach Facts](https://www.pestworldforkids.org/pest-info/bug-articles-by-type/how-long-can-a-cockroach-live-without-its-head/#:~:text=So%2C%20how%20long%20can%20a,mouth%20or%20head%20to%20breathe.)  

[CockroachDB Availability Model](https://www.cockroachlabs.com/docs/v21.2/multi-active-availability.html) 