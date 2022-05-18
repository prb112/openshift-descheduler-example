# Descheduler Profile - LifecycleAndUtilization - PodLifeTime Strategy

Before proceeding, the Descheduler Operator must be installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

> [PodLifeTime](https://github.com/kubernetes-sigs/descheduler#podlifetime): evicts pods that are too old. By default, pods that are older than 24 hours are removed. You can customize the pod lifetime value.

## Steps

1. Update the LifecycleAndUtilization Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/3_LifecycleAndUtilization_PodLifeTime.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and `podLifeTime.maxPodLifeTimeSeconds: 60`.

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

5. Create a DeploymentConfig

```
$ oc -n test apply -f files/3_LifecycleAndUtilization_PodLifeTime_dp.yml
deployment.apps/lifetime configured
```

6. Double check that it is started:

```
$ oc get pods -n=test       
NAME                    READY   STATUS      RESTARTS   AGE
lifetime-5977d9c459-flb9p   1/1     Running     0          83s
```

7. Follow the Logs until we see the processing

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler -f                                
I0510 15:17:35.984625       1 pod_lifetime.go:110] "Evicted pod because it exceeded its lifetime" pod="time/lifetime-5977d9c459-flb9p" maxPodLifeTime=300
I0510 15:17:35.984650       1 descheduler.go:287] "Number of evicted pods" totalEvicted=3
```

8. Delete the deployment

```
oc -n test delete deployment.apps/lifetime
deployment.apps "lifetime" deleted
```

# Summary

The PodLifeTime is probably the most simple.