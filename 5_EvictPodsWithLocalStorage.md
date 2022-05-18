# Descheduler Profile - EvictPodsWithLocalStorage

Before proceeding, the Descheduler Operator must be installed.

You must deploy an NFS server in the environment, in the UPI deployment the bastion server has an `nfs-server.service` installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

Per the Policy, the Descheduler Policy changes `evictLocalStoragePods` to true when the Policy is added.

> evictLocalStoragePods allows eviction of pods with local storage

There are two tests included: 

1. Running with a Deployment > Pod with Local Storage and EvictPodsWithLocalStorage
2. Running with a Deployment > Pod with Local Storage and no EvictPodsWithLocalStorage

## Steps

1. Update the EvictPodsWithLocalStorage Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/5_EvictPodsWithLocalStorage.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and `evictLocalStoragePods: true`.

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

5. Create a Deployment with an emptyDir volume

```
$ oc apply -n test -f files/5_EvictPodsWithLocalStorage_dp.yml
deployment.apps/local-pvc created
```

6. Check that the pods are started.

```
$ oc -n test get pods       
NAME                         READY   STATUS    RESTARTS   AGE
local-pvc-64555d976d-54565   1/1     Running   0          26s
```

7. Once you see a new set of pods created, the Eviction has happened, and it should show up in the logs. Wait on the logs to be updated.

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler --since=10h --tail=2000 | grep local-pvc
```

9. Scan for the *output* for the following lines:

```
I0512 18:58:33.650269       1 evictions.go:160] "Evicted pod" pod="test/local-pvc-64555d976d-54565" reason="PodLifeTime"
I0512 18:58:33.690168       1 pod_lifetime.go:110] "Evicted pod because it exceeded its lifetime" pod="test/local-pvc-64555d976d-54565" maxPodLifeTime=300
```

10. Update the EvictPodsWithLocalStorage Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/5_EvictPodsWithLocalStorage_no.yml
kubedescheduler.operator.openshift.io/cluster created
```

11. Check the Pod Age is greater than 5 minutes. (you might need to check multiple times)

```
$ oc -n test get pods
NAME                         READY   STATUS    RESTARTS   AGE
local-pvc-64577d976d-54565   1/1     Running   0          5m43s
```

Note, you won't find logs indicating the Pod were removed.

12. Delete the Deployment local-pvc

```
$ oc -n test delete deployment local-pvc
deployment.apps "local-pvc" deleted
```

# Summary 

You have seen how to use EvictPodsWithLocalStorage.