# Descheduler Profile - AffinityAndTaints - RemovePodsViolatingInterPodAntiAffinity Strategy

Before proceeding, the Descheduler Operator must be installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests.
**WARNING**

> [RemovePodsViolatingInterPodAntiAffinity](https://github.com/kubernetes-sigs/descheduler/tree/master#removepodsviolatinginterpodantiaffinity): This strategy makes sure that pods violating interpod anti-affinity are removed from nodes. For example, if there is podA on a node and podB and podC (running on the same node) have anti-affinity rules which prohibit them to run on the same node, then podA will be evicted from the node so that podB and podC could run. This issue could happen, when the anti-affinity rules for podB and podC are created when they are already running on node.

To accomplish this approach, we're going to layer a ReplicaSet with an anti-affinity for a label that doesn't yet exist. Then the Pods via a ReplicaSet that violate the already scheduled antiaffinity. 

## Steps

0. Reassign the labels for `topology.kubernetes.io/zone` on each node in the cluster.

```
$ oc label node 'worker-0.xip.io' topology.kubernetes.io/zone-
$ oc label node 'worker-0.xip.io' topology.kubernetes.io/zone=a
$ oc label node 'worker-1.xip.io' topology.kubernetes.io/zone-
$ oc label node 'worker-1.xip.io' topology.kubernetes.io/zone=a
$ oc label node 'worker-2.xip.io' topology.kubernetes.io/zone-
$ oc label node 'worker-2.xip.io' topology.kubernetes.io/zone=a
node/worker-0.xip.io unlabeled
node/worker-0.xip.io labeled
node/worker-1.xip.io unlabeled
node/worker-1.xip.io labeled
node/worker-2.xip.io unlabeled
node/worker-2.xip.io labeled
```

1. Update the AffinityAndTaints Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/1_AffinityAndTaints.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and `RemovePodsViolatingInterPodAntiAffinity.enabled` is `true`.

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

5. Create the ReplicaSet frontend

```
$ oc -n test apply -f files/1_AffinityAndTaints_RemovePodsViolatingInterPodAntiAffinity_frontend.yml
replicaset.apps/frontend created
```

6. Check the distribution of the Pods to nodes (ideally 1-to-1)

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName'
Name             NodeName
frontend-8xrx5   worker-0.xip.io
frontend-nb458   worker-1.xip.io
frontend-wsms7   worker-2.xip.io
```

7. If the mapping is less than 1-1, scale up the number of replicas until you have at least one pod on each worker node.

```
$ oc -n test scale replicaset/frontend --replicas=6
replicaset.apps/frontend scaled
```

8. Create the replicaset backend

```
$ oc -n test apply -f files/1_AffinityAndTaints_RemovePodsViolatingInterPodAntiAffinity_backend.yml
replicaset.apps/backend created
```

9. Check that the Pods are distributed across both nodes.

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName'
Name             NodeName
backend-nh8kt    worker-0.xip.io
backend-tb55w    worker-1.xip.io
frontend-52mbm   worker-0.xip.io
frontend-m6wdm   worker-1.xip.io
frontend-nbm9c   worker-2.xip.io
```

10. Add a label that now violates the constraints `app=backend` for one of the backend pods.

```
$ oc -n test label pod/backend-nh8kt app=backend
pod/backend-nh8kt labeled
```

11. Grab the descheduler log 

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler  --tail=2000 > out.log 
```

12. Scan the log for indications of deschduling for Pod Anti Affinity.

```
I0518 14:17:23.175087       1 evictions.go:160] "Evicted pod" pod="test/frontend-52mbm" reason="InterPodAntiAffinity"
I0518 14:17:23.175512       1 node.go:169] "Pod fits on node" pod="test/frontend-52mbm" node="worker-0.xip.io"
I0518 14:17:23.176180       1 event.go:294] "Event occurred" object="test/frontend-52mbm" kind="Pod" apiVersion="v1" type="Normal" reason="Descheduled" message="pod evicted by sigs.k8s.io/deschedulerInterPodAntiAffinity"
I0518 14:17:23.251139       1 evictions.go:345] "Pod lacks an eviction annotation and fails the following checks" pod="test/frontend-52mbm" checks="pod is terminating"
I0518 14:17:23.252571       1 descheduler.go:287] "Number of evicted pods" totalEvicted=1
```

13. Confirm you do not see any frontend pods distributed on to the worker which has the backend pod that you labeled. 

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName'
Name             NodeName
backend-nh8kt    worker-0.xip.io
backend-tb55w    worker-1.xip.io
frontend-52mbm   worker-0.xip.io
frontend-m6wdm   worker-1.xip.io
frontend-nbm9c   worker-2.xip.io
```

14. Delete the ReplicaSet

```
$ oc -n test delete replicaset/backend replicaset/frontend
replicaset.apps "backend" deleted
replicaset.apps "frontend" deleted
```

# Summary 
This has shown how to test the InterPodAntiAffinity violations after scheduling and how the descheduler enforces it.

# Reference

- [OpenShift - Advanced Scheduling and Pod Affinity/Anti-affinity](https://docs.openshift.com/container-platform/3.11/admin_guide/scheduling/pod_affinity.html)