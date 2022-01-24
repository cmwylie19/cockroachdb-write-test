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

## Comparison of Availability Models
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

## Prerequisites
[Install CockroachDB Cli](https://www.cockroachlabs.com/docs/v21.2/install-cockroachdb) 
[Install HA Proxy](https://www.cockroachlabs.com/docs/v21.2/demo-fault-tolerance-and-recovery#step-2-set-up-load-balancing)


## Step 1. Start a 6-node cluster
**Start 6 instances of CockroachDB.**
```
cockroach start \
--insecure \
--store=fault-node1 \
--listen-addr=localhost:26257 \
--http-addr=localhost:8080 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--background

cockroach start \
--insecure \
--store=fault-node2 \
--listen-addr=localhost:26258 \
--http-addr=localhost:8081 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--background

cockroach start \
--insecure \
--store=fault-node3 \
--listen-addr=localhost:26259 \
--http-addr=localhost:8082 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--background

cockroach start \
--insecure \
--store=fault-node4 \
--listen-addr=localhost:26260 \
--http-addr=localhost:8083 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--background

cockroach start \
--insecure \
--store=fault-node5 \
--listen-addr=localhost:26261 \
--http-addr=localhost:8084 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--background

cockroach start \
--insecure \
--store=fault-node6 \
--listen-addr=localhost:26262 \
--http-addr=localhost:8085 \
--join=localhost:26257,localhost:26258,localhost:26259 \
--background
```

**Use the cockroach init command to perform a one-time initialization of the cluster**   
```
cockroach init \
--insecure \
--host=localhost:26257
```

## Step 2. Set up load balancing
In this module, you'll run a sample workload to simulate multiple client connections. Each node is an equally suitable SQL gateway for the load, but it's always recommended to spread requests evenly across nodes. You'll use the open-source HAProxy load balancer to do that here.  

**Run `cockroach gen haproxy` command specifying the port of any node, this command generates an `haproxy.cfg` file automatically configured to work with the nodes of your running cluster**
```
cockroach gen haproxy \
--insecure \
--host=localhost \
--port=26257
```

**In haproxy.cfg, change bind :26257 to bind :26000. This changes the port on which HAProxy accepts requests to a port that is not already in use by a node.**
```
sed -i.saved 's/^    bind :26257/    bind :26000/' haproxy.cfg
```

**Start HAProxy with `-f` flag pointing to the `haproxy.cfg` file:**
```
haproxy -f haproxy.cfg &
```

## Step 3. Run a sample workload
Now that you have a load balancer running in front of your cluster, use the cockroach workload command to run CockroachDB's built-in version of the YCSB benchmark, simulating multiple client connections, each performing mixed read/write operations.


## Step 4. Check the workload
Initially, the workload creates a new database called ycsb, creates a usertable table in that database, and inserts a bunch of rows into the table. Soon, the load generator starts executing approximately 95% reads and 5% writes.


## Step 5. Simulate a single node failure
When a node fails, the cluster waits for the node to remain offline for 5 minutes by default before considering it dead, at which point the cluster automatically repairs itself by re-replicating any of the replicas on the down nodes to other available nodes.


## Step 6. Check load continuity and cluster health
Go back to the DB Console, click Metrics on the left, and verify that the cluster as a whole continues serving data, despite one of the nodes being unavailable and marked as Suspect:


## Step 7. Watch the cluster repair itself
Click Overview on the left:

## Step 8. Prepare for two simultaneous node failures
At this point, the cluster has recovered and is ready to handle another failure. However, the cluster cannot handle two near-simultaneous failures in this configuration. Failures are "near-simultaneous" if they are closer together than the server.time_until_store_dead cluster setting plus the time taken for the number of replicas on the dead node to drop to zero. If two failures occurred in this configuration, some ranges would become unavailable until one of the nodes recovers.


## Step 9. Simulate two simultaneous node failures
Use the cockroach quit command to stop two nodes:

## Step 10. Check load continuity and cluster health
## Step 11. Clean up
[Demo Link](https://www.cockroachlabs.com/docs/v21.2/demo-fault-tolerance-and-recovery)


# References
[Cockroach Facts](https://www.pestworldforkids.org/pest-info/bug-articles-by-type/how-long-can-a-cockroach-live-without-its-head/#:~:text=So%2C%20how%20long%20can%20a,mouth%20or%20head%20to%20breathe.)  

[CockroachDB Availability Model](https://www.cockroachlabs.com/docs/v21.2/multi-active-availability.html) 