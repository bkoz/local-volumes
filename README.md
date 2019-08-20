# Notes from testing local volumes on OpenShift 3.11

#### In general, follow this documentation
https://docs.openshift.com/container-platform/3.11/install_config/configuring_local.html

Create file systems on storage devices.

Create mount points on app nodes and mount storage.

I had permisson issues if ```/mnt/local-storage``` was not used.

```
mkdir -p /mnt/local-storage/ssd/disk1
mkdir -p /mnt/local-storage/ssd/disk2
```

Set the SELinux context after mounting storage.

```
chcon -R unconfined_u:object_r:svirt_sandbox_file_t:s0 /mnt/local-storage/
```

#### OpenShift Configuration

This requires cluster-admin privileges.

```
oc new-project local-storage

oc create serviceaccount local-storage-admin

oc adm policy add-scc-to-user privileged -z local-storage-admin

oc create -f configmap.yaml
```

#### Create the provisioner template.

```
oc create -f https://raw.githubusercontent.com/openshift/origin/release-3.11/examples/storage-examples/local-examples/local-storage-provisioner-template.yaml
```

#### Create the provisioner daemonset.
```
oc new-app -p CONFIGMAP=local-volume-config -p SERVICE_ACCOUNT=local-storage-admin -p NAMESPACE=local-storage -p PROVISIONER_IMAGE=registry.redhat.io/openshift3/local-storage-provisioner:v3.11 local-storage-provisioner
```
There should be a ```local-volume-provisioner``` running on each node.

Example output.

```
oc get pods -n local-storage

NAME                             READY     STATUS    RESTARTS   AGE
local-volume-provisioner-drh5j   1/1       Running   0          4h
local-volume-provisioner-hzjr5   1/1       Running   0          4h
local-volume-provisioner-zx2gh   1/1       Running   0          4h
```

#### Create the storage class.
```
oc create -f storage-class-ssd.yaml
```
#### Check the PVs

Example output.

```
oc get pv | grep local

$ oc get pv | grep local
local-pv-4bb2f206                          9951Mi     RWO            Delete           Available                                                                   local-ssd                           4h
local-pv-5abd0294                          9951Mi     RWO            Delete           Bound       cake/www-web-2                                                  local-ssd                           4h
local-pv-9dced8fb                          9951Mi     RWO            Delete           Available                                                                   local-ssd                           4h
local-pv-aa8dbe40                          9951Mi     RWO            Delete           Bound       cake/www-web-0                                                  local-ssd                           4h
local-pv-e31d6b9                           9951Mi     RWO            Delete           Available                                                                   local-ssd                           4h
local-pv-e57f984f                          9951Mi     RWO            Delete           Bound       cake/www-web-1                                                  local-ssd                           4h
```

#### Create the test stateful set.

This can be performed as a regular user but the default service account for this project requires the ```anyuid``` SCC.

```
oc new-project storage-test

oc adm policy add-scc-to-user anyuid -z default -n storage-test 

oc create -f nginx-statefullset.yaml
```

Example output.
```
oc get pods 

NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          4h
web-1     1/1       Running   0          4h
web-2     1/1       Running   0          4h
```
```
oc get pvc

NAME        STATUS    VOLUME              CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound     local-pv-aa8dbe40   9951Mi     RWO            local-ssd      4h
www-web-1   Bound     local-pv-e57f984f   9951Mi     RWO            local-ssd      4h
www-web-2   Bound     local-pv-5abd0294   9951Mi     RWO            local-ssd      4h
```

#### How to clean up the stateful set.

```
oc delete statefulset web
oc delete svc nginx
oc delete pvc --all
```