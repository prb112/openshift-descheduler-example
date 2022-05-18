# Descheduler Profile - AffinityAndTaints - RemovePodsViolatingNodeTaints Strategy

Before proceeding, the Descheduler Operator must be installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

> [RemovePodsViolatingNodeAffinity](https://github.com/kubernetes-sigs/descheduler/tree/master#removepodsviolatingnodetaints): This strategy makes sure that pods violating NoSchedule taints on nodes are removed. For example there is a pod "podA" with a toleration to tolerate a taint key=value:NoSchedule scheduled and running on the tainted node. If the node's taint is subsequently updated/removed, taint is no longer satisfied by its pods' tolerations and will be evicted.

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

This ConfigMap should show the excluded namespaces and `RemovePodsViolatingNodeTaints.enabled = true`. Note, there are no excluded `taints`.

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

5. Create a ReplicaSet

```
$ oc -n test apply -f files/1_AffinityAndTaints_RemovePodsViolatingNodeTaints.yml
deployment.apps/taints-app created
```

6. Check the Pod distribution to see the rebalancing.

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName'
Name       NodeName
taints-app-482lg   worker-0.xip.io
taints-app-hqvm2   worker-1.xip.io
taints-app-pnlx8   worker-2.xip.io
```

7. Add a `NoSchedule` taint to one of the nodes.

```
$ oc adm taint nodes worker-0.xip.io key1=value1:NoSchedule
node/worker-0.xip.io tainted
```

8. Grab the log for the descheduler

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler  --tail=2000 > out.log
```

9. Verify you see `deschedulerNodeTaint`

```
I0518 15:09:23.193988       1 node_taint.go:106] "Not all taints with NoSchedule effect are tolerated after update for pod on node" pod="test/taints-app-99d58f75f-b6fz2" node="worker-0.xip.io"
I0518 15:09:23.260618       1 evictions.go:160] "Evicted pod" pod="test/taints-app-99d58f75f-b6fz2" reason="NodeTaint"
I0518 15:09:23.260807       1 event.go:294] "Event occurred" object="test/taints-app-99d58f75f-b6fz2" kind="Pod" apiVersion="v1" type="Normal" reason="Descheduled" message="pod evicted by sigs.k8s.io/deschedulerNodeTaint"
I0518 15:09:23.262133       1 evictions.go:345] "Pod lacks an eviction annotation and fails the following checks" pod="test/taints-app-99d58f75f-b6fz2" checks="pod is terminating"
```

10. Check the Pod distribution to see the rebalancing.

```
$ oc -n test get pods -o=custom-columns='Name:metadata.name,NodeName:spec.nodeName'
Name       NodeName
taints-app-c4qjq   worker-1.xip.io
taints-app-hqvm2   worker-1.xip.io
taints-app-pnlx8   worker-2.xip.io
```

11. Remove the `NoSchedule` taint from the selected node.

```
$ oc adm taint nodes worker-0.xip.io key1=value1:NoSchedule-
node/worker-0.xip.io untainted
```

12. Clean up Deployment

```
$ oc -n test delete deployment taints-app
deployment.apps "taints-app" deleted
```

# Summary
This shows how to work with taints and the descheduler.