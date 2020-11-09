# OCS 4.4 on Bare Metal

## Table of Contents

- [OCS 4.4 on Bare Metal](#ocs-44-on-bare-metal)
  - [Table of Contents](#table-of-contents)
  - [Authors](#authors)
  - [Description](#description)
  - [Prerequisites](#prerequisites)
  - [Install Local Storage Operator](#install-local-storage-operator)
  - [Configure Local Storage Operator](#configure-local-storage-operator)
  - [Install OpenShift Container Storage Operator](#install-openshift-container-storage-operator)
  - [Configure OpenShift Container Storage Operator](#configure-openshift-container-storage-operator)
  - [Appendix](#appendix)
    - [Limit Resources](#limit-resources)

## Authors

- Jared Hocutt ([@jaredhocutt](https://github.com/jaredhocutt))
- John Call ([@johnsimcall](https://github.com/johnsimcall))
- Kevin Jones ([@kjw3](https://github.com/kjw3))
- Marc Chisinevski ([@marcredhat](https://github.com/marcredhat))

## Description

This guide describes the steps to install and configure OpenShift Container
Storage (OCS) 4.4 on an OpenShift cluster that has been installed using the
bare metal UPI installation method.

Follow the OCS/OpenShift Container Platform interoperability matrix at https://access.redhat.com/articles/4731161.

Additional information can be found in the [official docs][1].

## Prerequisites

1. You must be logged into the OpenShift cluster.

2. You must have at least three OpenShift worker nodes in the cluster with
   locally attached storage devices on each of them.  The additional storage
   devices should not have any partitions, LVM, etc...

   - Each worker node must have a minimum of 16 CPUs and 64 GB memory.

   - Each of the three worker nodes must have at least one raw block device
     available to be used by OpenShift Container Storage.

   - Each worker node must be labeled to deploy OpenShift Container Storage
     pods. To label the nodes, use the following command:

     ```bash
     oc label nodes <NodeName> cluster.ocs.openshift.io/openshift-storage=''
     ```

If not already enabled, open the following iptables ports on all OpenShift cluster nodes as follows:

```bash
$ sudo iptables -A INPUT -p tcp -m multiport --destination-ports 9001:9022,17001:17020 -j ACCEPT
$ sudo iptables -A INPUT -p udp -m multiport --destination-ports 9002 -j ACCEPT
```


On all the worker nodes, prep the disks targeted for installation. Run the wipefs command if these disks already had filesystems on them. 

For this exercise, each worker has four drives, one being the root disk, and the other three targeted for OpenShift Container Storage. 

On each worker, run something similar to the command shown below.

*** Ensure that you are not running this against disks with data on them. 

```bash
$ for i in b c d; do sudo wipefs -a /dev/sd$i; done
```


```bash
$ for i in b c d; do sudo mkfs.ext4 /dev/sd$i; done
```


## Install Local Storage Operator

1. Create a namespace called `local-storage` as follows.

   ```bash
   oc new-project local-storage
   ```

2. Create `OperatorGroup` and `Subscription` CR `local-storage-operator.yaml`.

   ```yaml
   ---
   apiVersion: operators.coreos.com/v1
   kind: OperatorGroup
   metadata:
     name: local-storage
     namespace: local-storage
   spec:
     targetNamespaces:
     - local-storage

   ---
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: local-storage-operator
     namespace: local-storage
   spec:
     channel: "4.4"
     name: local-storage-operator
     source: redhat-operators
     sourceNamespace: openshift-marketplace
     installPlanApproval: "Automatic"
   ```

3. Install the Local Storage Operator.

   ```bash
   oc apply -f local-storage-operator.yaml
   ```

4. Verify that the Local Storage Operator shows the Status as `Succeeded`.

   ```bash
   oc get csv
   ```


## Configure Local Storage Operator

1. Find the available storage devices.

   - List and verify the name of the worker nodes with the OpenShift Container
     Storage label.

      ```bash
      oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
      ```

     

   - For each worker node that is used for OpenShift Container Storage
     resources, find the unique `/dev/disk/by-id/` or `/dev/disk/by-path/`
     device name for each available raw block device.

     ```bash
     oc debug node/<name> -- chroot /host lsblk -o NAME,SIZE,MOUNTPOINT
     ```

     Example output:

     ```text
     Starting pod/storage0examplecom-debug ...
     To use host binaries, run `chroot /host`
     NAME                           SIZE MOUNTPOINT MODEL
     sda                          139.8G            INTEL SSDSC2BB15
     |-sda1                         384M /boot
     |-sda2                         127M /boot/efi
     |-sda3                           1M
     `-sda4                       139.2G
       `-coreos-luks-root-nocrypt 139.2G /sysroot
     sdb                          139.8G      	     INTEL SSDSC2BB15
     sdc                          931.5G        	ST1000NX0303
     sdd                          931.5G        	SanDisk SSD PLUS

     Removing debug pod ...
     ```

     In this example, the local devices available are `sdb`, `sdc`, and `sdd`.

   - Find the unique `by-id` or `by-path` device name depending on the hardware
     serial number for each device.

     ```bash
     oc debug node/<node_name> -- chroot /host find /dev/disk/by-path/ -printf '%p -> %l\n' | sort
     ```

     Example output:

     ```text
     Starting pod/storage0examplecom-debug ...
     To use host binaries, run `chroot /host`

     Removing debug pod ...
     /dev/disk/by-path/ ->
     /dev/disk/by-path/pci-0000:00:1f.2-ata-1-part1 -> ../../sda1
     /dev/disk/by-path/pci-0000:00:1f.2-ata-1-part2 -> ../../sda2
     /dev/disk/by-path/pci-0000:00:1f.2-ata-1-part3 -> ../../sda3
     /dev/disk/by-path/pci-0000:00:1f.2-ata-1-part4 -> ../../sda4
     /dev/disk/by-path/pci-0000:00:1f.2-ata-1 -> ../../sda
     /dev/disk/by-path/pci-0000:00:1f.2-ata-2 -> ../../sdb
     /dev/disk/by-path/pci-0000:00:1f.2-ata-3 -> ../../sdc
     /dev/disk/by-path/pci-0000:00:1f.2-ata-4 -> ../../sdd
     ```

     In this example, the by-path device names are:

     - /dev/disk/by-path/pci-0000:00:1f.2-ata-2
     - /dev/disk/by-path/pci-0000:00:1f.2-ata-3
     - /dev/disk/by-path/pci-0000:00:1f.2-ata-4

   - Repeat finding the device name `by-path` for all the other nodes that have
     the storage devices to be used by OpenShift Container Storage.

2. Create LocalVolume CR for block PVs.

   Example of `LocalVolume` CR `local-storage-block.yaml` using OCS label as
   node selector.

   ```yaml
   apiVersion: local.storage.openshift.io/v1
   kind: LocalVolume
   metadata:
     name: local-block
     namespace: local-storage
   spec:
     nodeSelector:
       nodeSelectorTerms:
       - matchExpressions:
           - key: cluster.ocs.openshift.io/openshift-storage
             operator: Exists
     storageClassDevices:
       - storageClassName: localblock
         volumeMode: Block
         devicePaths:
           - /dev/disk/by-path/pci-0000:00:1f.2-ata-2   # <-- modify this
           - /dev/disk/by-path/pci-0000:00:1f.2-ata-3   # <-- modify this
           - /dev/disk/by-path/pci-0000:00:1f.2-ata-4   # <-- modify this
   ```

   Update the `devicePaths` section to list all of the devices you would like
   to use. The Local Storage Operator will look for **all** of the devices in
   the list on **all** of the nodes. If you use non-unique values (e.g.
   `/dev/sda`, `/dev/disk/by-path/...ata-2`, etc...) this can result in the
   Local Storage Operator trying to create a volume from a device that may
   already be in use. Identifying each device's unique ID (e.g.
   `/dev/disk/by-id/...`) is a bit more laborious, but provides assurance that
   the Local Storage Operator and OpenShift Container Storage will only use the
   specified devices.

3. Create the LocalVolume CR for block PVs.

   ```bash
   oc create -f local-storage-block.yaml
   ```

4. Check if the pods are created.

   ```bash
   oc get pods -n local-storage
   ```

5. Check if the PVs are created.

   ```bash
   oc get pv
   ```

   Example output:

   ```text
   NAME            	CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS  	CLAIM   STORAGECLASS   REASON   AGE
   Local-pv-16980ea    931Gi  	RWO        	Delete       	Available       	localblock          	21h
   local-pv-316176c7   931Gi  	RWO        	Delete       	Available       	localblock          	21h
   local-pv-34e949f9   931Gi  	RWO        	Delete       	Available       	localblock          	21h
   local-pv-3f73d4ab   139Gi  	RWO        	Delete       	Available       	localblock          	21h
   local-pv-4ff9b09c   139Gi  	RWO        	Delete       	Available       	localblock          	21h
   local-pv-5f491d1d   139Gi  	RWO        	Delete       	Available       	localblock          	21h
   local-pv-76065ad9   931Gi  	RWO        	Delete       	Available       	localblock          	21h
   local-pv-bad3e5ca   931Gi  	RWO        	Delete       	Available       	localblock          	21h
   local-pv-d7f83ca8   931Gi  	RWO        	Delete       	Available       	localblock          	21h
   ```

6. Check for the new `localblock` `StorageClass`.

   ```bash
   oc get sc | grep localblock
   ```

   Example output:

   ```text
   NAME            PROVISIONER                     AGE
   localblock     kubernetes.io/no-provisioner    10m20s
   ```

## Install OpenShift Container Storage Operator

1. Create a namespace called openshift-storage and enable metrics:

   ```bash
   oc adm new-project openshift-storage

   oc label namespace openshift-storage openshift.io/cluster-monitoring="true"
   ```

2. Create `OperatorGroup` and `Subscription` CR `ocs-storage-operator.yaml`
   ([see docs][2]).

   ```yaml
   ---
   apiVersion: operators.coreos.com/v1
   kind: OperatorGroup
   metadata:
     name: openshift-storage
     namespace: openshift-storage
   spec:
     targetNamespaces:
     - openshift-storage

   ---
   apiVersion: operators.coreos.com/v1alpha1
   kind: Subscription
   metadata:
     name: ocs-operator
     namespace: openshift-storage
   spec:
     channel: "stable-4.4"
     name: ocs-operator
     source: redhat-operators
     sourceNamespace: openshift-marketplace
     installPlanApproval: "Automatic"
   ```

3. Install the Local Storage Operator.

   ```bash
   oc apply -f ocs-storage-operator.yaml
   ```

4. Verify that the Local Storage Operator shows the Status as `Succeeded`.

   ```bash
   oc get csv
   ```

   Example output:

   ```text
   NAME                        	DISPLAY                   	VERSION   REPLACES   PHASE
   lib-bucket-provisioner.v1.0.0   lib-bucket-provisioner    	1.0.0            	Succeeded
   ocs-operator.v4.4.2         	OpenShift Container Storage   4.4.2            	Succeeded
   ```

## Configure OpenShift Container Storage Operator

1. Create `StorageCluster` CR `cluster-service-metal.yaml` using
   `monDataDirHostPath` and `localblock` storage classes.

   ```yaml
   ---
   apiVersion: ocs.openshift.io/v1
   kind: StorageCluster
   metadata:
     name: ocs-storagecluster
     namespace: openshift-storage
   spec:
     manageNodes: false
     monDataDirHostPath: /var/lib/rook
     storageDeviceSets:
     - count: 1              # <-- modify this (disks-per-node)
       dataPVCTemplate:
         spec:
           accessModes:
           - ReadWriteOnce
           resources:
             requests:
               storage: 2Ti   # <-- modify this (1Gi, or size of PV)
           storageClassName: localblock
           volumeMode: Block
       name: ocs-deviceset
       placement: {}
       portable: false
       replica: 3
       resources: {}
   ```

   The `count` of `storageDeviceSets` must be equal to the number of storage
   devices available on each host (e.g. `1`).

   The storage size for `storageDeviceSets` must be less than or equal to the
   size of the raw block devices.

2. Create the StorageCluster CR.

   ```bash
   oc create -f cluster-service-metal.yaml
   ```

3. Refer to [Verifying your OpenShift Container Storage installation][3] to
   verify a successful installation.

   - See that the storageCluster CR is `Ready`.

     ```bash
     oc get storagecluster
     ```

     Example output:

     ```text
     NAME             	AGE 	PHASE   CREATED AT         	VERSION
     ocs-storagecluster   4m41s   Ready   2020-05-01T18:41:08Z   4.4.0
     ```

4. Consider defining OpenShift Container Storage CephFS as the default storage
   class.

   ```bash
   oc annotate storageclass/ocs-storagecluster-cephfs storageclass.kubernetes.io/is-default-class="true"
   ```

5. Verify that OpenShift Container Storage CephFS is the default storage class.

   ```bash
   oc get sc
   ```

   Example output:

   ```text
   NAME                              	PROVISIONER                         	RECLAIMPOLICY   VOLUMEBINDINGMODE  	ALLOWVOLUMEEXPANSION   AGE
   localblock                        	kubernetes.io/no-provisioner        	Delete      	WaitForFirstConsumer   false              	21h
   ocs-storagecluster-ceph-rbd       	openshift-storage.rbd.csi.ceph.com  	Delete      	Immediate          	false              	8m26s
   ocs-storagecluster-cephfs (default)   openshift-storage.cephfs.csi.ceph.com   Delete      	Immediate          	false              	8m26s
   openshift-storage.noobaa.io       	openshift-storage.noobaa.io/obc     	Delete      	Immediate          	false              	4m2s
   ```

6. Test

Set up a deployment file called pvctestpod.yml

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pv-claim
  labels:
    app: test-pv
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: test-pv-pod
  labels:
    app: test-pv
spec:
  volumes:
    - name: test-data
      persistentVolumeClaim:
        claimName: test-pv-claim
  containers:
    - name: test-pv-container
      image: nginx
      volumeMounts:
        - mountPath: "/test"
          name: test-data
```

```bash
# oc apply -f pvctestpod.yml

# oc get po/test-pv-pod -w
NAME          READY   STATUS              RESTARTS   AGE
test-pv-pod   0/1     ContainerCreating   0          9s
test-pv-pod   0/1     ContainerCreating   0          10s
test-pv-pod   0/1     ContainerCreating   0          15s
test-pv-pod   1/1     Running             0          18s


# oc exec -it `oc get po -l "app=test-pv" -o custom-columns=":metadata.name"` bash
```

```bash
root@test-pv-pod:/# df -h /test
Filesystem                      Size  Used Avail Use% Mounted on
/dev/pxd/pxd365962277412932531  976M  1.3M  908M   1% /test
root@test-pv-pod:/# touch /test/foo
root@test-pv-pod:/# ls -l /test/foo
-rw-r--r--. 1 root root 0 Jul 30 19:28 /test/foo
root@test-pv-pod:/# rm /test/foo
root@test-pv-pod:/# 
```


## Appendix

### Scale out by adding devices and/or nodes: 
https://www.openshift.com/blog/deploying-openshift-container-storage-using-local-devices

### Limit Resources

If you are going to attempt to install OCS on undersized machines that do not
meet the minimum resource requirements, you can define the following to limit
the CPU and memory for each component.

:warning: **IMPORTANT** :warning:
> Reducing the resource requests/limits may have unintended consequences
> including having critical pods evicted when the nodes become resource
> constrained. Please proceed with caution.  More discussion [here][4].

```yaml
resources:
  mds:
    limits:
      cpu: 0.5
    requests:
      cpu: 0.25
  noobaa-core:
    limits:
      cpu: 0.5
      memory: 4Gi
    requests:
      cpu: 0.25
      memory: 2Gi
  noobaa-db:
    limits:
      cpu: 0.5
      memory: 4Gi
    requests:
      cpu: 0.25
      memory: 2Gi
  mon:
    limits:
      cpu: 0.5
      memory: 2Gi
    requests:
      cpu: 0.25
      memory: 1Gi
  rook-ceph-rgw:
    limits:
      cpu: 250m
      memory: 2Gi
    requests:
      cpu: 100m
      memory: 1Gi
```

[1]: https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.4/html-single/deploying_openshift_container_storage/index#installing-openshift-container-storage-using-local-storage-devices_rhocs
[2]: https://docs.openshift.com/container-platform/4.4/operators/olm-adding-operators-to-cluster.html#olm-installing-operator-from-operatorhub-using-cli_olm-adding-operators-to-a-cluster
[3]: https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.4/html-single/deploying_openshift_container_storage/index#verifying-your-openshift-container-storage-installation_rhocs
[4]: https://github.com/openshift/ocs-operator/pull/521#pullrequestreview-413936502
