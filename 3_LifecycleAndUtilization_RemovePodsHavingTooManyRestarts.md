# Descheduler Profile - LifecycleAndUtilization - RemovePodsHavingTooManyRestarts Strategy

Before proceeding, the Descheduler Operator must be installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

Per the documentation:
> [RemovePodsHavingTooManyRestarts](https://github.com/kubernetes-sigs/descheduler#removepodshavingtoomanyrestarts): removes pods whose containers have been restarted too many times. Pods where the sum of restarts over all containers (including Init Containers) is more than 100.

We have to wait for *100* restarts. It is not configurable through the operator.

Backoff times are hardcoded in Kubernetes as described here kubernetes/kubernetes#57291 and cannot be overridden. 

There are essentially two tests included: 

1. spec.containers restart 100 times and are evicted
2. spec.initContainers restart 100 times and are evicted. 

This test takes a LONG time approximately 10 hrs for each case.  Feel free to RUN them in parallel with slight modifications to the deployment names.

## Steps

1. Update the **LifecycleAndUtilization** Policy with `spec.container` failures.

```
$ oc apply -n openshift-kube-descheduler-operator -f files/3_LifecycleAndUtilization_RemovePodsHavingTooManyRestarts.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap shows the excluded namespaces and `podsHavingTooManyRestarts.podRestartThreshold: 100` as a setting.

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

5. Create a Deployment that fails.

```
$ oc -n test apply -f files/3_LifecycleAndUtilization_RemovePodsHavingTooManyRestarts_dp.yml
deployment.apps/demo created
```

6. Double check that it is scheduled to nodes:

```
$ oc -n test get pods 
NAME                READY   STATUS      RESTARTS      AGE
demo-8z7lh          0/1     Error       2 (23s ago)   44s
demo-t9tps          0/1     Error       2 (23s ago)   44s
```

7. You need to monitor the Pods over the next few hours to see the status and the `RESTARTS`.

```
$ oc -n test get pods 
NAME                    READY   STATUS             RESTARTS        AGE
demo-64d9c84b75-665nv   0/1     CrashLoopBackOff   9 (4m52s ago)   26m
demo-64d9c84b75-7hxdv   0/1     CrashLoopBackOff   9 (5m2s ago)    26m
```

The **CrashLoopBackOff** is not tunable, so it'll take some time to hit `100`.

8. Once you see a new set of pods `demo-64d9c84b75-665nv` changed to for instance `demo-64d9c84b75-dgp5b`, the Eviction has happened, and it should show up in the logs. Wait on the logs to be updated.

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler --since=10h --tail=20000 > out.log
```

9. Scan for the output.log following lines:

```
I0511 10:23:30.306748       1 evictions.go:160] "Evicted pod" pod="test/demo-64d9c84b75-665nv" reason="TooManyRestarts"
I0511 10:25:30.384381       1 evictions.go:160] "Evicted pod" pod="test/demo-64d9c84b75-7hxdv" reason="TooManyRestarts"
```

You've now seen `spec.container` TooManyRestarts policy take action.

10. Delete the Deployment demo (we're going to re-use the name)

```
$ oc -n test delete deployment.apps/demo
deployment.apps "demo" deleted
```

11. Update the **LifecycleAndUtilization** Policy with spec.initContainers failures, by creating a Deployment that fails.

```
$ oc -n test apply -f files/3_LifecycleAndUtilization_RemovePodsHavingTooManyRestarts_init.yml
deployment.apps/demo created
```

12. Double check that it is started:

```
$ oc -n test get pods 
NAME                   READY   STATUS      RESTARTS      AGE
demo-64d9c84b75-dgp5b  0/1     Error       2 (23s ago)   44s
demo-64d9c84b75-mrwd2  0/1     Error       2 (23s ago)   44s
```

13. Wait for ~10 hrs, you need to follow the descheduler cluster pod.

```
$ oc -n test get pods 
NAME                    READY   STATUS                  RESTARTS         AGE
demo-64d9c84b75-dgp5b   0/1     Init:CrashLoopBackOff   27 (63s ago)     114m
demo-64d9c84b75-mrwd2   0/1     Init:CrashLoopBackOff   26 (4m53s ago)   112m
```

14. Once you see a new set of pods created, the Eviction has happened, and it should show up in the logs. Wait on the logs to be updated.

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler --since=10h --tail=20000 > out.log
```

15. Scan for the *output.log* following lines:

```
I0511 10:23:30.306748       1 evictions.go:160] "Evicted pod" pod="test/demo-64d9c84b75-zlw5r" reason="TooManyRestarts"
I0511 10:25:30.384381       1 evictions.go:160] "Evicted pod" pod="test/demo-64d9c84b75-v46kq" reason="TooManyRestarts"
```

16. Delete the Deployment demo (we're going to re-use the name)

```
oc -n test delete deployment.apps/demo
deployment.apps "demo" deleted
```

# Summary

The RemovePodsHavingTooManyRestarts is has two configuration for the strategy that forces TooMany restarts.

I did look into using a custom BackOff. It's hard coded in Kubernetes and OKD. If you want to learn more, you can look at:

- **Debug** https://containersolutions.github.io/runbooks/posts/kubernetes/crashloopbackoff/
- **kubernetes/kubernetes** https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L944
- **Issue** https://github.com/kubernetes/kubernetes/issues/57291