# Descheduler Profile - AffinityAndTaints - RequiredDuringSchedulingIgnoredDuringExecution Strategy

Before proceeding, the Descheduler Operator must be installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

> [RemovePodsViolatingNodeAffinity](https://github.com/kubernetes-sigs/descheduler/tree/master#removepodsviolatingnodeaffinity): This strategy makes sure all pods violating node affinity are eventually removed from nodes. Node affinity rules allow a pod to specify requiredDuringSchedulingIgnoredDuringExecution type, which tells the scheduler to respect node affinity when scheduling the pod but kubelet to ignore in case node changes over time and no longer respects the affinity. 

Importantly, when enabled, the strategy serves as a temporary implementation of requiredDuringSchedulingRequiredDuringExecution and evicts pod for kubelet that no longer respects node affinity.

## Steps

1. Update the AffinityAndTaints Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/1_AffinityAndTaints.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and `RemovePodsViolatingNodeAffinity.params.nodeAffinityType` is `requiredDuringSchedulingIgnoredDuringExecution` is set.

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

5. Label the zones so it's unbalanced (a,b)

a. `worker-0`

```
$ oc label node 'worker-0.xip.io' topology.kubernetes.io/zone=a
node/worker-0.xip.io labeled
```

b. `worker-1`

```
$ oc label node 'worker-1.xip.io' topology.kubernetes.io/zone=a
node/worker-1.xip.io not labeled
```

SKIP c. `worker-2`

```
$ oc label node 'worker-2.xip.io' topology.kubernetes.io/zone=b
node/worker-1.xip.io not labeled
```

5. Create a ReplicaSet

```
$ oc -n test apply -f files/1_AffinityAndTaints_RequiredDuringSchedulingIgnoredDuringExecution_dp.yml
replicaset.apps/ua created
```

6. Check the Pod distribution to see the rebalancing.

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName'
Name       NodeName
ua-482lg   worker-0.xip.io
ua-hqvm2   worker-1.xip.io
ua-pnlx8   worker-1.xip.io
```

7. Remove the label for `worker-1`

```
$ oc label node 'worker-1.xip.io' topology.kubernetes.io/zone-
node/worker-1.xip.io unlabeled
```

8. Check the Pod distribution to see the rebalancing.

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName'
Name       NodeName
ua-6466x   worker-0.xip.io
ua-hjjbt   worker-1.xip.io
ua-mh4td   worker-1.xip.io
```

9. Check the Logs until we see the processing

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler  --tail=200                                 
I0517 17:02:22.407548       1 node.go:165] "Pod does not fit on node" pod="test/ua-j2zhv" node="worker-1.xip.io"
I0517 17:02:22.407585       1 node.go:147] "Pod can possibly be scheduled on a different node" pod="test/ua-j2zhv" node="worker-0.xip.io"
I0517 17:02:22.407608       1 evictions.go:345] "Pod lacks an eviction annotation and fails the following checks" pod="powervm-rmc/powervm-rmc-qsp8j" checks="[pod is a DaemonSet pod, pod has local storage and descheduler is not configured with evictLocalStoragePods]"
I0517 17:02:22.407621       1 node_affinity.go:108] "Evicting pod" pod="test/ua-v645d"
I0517 17:02:22.426179       1 round_trippers.go:553] POST https://172.30.0.1:443/api/v1/namespaces/test/pods/ua-v645d/eviction 201 Created in 18 milliseconds
I0517 17:02:22.426258       1 evictions.go:160] "Evicted pod" pod="test/ua-v645d" reason="NodeAffinity"
I0517 17:02:22.426318       1 node_affinity.go:108] "Evicting pod" pod="test/ua-j2zhv"
I0517 17:02:22.426738       1 event.go:294] "Event occurred" object="test/ua-v645d" kind="Pod" apiVersion="v1" type="Normal" reason="Descheduled" message="pod evicted by sigs.k8s.io/deschedulerNodeAffinity"
I0517 17:02:22.432454       1 round_trippers.go:553] POST https://172.30.0.1:443/api/v1/namespaces/test/events 201 Created in 5 milliseconds
I0517 17:02:22.450136       1 round_trippers.go:553] POST https://172.30.0.1:443/api/v1/namespaces/test/pods/ua-j2zhv/eviction 201 Created in 23 milliseconds
I0517 17:02:22.450220       1 evictions.go:160] "Evicted pod" pod="test/ua-j2zhv" reason="NodeAffinity"
...
I0517 17:02:22.451634       1 event.go:294] "Event occurred" object="test/ua-j2zhv" kind="Pod" apiVersion="v1" type="Normal" reason="Descheduled" message="pod evicted by sigs.k8s.io/deschedulerNodeAffinity"
```

11. Clean up replicaset

```
$ oc -n test delete replicaset/ua                                          
replicaset.apps "ua" deleted
```

# Summary
This is a nodeAffinity alignment.