# OpenShift v4 OC commands 
Here is primary source of the OC cli documentation.

[RedHat OC CLI](https://docs.openshift.com/container-platform/4.12/cli_reference/index.html)

## Check all API resources on the OpenShift Cluster
Every resource is object and all object created through APIs

```oc api-resources```

```oc api-versions```

# Get on any resource
You can use get on any resource you got from oc api-resources command.

```oc get pods```
```oc get node -o wide```
```oc get node -o wide```
```oc get pods -o wide```

### Get pods with node details
```oc get po -A -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName```
```oc get pods -l "<Label>" --all-namespaces -o yaml | yq '.items[] | (.metadata.namespace + "," + .spec.hostname + "," + .spec.nodeName)' | sort | uniq -c```

### Find the pod and node on current namespace
```oc get po  -o wide | grep -v NODE | awk '{print $1 " - " $7}' ```

### Get and check if all services are ready
```oc get --raw=/readyz?verbose```

### Get Pods and grep on something also count
```oc get pods -A | grep "CrashLoopBackOff" | wc -l```

### Get pod disruption budget 
```oc get pdb -A```

### Get Persistent volume details
```oc get pv```
```oc get pvc```

# Describe on any resource
```oc describe <resource name>```

# Edit the resource manifest
```oc edit <resource name>```

# Remote login to pod
```oc rsh <pod name> ```

# Cluster Version
```oc get clusterversion -o jsonpath='{.items[].spec.clusterID}{"\n"}'```

# 
