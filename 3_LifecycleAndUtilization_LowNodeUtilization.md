# Descheduler Profile - LifecycleAndUtilization - LowNodeUtilization Strategy

Before proceeding, the Descheduler Operator must be installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

> [LowNodeUtilization](https://github.com/kubernetes-sigs/descheduler/tree/master#lownodeutilization): All types of pods with the annotation descheduler.alpha.kubernetes.io/evict are eligible for eviction. This annotation is used to override checks which prevent eviction and users can select which pod is evicted. Users should know how and if the pod will be recreated.

## Steps 

1. Update the LifecycleAndUtilization Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/3_LifecycleAndUtilization_LowNodeUtilization.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and the nodeResourceUtilizationThresholds.

```
nodeResourceUtilizationThresholds:
    targetThresholds:
      cpu: 50
      memory: 50
      pods: 50
    thresholds:
      cpu: 20
      memory: 20
      pods: 20
```

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
$ oc -n test apply -f files/3_LifecycleAndUtilization_LowNodeUtilization_dp.yml
```

7. Check the pods are all on the other node 

```
$ oc -n test get pods -A -o=custom-columns='DATA:metadata.name,DATA:spec.nodeName'
unbalanced-6d757874c4-5bsml                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-bjxfr                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-bsmqk                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-ccns4                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-kvx7q                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-ldvfn                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-m5tgb                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-nlfkm                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-nrxwg                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-rdkmx                                                      worker-0.rdr-rhop.sslip.io
```

8. Uncordon the original worker.

```
$ oc adm uncordon worker-1.rdr-rhop.sslip.io              
node/worker-1.rdr-rhop.sslip.io uncordoned
```

9. Check the Logs until we see the processing

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler  --tail=200                                 
I0511 20:53:59.692336       1 descheduler.go:287] "Number of evicted pods" totalEvicted=5
```

10. Check the pods are now redistributed. 

```
$ oc -n test get pods -A -o=custom-columns='DATA:metadata.name,DATA:spec.nodeName'   
unbalanced-6d757874c4-5bsml                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-bjxfr                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-bsmqk                                                      worker-1.rdr-rhop.sslip.io
unbalanced-6d757874c4-ccns4                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-kvx7q                                                      worker-1.rdr-rhop.sslip.io
unbalanced-6d757874c4-ldvfn                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-m5tgb                                                      worker-0.rdr-rhop.sslip.io
unbalanced-6d757874c4-nlfkm                                                      worker-1.rdr-rhop.sslip.io
unbalanced-6d757874c4-nrxwg                                                      worker-1.rdr-rhop.sslip.io
unbalanced-6d757874c4-rdkmx                                                      worker-1.rdr-rhop.sslip.io
```


11. Delete the deployment

```
oc -n test delete deployment.apps/unbalanced
deployment.apps "unbalanced" deleted
```

## Summary

This Profile shows the LowNodeUtilization and how it causes descheduling and scheduling on the uncordoned *or* underutilized node.