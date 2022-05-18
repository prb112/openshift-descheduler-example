# Descheduler Profile - SoftTopologyAndDuplicates - RemovePodsViolatingTopologySpreadConstraint Strategy

Before proceeding, the Descheduler Operator must be installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

> [RemovePodsViolatingTopologySpreadConstraint](https://github.com/kubernetes-sigs/descheduler/tree/0b2c10d6cee917bc553f743c44c51e730f9b1205#removepodsviolatingtopologyspreadconstraint): This strategy makes sure that pods violating topology spread constraints are evicted from nodes. Specifically, it tries to evict the minimum number of pods required to balance topology domains to within each constraint's maxSkew. This strategy requires k8s version 1.18 at a minimum.

## Steps

1. Update the SoftTopologyAndDuplicates Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/4_SoftTopologyAndDuplicates_RemovePodsViolatingTopologySpreadConstraint.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and `strategies.RemovePodsViolatingTopologySpreadConstraint` and `includeSoftConstraints: true` is configured.

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
$ oc -n test apply -f files/4_SoftTopologyAndDuplicates_RemovePodsViolatingTopologySpreadConstraint_dp.yml
deployment.apps/unbalanced created
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
I0513 13:57:31.392584       1 topologyspreadconstraint.go:94] "Processing namespaces for topology spread constraints"
I0513 13:57:31.606334       1 topologyspreadconstraint.go:170] "Skipping topology constraint because it is already balanced" constraint={MaxSkew:1 TopologyKey:topology.kubernetes.io/zone WhenUnsatisfiable:ScheduleAnyway LabelSelector:nil}
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

This Profile shows the SoftTopologyAndDuplicates and how it causes descheduling and scheduling on the node with a Soft Topology Constraint.