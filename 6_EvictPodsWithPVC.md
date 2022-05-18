# Descheduler Profile - EvictPodsWithPVC

Before proceeding, the Descheduler Operator must be installed.

You must deploy an NFS server in the environment, in the UPI deployment the bastion server has an `nfs-server.service` installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

Per the Policy, the Descheduler Policy changes `ignorePvcPods` to true when the Policy is added.

> ignorePvcPods set whether PVC pods should be evicted or ignored

There are two tests included: 

1. Running with a Deployment > Pod with PVC Storage and the EvictPodsWithPVC
2. Running with a Deployment > Pod with PVC Storage and no EvictPodsWithPVC

## Steps

1. Update the EvictPodsWithPVC Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/6_EvictPodsWithPVC.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and `ignorePvcPods: true`.

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

5. Create a PersistentVolume, PersistentVolumeClaim and Deployment

```
$ oc -n test apply -f files/6_EvictPodsWithPVC_dp.yml    
persistentvolume/evict-pv created
persistentvolumeclaim/evict-pvc created
deployment.apps/lifetime-store created
```

6. Check that the evict-pv PV is available

```
$ oc -n test get persistentvolume/evict-pv
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM            STORAGECLASS   REASON   AGE
evict-pv   1Gi        RWO            Retain           Available   default/claim1                           19s
```

7. Check the pvc is Bound

```
$ oc -n test get persistentvolumeclaim/evict-pvc                                                                                           130 ↵
NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
evict-pvc   Bound    evict-pv   1Gi        RWX                           11m
```

8. Once you see a new set of pods created, the Eviction has happened, and it should show up in the logs. Wait on the logs to be updated.

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler --since=10h --tail=2000 | grep lifetime-store 
```

9. Scan for the *output* for the following lines:

```
I0512 17:53:29.016475       1 evictions.go:160] "Evicted pod" pod="test/lifetime-store-d474d8fd8-n6snx" reason="PodLifeTime"
I0512 17:53:29.016625       1 pod_lifetime.go:110] "Evicted pod because it exceeded its lifetime" pod="test/lifetime-store-d474d8fd8-n6snx" maxPodLifeTime=300
```

10. Update the EvictPodsWithPVC Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/6_EvictPodsWithPVC_no.yml
kubedescheduler.operator.openshift.io/cluster created
```

11. Check the Pod Age is greater than 5 minutes. (you might need to check multiple times)

```
$ oc -n test get pods
NAME                             READY   STATUS    RESTARTS   AGE
lifetime-store-d474d8fd8-hltzx   1/1     Running   0          5m43s
```

Note, you won't find logs indicating the Pod were removed.

12. Delete the Deployment lifetime-store

```
$ oc -n test delete deployment lifetime-store
deployment.apps "lifetime-store" deleted
```

13. Delete the pvc lifetime-store

```
$ oc -n test delete persistentvolumeclaim/evict-pvc
persistentvolumeclaim "evict-pvc" deleted
```

14. Delete the pv lifetime-store

```
$ oc delete persistentvolume/evict-pv
persistentvolume/evict-pv deleted
```

# Summary 

You have seen how to use EvictPodsWithPVC.