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
## CockroachDB Availability Testing
This demo tests availability of CockroachDB using different deployment modes. Secondly, this demonstrates how to use configure CockroachDB to remain available during failure and recover after failures. For our tests, we have CockroachDB deployed behind a service so we can easily test availability of CockroachDB by sending requests through the service.

- CockroachDB with 1 Replica (Deployment)
- Cockroach with 3 Replicas (StatefulSet)
- Cockroach with 3 Replicas (StatefulSet) across 3 different Clusters in 3 different availability zones.

---

First create the namespace for our testing:
```
kubectl create ns cockroachdb
```

Next, create a new cockroach pod in the `cockroachdb` namespace for interacting with the CockroachDB.
```
kubectl run crdb-tester --image=cockroachdb/cockroach:v21.2.4 --command -- sleep 9999s
```

## Single Replica Test
**Note** - _This is not a recommended approach to running cockroachDB since we have purposely removed any redudancy added by the `StatefulSet` and have no option of failover. It is generally recommended that clusters have at least three nodes, to ensure that CockroachDB’s automated replication can keep data safe and available. In this test we deploy CockroachDB through a single replica `Deployment`._

### Setup Single Replica CockroachDB
Deploy the manifests:
```
kubectl apply -f k8s/single-replica-crdb.yaml
```
output
```
serviceaccount/cockroachdb created
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-crdb-sa created
service/cockroachdb-public created
service/cockroachdb created
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/cockroachdb-budget created
deployment.apps/cockroachdb created
persistentvolume/crdb-pv created
persistentvolumeclaim/crdb-pvc created
``` 
Get the pods in the cockroach namespace, wait until they are running
```
kubectl get pods -n cockroachdb
```
output:
```
NAME                           READY   STATUS              RESTARTS   AGE
cockroachdb-555865976b-dngh8   1/1     Running             0          19s
crdb-tester                    1/1     Running             0          8m52s
```

**Single Replica- Write Test**, create a table.
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public:26257 --execute="CREATE TABLE roaches (name STRING, country STRING); INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States')"
```
output: ✅
```
INSERT 2
```
**Single Replica- Read Test**, read from the table.
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public:26257 --execute="SELECT * FROM roaches;" 
```
output: ✅
```
          name          |    country
------------------------+----------------
  American Cockroach    | United States
  Brownbanded Cockroach | United States
(2 rows)


Time: 36ms
```
**Single Replica- Simulate Failure in pod, read from table**, 
We complete this test by contually killing the CockroachDB instance and try reading form the `cockroach-public` service while the pod is down. 
```
# Terminal 1
for x in $(seq 999); do kubectl delete pod -l app=cockroachdb -n cockroachdb; sleep 1s; done

# Terminal 2
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public:26257 --execute="SELECT * FROM roaches;" 
```
output: ❌
```
ERROR: cannot dial server.
Is the server running?
If the server is running, check --host client-side and --advertise server-side.

dial tcp 172.30.94.81:26257: i/o timeout
Failed running "sql"
command terminated with exit code 1
```

**Single Replica- Stop Simulating Failure and read from table**, 
We can at least verify that the data was properly written to disk and that it is retreivable after we stop killing the CockroachDB instance. We read from the `cockroach-public` service when the pod comes back up. 
```
# Terminal 1
Control-C, stop killing the cockroachdb pod.

# Terminal 2
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public:26257 --execute="SELECT * FROM roaches;" 
```
output: ✅
```
          name          |    country
------------------------+----------------
  American Cockroach    | United States
  Brownbanded Cockroach | United States
(2 rows)


Time: 21ms
```

**Summary**
In this scenerio where cockraoch is deployed as a single replica where there is no replication. If the instance of the database is down there is simply nothing we can do about it when there is just one replica. This was just for testing purposes. Now in the next test we will let CockroachDB do what it does best and replicate.


**Cleanup**
```
kubectl delete -f k8s/single-replica-crdb.yaml
```

## 3 Replica Test
**Note** - This is a recommended approach for running cockroachDB since we have improved redudancy and failover by adding back the `StatefulSet`. We should see Cockroach's automated replication of our data in this test.


### Setup 3 Replica CockroachDB  
_We are performing the StatefulSet's init function in an `initContainer` on a job so all you need to do is apply the k8s folder and wait._

The `initConatiner` sleeps for **35 seconds** while waiting for the pods to become ready then the job's pod template has a command that does the cockroach init function.
```
kubectl apply -f k8s/3-replica-crdb.yaml
```
output:
```
serviceaccount/cockroachdb created
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-crdb-sa created
service/cockroachdb-public created
service/cockroachdb created
Warning: policy/v1beta1 PodDisruptionBudget is deprecated in v1.21+, unavailable in v1.25+; use policy/v1 PodDisruptionBudget
poddisruptionbudget.policy/cockroachdb-budget created
job.batch/cluster-init created
statefulset.apps/cockroachdb created
```

Wait about 40 seconds for the pods become ready after the init function occurs.   
```
kubectl get pods -n cockroachdb
```
output:
```
NAME                    READY   STATUS      RESTARTS   AGE
cluster-init--1-x6kjv   0/1     Completed   0          43s
cockroachdb-0           1/1     Running     0          43s
cockroachdb-1           1/1     Running     0          43s
cockroachdb-2           1/1     Running     0          43s
crdb-tester             1/1     Running     0          21m
```


**3 Replica- Write Test**, create a table.
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public:26257 --execute="CREATE TABLE roaches (name STRING, country STRING); INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States')"
```
output: ✅
```
INSERT 2
```
**3 Replica- Read Test**, read from the table.
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public:26257 --execute="SELECT * FROM roaches;" 
```
output: ✅
```
          name          |    country
------------------------+----------------
  American Cockroach    | United States
  Brownbanded Cockroach | United States
(2 rows)


Time: 36ms
```

**3 Replica- Simulate Failure in master pod, still READ data from database**, 
We complete this test by contually killing the CockroachDB instance and try reading form the `cockroach-public` service while the pod is down. 
```
# Terminal 1
for x in $(seq 999); do kubectl delete pod cockroachdb-0 -n cockroachdb; sleep 1s; done

# Terminal 2
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public:26257 --execute="SELECT * FROM roaches;" 
```
output: ✅
```
          name          |    country
------------------------+----------------
  American Cockroach    | United States
  Brownbanded Cockroach | United States
(2 rows)


Time: 3ms
```

**3 Replica- Simulate Failure in master pod, still WRITE data to database**, 
We complete this test by contually killing the CockroachDB instance and try reading form the `cockroach-public` service while the pod is down. 
```
# Terminal 1
for x in $(seq 999); do kubectl delete pod cockroachdb-0 -n cockroachdb; sleep 1s; done

# Terminal 2
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public:26257 --execute="INSERT INTO roaches VALUES ('Flying Venomous Cockroach', 'Colombia')"
```
output: ✅
```
INSERT 1


Time: 23ms
```

**3 Replica- Stop Simulating Failure and read from table**, 
We want to ensure the "Flying Venomous Cockroach" was added to the database despite the master pod being down.
```
# Terminal 1
Control-C, stop killing the cockroachdb-0 pod.

# Terminal 2
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public:26257 --execute="SELECT * FROM roaches;" 
```
output: ✅
```
            name            |    country
----------------------------+----------------
  American Cockroach        | United States
  Brownbanded Cockroach     | United States
  Flying Venomous Cockroach | Colombia
(3 rows)


Time: 3ms
```

**Summary**
In this scenerio CockroachDB's automated replication is displayed as we continously deleted the master pod. The database continued to function as normal and we were able to read and write data. This is the reason why CockroachDB was built, to ensure your data is safe even during failure.


**Cleanup**
```
kubectl delete -f k8s/3-replica-crdb.yaml
```



[Cockroach Facts](https://www.pestworldforkids.org/pest-info/bug-articles-by-type/how-long-can-a-cockroach-live-without-its-head/#:~:text=So%2C%20how%20long%20can%20a,mouth%20or%20head%20to%20breathe.)  

[CockroachDB Availability Model](https://www.cockroachlabs.com/docs/v21.2/multi-active-availability.html) 



