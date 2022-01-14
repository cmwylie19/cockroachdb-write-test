# CockroachDB Write Test
**Question** _Does cockroachDB support writing to slaves in the `StatefulSet`?_

## Environment:
**Kubernetes**: OpenShift 4.9.9 Cluster on AWS  
**AWS Instance Type**: m5.xlarge   
**Cockroach Database**: v21.2.4

## Setup Environment
_We are performing the StatefulSet's init function in an `initContainer` on a job so all you need to do is apply the k8s folder and wait._

The `initConatiner` sleeps for **35 seconds** while waiting for the pods to become ready then the job's pod template has a command that does the cockroach init function.
```
kubectl create ns cockroachdb 
kubectl apply -f k8s
```

## Testing
Wait about 40 seconds for the pods to come up and the init function to occur.   

First, make sure your pods have come up and are ready.
```
kubectl get pods -n cockroachdb
```

Write to the cockroach master pod. Create a table called roaches from the CockroachDB docs example. Insert two values into the table.
```
kubectl exec -it pod/cockroachdb-0 -n cockroachdb -- cockroach sql --insecure --execute="CREATE TABLE roaches (name STRING, country STRING); INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States')"
```
output
```
INSERT 2
```
in a second terminal, create a script to always kill the master after a second. This is to ensure that we do not write to the master.
```
for x in $(seq 999); do kubectl delete pod cockroachdb-0 -n cockroachdb; sleep 1s; done
```

in the first terminal, lets write a new entry to the Cockroach database by writing to a slave pod (`cockroachdb-1`) to see if writing to slaves is supported by cockroachdb
```
kubectl exec -it pod/cockroachdb-1 -n cockroachdb -- cockroach sql --insecure --execute="INSERT INTO roaches VALUES ('Flying Venomous Cockroach', 'Colombia')"
```
output
```
INSERT 1
```

_seems like writing worked but we will guarantee it later._   
   
Create a new cockroach pod in the default namespace for testing
```
kubectl run crdb-tester --image=cockroachdb/cockroach:v21.2.4 --command -- sleep 9999s
```

Exec into the new `crdb-tester` pod and query the CockroachDB service to make sure the data was written
```
kubectl exec -it crdb-tester -- cockroach sql --insecure --host=cockroachdb-public.cockroachdb.svc.cluster.local:26257 --execute="SELECT * FROM roaches;"
```

output
```
            name            |    country
----------------------------+----------------
  American Cockroach        | United States
  Brownbanded Cockroach     | United States
  Flying Venomous Cockroach | Colombia
(3 rows)


Time: 71ms
```

You can see "Flying Venomous Cockroach" from Colombia was inserted into the roaches database. That means that the write worked on the StatefulSet slave.

## Extra Credit
but wait, we queried through a headless service, that could have been pod `cockroachdb-1` that answered the request. The same pod we used to insert. Lets make sure that the data was properly replicated in the `StatefulSet` by querying all in the database from the `cockroachdb-2` pod.   
```
k exec -it cockroachdb-2 -n cockroachdb -- cockroach sql --insecure --execute="SELECT * FROM roaches;"
```


## Cleanup
Delete all of the resources, then the namespace.

Delete tester pod
```
kubectl delete pod crdb-tester --force --grace-period=0
```

delete cockroach manifests
```
kubectl delete -f k8s --force --grace-period=0
```

delete cockroach namespace
```
kubectl delete pods,sa,secrets,cm --all --force --grace-period=0 -n cockroachdb;

kubectl delete ns cockroachdb
```