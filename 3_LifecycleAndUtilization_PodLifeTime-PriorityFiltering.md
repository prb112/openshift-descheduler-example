# Descheduler Profile - LifecycleAndUtilization - PodLifeTime Strategy - PriorityFiltering - thresholdPriority

Before proceeding, the Descheduler Operator must be installed.

**WARNING**
Do not run this in a LIVE cluster, this should be dedicated to the specific tests, as it will EVICT running pods every 1 minute when the Pods are older than `5m`.
**WARNING**

> [Pod Evictions](https://github.com/kubernetes-sigs/descheduler#pod-evictions): All types of pods with the annotation descheduler.alpha.kubernetes.io/evict are eligible for eviction. This annotation is used to override checks which prevent eviction and users can select which pod is evicted. Users should know how and if the pod will be recreated.
> [Priority filtering](https://github.com/kubernetes-sigs/descheduler#priority-filtering)

[thresholdPriority](https://github.com/kubernetes-sigs/descheduler/blob/0b2c10d6cee917bc553f743c44c51e730f9b1205/pkg/descheduler/evictions/evictions.go#L224) compares spec.template.spec.priority against a set Descheduler Policy value.

Note the docs say that you can't configure both thresholdPriority and thresholdPriorityClassName on the Pod itself, if the given priority class does not exist, descheduler won't create it and will throw an error. Note, the priority admissions controller doesn't let us specify the spec.priority, rather we have to set the `priorityClassName`.

There are three tests included in the yaml: 

1. low-priority which should be evicted
2. high-priority which should **not** be evicted
3. system-cluster-critical which should **not** be evicted

## Steps

1. Update the LifecycleAndUtilization Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/3_LifecycleAndUtilization_PodLifeTime-PriorityFiltering-ThresholdPriority.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and the nodeResourceUtilizationThresholds. Note, this impacts all strategies in the profile. 

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

5. Create a few deployments

```
$ oc --namespace=test apply -f files/3_LifecycleAndUtilization_PodLifeTime-PriorityFiltering-ThresholdPriority_pods.yml
priorityclass.scheduling.k8s.io/lowpriority created
priorityclass.scheduling.k8s.io/highpriority created
deployment.apps/system-cluster-critical-priority created
deployment.apps/high-priority created
deployment.apps/low-priority created
```

6. Double check that it is started:

```
$ oc get pods -n test
NAME                                                READY   STATUS    RESTARTS   AGE
high-priority-6d965c5c-4j66d                        1/1     Running   0          10m
low-priority-6c879b99bf-lp47w                       1/1     Running   0          3m33s
system-cluster-critical-priority-5b9b74b949-vr2tf   1/1     Running   0          10m
```

*Note* the time difference in the running pod.

7. Follow the Logs until we see the processing

### high-priority 

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler --tail=200 | grep -i high-p
I0511 19:36:58.888502       1 evictions.go:345] "Pod lacks an eviction annotation and fails the following checks" pod="test/high-priority-6d965c5c-4j66d" checks="pod has higher priority than specified priority class threshold"
```

### low-priority

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler --tail=2000 | grep -i low-priority
I0511 19:35:58.922583       1 event.go:294] "Event occurred" object="test/low-priority-6c879b99bf-drr2v" kind="Pod" apiVersion="v1" type="Normal" reason="Descheduled" message="pod evicted by sigs.k8s.io/deschedulerPodLifeTime"
```

### system critical priority

```
$ oc -n openshift-kube-descheduler-operator logs -l app=descheduler --tail=2000 | grep -i system-cluster-critical-priority                                                           130 ↵
I0511 19:35:59.091870       1 evictions.go:345] "Pod lacks an eviction annotation and fails the following checks" pod="test/system-cluster-critical-priority-5b9b74b949-vr2tf" checks="[pod has system critical priority, pod has higher priority than specified priority class threshold]"
```

8. Delete the deployment

```
oc -n test delete deployment.apps/system-cluster-critical-priority
deployment.apps/system-cluster-critical-priority deleted
```

```
oc -n test delete deployment.apps/low-priority
deployment.apps/low-priority deleted
```

```
oc -n test delete deployment.apps/high-priority
deployment.apps/high-priority deleted
```

# Summary

The PodLifeTime is probably the most simple and easiest to show filtering with.


## Notes
1. I hit this when I created standalone pods.

> I0511 18:57:58.986525       1 evictions.go:345] "Pod lacks an eviction annotation and fails the following checks" pod="test/low-priority" checks="pod does not have any ownerRefs"

I did not get the test case to work as the code checks the Descheduler policy.

// HaveEvictAnnotation checks if the pod have evict annotation
func HaveEvictAnnotation(pod *v1.Pod) bool {
	_, found := pod.ObjectMeta.Annotations[evictPodAnnotationKey]
	return found
}

// IsPodEvictableBasedOnPriority checks if the given pod is evictable based on priority resolved from pod Spec.
func IsPodEvictableBasedOnPriority(pod *v1.Pod, priority int32) bool {
	return pod.Spec.Priority == nil || *pod.Spec.Priority < priority
}

	ev := &evictable{}
	if pe.evictFailedBarePods {
		ev.constraints = append(ev.constraints, func(pod *v1.Pod) error {
			ownerRefList := podutil.OwnerRef(pod)
			// Enable evictFailedBarePods to evict bare pods in failed phase
			if len(ownerRefList) == 0 && pod.Status.Phase != v1.PodFailed {
				return fmt.Errorf("pod does not have any ownerRefs and is not in failed phase")
			}
			return nil
		})
	} else {
		ev.constraints = append(ev.constraints, func(pod *v1.Pod) error {
			ownerRefList := podutil.OwnerRef(pod)
			// Moved from IsEvictable function for backward compatibility
			if len(ownerRefList) == 0 {
				return fmt.Errorf("pod does not have any ownerRefs")
			}
			return nil
		})
	}

2. Reference https://access.redhat.com/solutions/6127191

3. You can switch class name and priority in the deployment however the admission controller may not let you create the deployment.