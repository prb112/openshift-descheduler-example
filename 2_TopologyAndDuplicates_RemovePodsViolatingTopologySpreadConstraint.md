# Strategy: TopologyAndDuplicates Profile > PodTopologySpreadConstraint Strategy

Before proceeding, the Descheduler Operator must be installed on a Cluster with 3 Worker Nodes.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 3 minutes.
**WARNING**

> [RemovePodsViolatingTopologySpreadConstraint](https://github.com/kubernetes-sigs/descheduler#removepodsviolatingtopologyspreadconstraint): This strategy makes sure that pods violating topology spread constraints are evicted from nodes. Specifically, it tries to evict the minimum number of pods required to balance topology domains to within each constraint's maxSkew.

Since this is run as part of the TopologyAndDuplicates profile there is an implication of `RemoveDuplicates` which makes it harder to exercise without three nodes and at most a 1 Pod in a Kubernetes (Deployment, ReplicaSet, DaemonSet et cetra) as when the labels change in the environment the `RemoveDuplicates` will be executed first.

## Steps

0. Check you have Three Nodes

```
$ oc get nodes -lnode-role.kubernetes.io/worker
NAME                                                 STATUS   ROLES    AGE   VERSION
worker-0.xip.io   Ready    worker   17h   v1.23.5+70fb84c
worker-1.xip.io   Ready    worker   17h   v1.23.5+70fb84c
worker-2.xip.io   Ready    worker   17h   v1.23.5+70fb84c
```

1. Update the Topology Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/2_TopologyAndDuplicates.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and `strategies.RemovePodsViolatingTopologySpreadConstraint` and `includeSoftConstraints: false` is configured.

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

Note, you'll see `Error from server (NotFound): namespaces "test" not found` if its the first time the namespace is being created.


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

5. Create the ReplicaSet

a. first replicaset

```
$ oc -n test apply -f files/2_TopologyAndDuplicates_rs_a.yml
replicaset.apps/ua created
```

b. second replicaset

```
$ oc -n test apply -f files/2_TopologyAndDuplicates_rs_b.yml                                          
replicaset.apps/ub created
```

c. third replicaset

```
$ oc -n test apply -f files/2_TopologyAndDuplicates_rs_c.yml                                          
replicaset.apps/ub created
```

6. Verify the pods are scheduled on `worker-0`.

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName'
Name               NodeName
Name       NodeName
ua-dwwwh   worker-0.xip.io
ua-gql9r   worker-0.xip.io
ub-7j6bw   worker-1.xip.io
ub-rhhx8   worker-0.xip.io
uc-4f52z   worker-2.xip.io
uc-kkffv   worker-1.xip.io
```

7. Add a label to worker-1

```
$ oc label node 'worker-1.xip.io' topology.kubernetes.io/zone-
node/worker-1.xip.io unlabeled
```


8. Reassing the label for worker-1

```
oc label node 'worker-1.xip.io' topology.kubernetes.io/zone=a 
node/worker-1.xip.io labeled
```

9. Check the Pod distribution to see the rebalancing.

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName'
Name       NodeName
ua-482lg   worker-0.xip.io
ua-hqvm2   worker-1.xip.io
ua-pnlx8   worker-2.xip.io
ub-k7g2m   worker-2.xip.io
ub-ndmqp   worker-1.xip.io
ub-vsnpt   worker-0.xip.io
uc-l5pkk   worker-0.xip.io
uc-rc9bp   worker-2.xip.io
uc-tkpn8   worker-1.xip.io
```

10. Verify the logs show an eviction based on the deschedulerPodTopologySpread

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler  --tail=20000
I0517 15:39:38.704420       1 event.go:294] "Event occurred" object="test/ua-hqvm2" kind="Pod" apiVersion="v1" type="Normal" reason="Descheduled" message="pod evicted by sigs.k8s.io/deschedulerPodTopologySpread"
I0517 15:39:38.709303       1 round_trippers.go:553] POST https://172.30.0.1:443/api/v1/namespaces/test/events 201 Created in 4 milliseconds
I0517 15:39:38.791390       1 round_trippers.go:553] POST https://172.30.0.1:443/api/v1/namespaces/test/pods/ub-vsnpt/eviction 201 Created in 87 milliseconds
I0517 15:39:38.791471       1 evictions.go:160] "Evicted pod" pod="test/ub-vsnpt" reason="PodTopologySpread"
I0517 15:39:38.791560       1 descheduler.go:287] "Number of evicted pods" totalEvicted=3
```

11. Clean up replicaset

```
$ oc -n test delete replicaset/ua replicaset/ub replicaset/uc                                                   
replicaset.apps "ua" deleted
replicaset.apps "ub" deleted
replicaset.apps "uc" deleted
```

# Summary
This is a simple realignment of pods using a topology spread constraint.


# Notes

1. Check the Node-Pod Distribution

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName' | awk '{print $NF}'  | sort | uniq -c
   1 NodeName
   4 worker-1.xip.io
   2 worker-2.xip.io
   1 worker-0.xip.io
```

2. Check the Node Labels for a worker

```
$ oc get nodes --show-labels -lnode-role.kubernetes.io/worker
NAME                                                 STATUS   ROLES    AGE   VERSION           LABELS
worker-0.xip.io   Ready    worker   20h   v1.23.5+70fb84c   beta.kubernetes.io/arch=ppc64le,beta.kubernetes.io/os=linux,kubernetes.io/arch=ppc64le,kubernetes.io/hostname=worker-0.xip.io,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos,topology.kubernetes.io/zone=b
worker-1.xip.io   Ready    worker   20h   v1.23.5+70fb84c   beta.kubernetes.io/arch=ppc64le,beta.kubernetes.io/os=linux,kubernetes.io/arch=ppc64le,kubernetes.io/hostname=worker-1.xip.io,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos,node=a,topology.kubernetes.io/zone=a
worker-2.xip.io   Ready    worker   20h   v1.23.5+70fb84c   beta.kubernetes.io/arch=ppc64le,beta.kubernetes.io/os=linux,kubernetes.io/arch=ppc64le,kubernetes.io/hostname=worker-2.xip.io,kubernetes.io/os=linux,node-role.kubernetes.io/worker=,node.openshift.io/os_id=rhcos,node=a,topology.kubernetes.io/zone=b
```

# References

- https://kubernetes.io/docs/concepts/workloads/pods/pod-topology-spread-constraints/
- https://kubernetes.io/blog/2020/05/introducing-podtopologyspread/
- https://kubernetes.io/docs/reference/labels-annotations-taints/ 