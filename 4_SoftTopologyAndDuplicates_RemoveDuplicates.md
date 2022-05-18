# Descheduler Profile - SoftTopologyAndDuplicates - RemoveDuplicates Strategy

Before proceeding, the Descheduler Operator must be installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

> [RemoveDuplicates](https://github.com/kubernetes-sigs/descheduler/tree/0b2c10d6cee917bc553f743c44c51e730f9b1205#removeduplicates): This strategy makes sure that there is only one pod associated with a ReplicaSet (RS), ReplicationController (RC), StatefulSet, or Job running on the same node. If there are more, those duplicate pods are evicted for better spreading of pods in a cluster. This issue could happen if some nodes went down due to whatever reasons, and pods on them were moved to other nodes leading to more than one pod associated with a RS or RC, for example, running on the same node. Once the failed nodes are ready again, this strategy could be enabled to evict those duplicate pods.

## Steps

1. Update the SoftTopologyAndDuplicates Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/4_SoftTopologyAndDuplicates_RemoveDuplicates.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and `strategies.RemoveDuplicates` is configured.

3. Check the descheduler cluster 

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler 
```

This log should show a started Descheduler.

4. Create a test namespace

```
$ oc get namespace test || oc create namespace test
namespace/test created
```

5. [Cordon](https://docs.openshift.com/container-platform/4.10/nodes/nodes/nodes-nodes-working.html) one of the workers so we unbalance the number of assigned pods. 

a. List the nodes

```
$ oc get nodes
```

b. Select a worker node, such as `worker-1.rdr-rhop.sslip.io` 

c. Cordon the node

```
$ oc adm cordon worker-1.rdr-rhop.sslip.io
node/worker-1.rdr-rhop.sslip.io cordoned
```

6. Create a deployment

```
$ oc -n test apply -f files/4_SoftTopologyAndDuplicates_RemoveDuplicates_dp.yml
replicaset.apps/unbalanced created
```

7. Check the pods are all on the other node 

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName' -lapp=unbalanced
Name                          NodeName
unbalanced-54pmt   worker-1.rdr-rhop.sslip.io
unbalanced-8mfx8   worker-1.rdr-rhop.sslip.io
```

8. Uncordon the worker.

```
$ oc adm uncordon worker-1.rdr-rhop.sslip.io              
node/worker-1.rdr-rhop.sslip.io uncordoned
```

9. Check the Logs until we see the processing

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler  --tail=200                                 
I0512 19:35:45.106891       1 duplicates.go:199] "Adjusting feasible nodes" owner={namespace:test kind:ReplicaSet name:unbalanced-6d757874c4 imagesHash:docker.io/ibmcom/pause-ppc64le:3.1} from=5 to=2
I0512 19:35:45.106915       1 duplicates.go:207] "Average occurrence per node" node="worker-1.rdr-rhop.sslip.io" ownerKey={namespace:test kind:ReplicaSet name:unbalanced-6d757874c4 imagesHash:docker.io/ibmcom/pause-ppc64le:3.1} avg=1
I0512 19:35:45.126413       1 evictions.go:160] "Evicted pod" pod="test/unbalanced-6d757874c4-8mfx8" reason="RemoveDuplicatePods"
I0512 19:35:45.126547       1 descheduler.go:287] "Number of evicted pods" totalEvicted=1
```

10. Check the pods are now redistributed. 

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName' -lapp=unbalanced
Name                          NodeName
unbalanced-54pmt   worker-1.rdr-rhop.sslip.io
unbalanced-8mfx8   worker-0.rdr-rhop.sslip.io
```

11. Delete the deployment

```
oc -n test delete replicaset.apps/unbalanced
replicaset.apps "unbalanced" deleted
```

## Summary

This Profile shows the SoftTopologyAndDuplicates and how it causes descheduling and scheduling on the uncordoned *or* maximum of 2 on a node.