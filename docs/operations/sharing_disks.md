# Sharing storage between multiple Virtual Machines

Shareable disks allows multiple VMs to share the same underlying storage. In order to use this feature, a special care is required because this could lead to data corruption and the loss of important data. Shareable disk demand either data synchronization at application level or the usage of clustered filesystems. These advanced configurations are out-of-scope of this documentation and use-cases specific.

If the underlying storage is using the storage logical unit number (LUN), it can be directly connected to a VM from the storage area network (SAN).
The SCSI passthrough allows to directly send requests to the device using the SCSI ioctls. This feature can be used for creatig cluster shared volumes orchestrated by SAN-aware applications running inside the VMs.

If the disk is a partition, a direct attached block device and generally doesn't support SCSI protocol, an alternative method for sharing the disk between multiple VMs is using the `shareable` option. The `shareable` option indicates to libvirt/QEMU that the disk is going to be accessed by multiple VMs and not to create a lock for the writes.

The choice between the two methods depends on the use-cases and available storage. For example, if you plan to use the [Cluster Shared Volumes](https://docs.microsoft.com/en-us/windows-server/failover-clustering/failover-cluster-csvs) (CSV) in Windows, this requires having read-write access to the same LUN.
If your plan is to use a cluster-filesystem like [GFS2](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/configuring_gfs2_file_systems/assembly_gfs2-usage-considerations-configuring-gfs2-file-systems) on top of block devices, then the shareable option is the right choice.

## Example using LUN disks

In this example, the local SCSI disk will be exposed and pass to the VM as a pre-provisioned volume using the PV/PVC Kubernetes interface.

The first step is to statically provision the Persistent Volume:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: scsi-pv
spec:
  volumeMode: Block
  storageClassName: local-scsi
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  local:
    path: /dev/sda
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node02
```
```bash
kubectl get pv
scsi-pv             1Gi        RWO            Retain           Available           local-scsi              57s
```

The PVC will be bound to a matching PV by Kubernetes and the disk can be used by reference the claimName in the VMI definition
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: scsi-pvc
spec:
  volumeMode: Block
  storageClassName: local-scsi
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
```bash
$ kubectl get pvc
NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
scsi-pvc   Bound    scsi-pv   1Gi        RWO            local-scsi     44m
```

Now, the we can define 2 VMs that have access to the same disk defined as `lun` disk:
```
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-scsi-1
  name: vm-scsi-1
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-scsi-1
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
          - lun:
              bus: scsi
            name: scsi-disk
        machine:
          type: ""
        resources:
          requests:
            memory: 2G
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/fedora-with-test-tooling-container-disk:devel
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
        name: cloudinitdisk
      - name: scsi-disk
        persistentVolumeClaim:
          claimName: scsi-pvc
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-scsi-2
  name: vm-scsi-2
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-scsi-2
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: kubevirt.io/vm
                operator: In
                values:
                - vm-scsi-1
            topologyKey: "kubernetes.io/hostname"
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
          - lun:
              bus: scsi
            name: scsi-disk
        machine:
          type: ""
        resources:
          requests:
            memory: 2G
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/fedora-with-test-tooling-container-disk:devel
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
        name: cloudinitdisk
      - name: scsi-disk
        persistentVolumeClaim:
          claimName: scsi-pvc
```

If the sg_dd utility is installed inside the 2 guests, you can have fun and test the sharing writing directly some strings:
```bash
$ virtctl console vm-scsi-1
$  lsscsi
[6:0:0:0]    disk    QEMU     QEMU HARDDISK    2.5+  /dev/sda 
$ printf "Test awesome shareable disks" | sudo sg_dd of=/dev/sda oflag=sgio count=150 conv=notrunc
# Log into the second VM
$ virtctl console vm-scsi-2
$ sudo sg_dd if=/dev/sda bs=1 count=150 conv=notrunc
Test awesome shareable disks150+0 records in
150+0 records out
```

## Example using raw block devices with shareable

In this example, we use Rook Ceph in order to dynamically provisioning the PVC.
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: block-pvc
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-ceph-block
```
```bash
$ kubectl get pvc
NAME        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
block-pvc   Bound    pvc-0a161bb2-57c7-4d97-be96-0a20ff0222e2   1Gi        RWO            rook-ceph-block   51s
```
Then, we can declare 2 VMs and setting the `shareable` option to true for the shared disk.
```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-block-1
  name: vm-block-1
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-block-1
    spec:
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
          - disk:
              bus: virtio
            shareable: true
            name: block-disk
        machine:
          type: ""
        resources:
          requests:
            memory: 2G
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/fedora-with-test-tooling-container-disk:devel
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
        name: cloudinitdisk
      - name: block-disk
        persistentVolumeClaim:
          claimName: block-pvc
---
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  labels:
    kubevirt.io/vm: vm-block-2
  name: vm-block-2
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/vm: vm-block-2
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: kubevirt.io/vm
                operator: In
                values:
                - vm-block-1
            topologyKey: "kubernetes.io/hostname"
      domain:
        devices:
          disks:
          - disk:
              bus: virtio
            name: containerdisk
          - disk:
              bus: virtio
            name: cloudinitdisk
          - disk:
              bus: virtio
            shareable: true
            name: block-disk
        machine:
          type: ""
        resources:
          requests:
            memory: 2G
      terminationGracePeriodSeconds: 0
      volumes:
      - containerDisk:
          image: registry:5000/kubevirt/fedora-with-test-tooling-container-disk:devel
        name: containerdisk
      - cloudInitNoCloud:
          userData: |-
            #cloud-config
            password: fedora
            chpasswd: { expire: False }
        name: cloudinitdisk
      - name: block-disk
        persistentVolumeClaim:
          claimName: block-pvc
                                        
```
Similarly to the previous example, we can try to write and read a string in the 2 guest to test the sharing.
```bash
$ virtctl console vm-block-1
$ printf "Test awesome shareable disks" | sudo dd  of=/dev/vdc bs=1 count=150 conv=notrunc
28+0 records in
28+0 records out
28 bytes copied, 0.0264182 s, 1.1 kB/s
# Log into the second guest
$ virtctl console vm-block-2
$ sudo dd  if=/dev/vdc bs=1 count=150 conv=notrunc
Test awesome shareable disks150+0 records in
150+0 records out
150 bytes copied, 0.136753 s, 1.1 kB/s
```

## Scheduling considerations

If you are using a local devices or a RWO PVC, putting the affinity on the VMs that share the storage guarantees they will be scheduled on the same node. In the examples, we put the affinity on the second VM using the label used by the first one. If you are using shared storage with RWX PVCs, then the affinity rule is not necessary as the storage can be attached simoultaneously on multiple nodes.

