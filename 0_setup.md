1. Login as Kube Administrator, such as `kubeadmin`

2. Click Admnistration > Namespaces

3. Enter the following in the Namespace Dialog: 
    - Namespace: openshift-kube-descheduler-operator
    - Labels: 
        - openshift.io/cluster-monitoring=true

The labels enable cluster-monitoring metrics.

4. Click Create

**OLM** If the environment has the Operator enabled for ppc64le

5. Click on Operators > Operator Hub

6. Search for `Kube Descheduler Operator`

7. Install the Descheduler

**Non-OLM** If the environment has the Operator not enabled for ppc64le

8. In the upper right, click on the drop down toggle > Copy Login Command

9. Click Display Token 

10. Copy the login command

11. Open a terminal, paste and run the login command. 

```
oc login
```

12. Create namespace

```
oc get namespace openshift-kube-descheduler-operator || oc create namespace openshift-kube-descheduler-operator
```

12. Create the Subscription

```
oc apply -f files/0_subscription.yml -n openshift-kube-descheduler-operator
subscription.operators.coreos.com/cluster-kube-descheduler-operator configured
```

13. Click Operators > Installed Operators > Kube Descheduler Operator 

14. Click on Kube Descheduler

You have setup the Operator and found where you can configure Kube Descheduler in the OpenShift Console.