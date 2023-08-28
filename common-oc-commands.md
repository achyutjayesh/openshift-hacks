# OpenShift v4 OC commands 
Here is primary source of the OC cli documentation.

[RedHat OC CLI](https://docs.openshift.com/container-platform/4.12/cli_reference/index.html)

## Explain any resource of the OpenShift like Man page
```oc explain pod```

```oc explain <Any OCP Resource Name>```

## Login to OCP Cluster
```oc login --username=<user name> --server=<url of the cluster>:6443 --insecure-skip-tls-verify=true```

```oc login --server https://<url>:6443 -u <user name>```

## Check all API resources on the OpenShift Cluster
Every resource is object and all object created through APIs

```oc api-resources```

```oc api-versions```

## Project and Status
```oc create ns <project name>```

```oc project```

```oc status```

# Create or Modify/Delete Resources
```oc create ns <name of the namespace/project>```

```oc create -f <Path of the single Yaml resource or directory> -n <Name Space>```

```oc replace -f <Path of the single Yaml resource or directory> -n <name space>```

```oc apply -f <Path of the single Yaml resource or directory> -n <name space>```

## Wait for the cluster to be deployed
```oc wait <OCP Resource> --for=condition=Ready --timeout=300s -n <name space>```

## Delete resource
```oc delete <name of the api resource> <resource>```

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

### Get Oject backup locally 
```oc get pvc <name of the resource> -o yaml > ~/xyz.yaml```

# Describe on any resource
```oc describe <resource name>```

# Edit the resource manifest
```oc edit <resource name>```

# Remote login to pod and execute commands
```oc rsh <pod name> ```

```oc rsh <Name of the pod> <curl localhost:8080 **any other commands**> | jq | grep <some text> -A 2```

# Cluster Version
```oc get clusterversion -o jsonpath='{.items[].spec.clusterID}{"\n"}'```

# Analyze Resources on the Cluster
## Display resource statistics & (CPU/memory) usage
```oc adm top nodes```

```oc adm top pods```

```oc adm top images```

```oc adm top imagestreams```

## Logs transfer from OCP POD to host toolbox
```oc rsync <nameof the pod>:<path of the file/folder> .```

# Memory check
```oc rsh <Name of the pod> df -h```

```oc adm top nodes```

# Work with Secret

## Extract Secret
```oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-```

## Get Secret
```oc get secret <Name of the secret> -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt```






