# openshift-descheduler-example

The purpose of this repository is to provide a set of simple proof-of-concept configuration scripts for OpenShift and the Descheduler on OpenShift on Power.

1. AffinityAndTaints
    - [1_AffinityAndTaints_RemovePodsViolatingInterPodAntiAffinity.md](1_AffinityAndTaints_RemovePodsViolatingInterPodAntiAffinity.md)
    - [1_AffinityAndTaints_RemovePodsViolatingNodeTaints.md](1_AffinityAndTaints_RemovePodsViolatingNodeTaints.md)
    - [1_AffinityAndTaints_RequiredDuringSchedulingIgnoredDuringExecution.md](1_AffinityAndTaints_RequiredDuringSchedulingIgnoredDuringExecution.md)
2. TopologyAndDuplicates	
    - [2_TopologyAndDuplicates_RemoveDuplicates.md](2_TopologyAndDuplicates_RemoveDuplicates.md)
    - [2_TopologyAndDuplicates_RemovePodsViolatingTopologySpreadConstraint.md](2_TopologyAndDuplicates_RemovePodsViolatingTopologySpreadConstraint.md)
3. LifecycleAndUtilization	
    - [3_LifecycleAndUtilization_LowNodeUtilization.md](3_LifecycleAndUtilization_LowNodeUtilization.md)
    - [3_LifecycleAndUtilization_PodLifeTime-PriorityFiltering.md](3_LifecycleAndUtilization_PodLifeTime-PriorityFiltering.md)
    - [3_LifecycleAndUtilization_PodLifeTime.md](3_LifecycleAndUtilization_PodLifeTime.md)
    - [3_LifecycleAndUtilization_RemovePodsHavingTooManyRestarts.md](3_LifecycleAndUtilization_RemovePodsHavingTooManyRestarts.md)
4. SoftTopologyAndDuplicates	
    - [4_SoftTopologyAndDuplicates_RemoveDuplicates.md](4_SoftTopologyAndDuplicates_RemoveDuplicates.md)
    - [4_SoftTopologyAndDuplicates_RemovePodsViolatingTopologySpreadConstraint.md](4_SoftTopologyAndDuplicates_RemovePodsViolatingTopologySpreadConstraint.md)
5. EvictPodsWithLocalStorage	
    - [5_EvictPodsWithLocalStorage.md](5_EvictPodsWithLocalStorage.md)
6. EvictPodsWithPVC	
    - [6_EvictPodsWithPVC.md](6_EvictPodsWithPVC.md)
7. DevPreviewLongLifecycle	
    - [7_DevPreviewLongLifecycle.md](7_DevPreviewLongLifecycle.md)

# Is this a Red Hat or IBM supported solution?

No. This is only a proof of concept that serves as a good starting point to understand how the Descheduler Profiles works in OpenShift.