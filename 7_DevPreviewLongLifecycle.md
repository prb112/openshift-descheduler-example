# Descheduler Profile - DevPreviewLongLifecycle

Before proceeding, the Descheduler Operator must be installed.

There are two tests included: 

1. Running with a Deployment > Pod with PVC Storage and the DevPreviewLongLifecycle
2. Running with a Deployment > Pod with PVC Storage and no DevPreviewLongLifecycle

## Steps

1. Update the DevPreviewLongLifecycle Policy

```
$ oc apply -n openshift-kube-descheduler-operator -f files/7_DevPreviewLongLifecycle.yml
kubedescheduler.operator.openshift.io/cluster created
```

2. Check the configmap to see the Descheduler Policy. 

```
$ oc -n openshift-kube-descheduler-operator get cm cluster -o=yaml
```

This ConfigMap should show the excluded namespaces and the strategies LowNodeUtilization and RemovePodsHavingTooManyRestarts is included and PodLifeTime is NOT.

This log should show a started Descheduler.

# Summary 

You have seen how to use DevPreviewLongLifecycle policy is a subset of the LifecycleAndUtilization policy.